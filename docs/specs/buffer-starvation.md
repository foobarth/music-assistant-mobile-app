# Audio Buffer Starvation — Stream Stutter During Reconnect

> **Spec:** Post-reconnect audio continuity
> **Status:** Draft
> **Date:** 2026-07-25
> **Source:** `AudioStreamManager.kt`, `LocalPlayerController.kt`, `SendspinClient.kt`

---

## 1. Summary

When the network degrades during driving (tunnel, dead zone, cellular handover), the Sendspin audio buffer drains before the WebSocket reconnects. The consumer posts `isStarved = true`, and `LocalPlayerController` tears down the stream when `isStarved + Reconnecting + isPlaying` all hold simultaneously. After reconnect, the server picks up at its own timeline — the gap is audible as silence or stutter. The user discovered that seeking back 10s fixes it, proving the server has the data. The pipeline should do this automatically.

---

## 2. Root Cause

### 2.1 Tear-down on starve is too aggressive

**File:** `LocalPlayerController.kt` lines 534–545

```kotlin
combine(client.state, client.isStarved, localPlayerData) { state, starved, data ->
    starved && data?.player?.isPlaying == true &&
        (state is SendspinState.Reconnecting || state is SendspinState.Error)
}.distinctUntilChanged().collect { lostDuringPlayback ->
    if (lostDuringPlayback) {
        pauseLocalIfPlaying()
        client.stopStream()
        // => queue cleared, decoder released, AudioTrack paused+flushed
    }
}
```

When `isStarved` flips `true` while the transport is `Reconnecting`, the stream is **immediately terminated**. This clears the queue and releases the decoder. When the reconnect completes and `stream/start` arrives, the pipeline starts from scratch with an empty buffer. Net result: silence until the new buffer fills.

### 2.2 Reconnect retransmit dedup loses overlap

**File:** `AudioStreamManager.kt` lines 267–273

```kotlin
val inserted = queueLock.withLock {
    if (ts <= lastConsumedTs) return@withLock false  // ← drops retransmitted overlap
    ...
}
```

After reconnect, the server retransmits frames from slightly **before** the disconnect point (to ensure gapless handover). But `lastConsumedTs` is the last timestamp the consumer *successfully played*. Every retransmitted frame with `ts <= lastConsumedTs` is dropped. The consumer gets nothing until the server's timeline passes `lastConsumedTs`, creating an audible gap.

### 2.3 No re-request mechanism

The client has no way to ask the server "resend from position X." The only recovery path is full `stream/start`, which clears state and starts fresh — losing the gap position. The user's manual seek back 10s triggers a `players/cmd/seek` + new `stream/start`, which works because the server sends audio from the new position.

---

## 3. Proposed Fixes

### 3.1 Grace period before teardown (immediate, low risk)

Instead of tearing down the instant `isStarved` becomes true during reconnect, wait up to N seconds for the reconnect to complete. If data resumes within the window, the pipeline continues uninterrupted.

```kotlin
// In LocalPlayerController.kt, the combine block
var starveTimer: Job? = null

combine(client.state, client.isStarved, localPlayerData) { ... }
    .collect { lostDuringPlayback ->
        starveTimer?.cancel()
        if (lostDuringPlayback) {
            starveTimer = launch {
                delay(8_000)  // 8s grace for reconnect to complete
                if (client.isStarved.value &&
                    client.state.value is SendspinState.Reconnecting) {
                    pauseLocalIfPlaying()
                    client.stopStream()
                    errorBus.emit(...)
                }
            }
        }
    }
```

**Effect:** Gives the reconnect window (typically 0.5–5s) a chance to complete before declaring the stream dead. During this grace period, if frames arrive (post-reconnect), `isStarved` flips `false` and the timer is cancelled.

**Risk:** Very low — 8s delay before teardown is invisible to the user (they're already hearing silence).

### 3.2 Auto-seek back after reconnect (medium risk)

When a reconnect lands after `isStarved` was `true` (implying the buffer ran dry), automatically seek back a few seconds on the server to fill the gap. This mimics the user's manual "skip back 10s" recovery.

**File:** `SendspinClient.kt` (post-reconnect handler in `runStateMachine`)

```kotlin
// After successful reconnect + auth/hello, check if we were starved
is WebSocketState.Connected -> {
    when (_state.value) {
        is SendspinState.Reconnecting -> {
            val reconnecting = _state.value as SendspinState.Reconnecting
            val wasStarved = audioPipeline.isStarved.value  // buffer was empty
            val reconnectAttempt = reconnecting.attempt
            // ... auth/hello ...
            if (wasStreaming && reconnectAttempt < THRESHOLD) {
                if (wasStarved) {
                    // Buffer was dry — request re-send from a safe offset
                    val seekBackSec = 3  // re-request 3s before starve point
                    sendCommand("seek_by", CommandValue(offset = -seekBackSec))
                    delay(500)  // let server process the seek
                }
                mediaPlayerController.resume()
            }
        }
    }
}
```

**Effect:** After a reconnect where the buffer ran dry, the client asks the server to re-send audio from 3 seconds before the starve point. This fills the gap seamlessly.

**Risk:** Medium — seek during reconnect adds latency; must not conflict with `stream/start` from the server.

### 3.3 Keep overlap buffer (higher complexity)

Instead of discarding frames with `ts <= lastConsumedTs`, keep the last 2–3 seconds of decoded PCM in a ring buffer so that after reconnect, the consumer can continue from where it left off while waiting for fresh server data. If no fresh data arrives within the ring buffer's duration, the pipeline transitions to "buffering" (like YouTube/Spotify).

**File:** `AudioStreamManager.kt`

```kotlin
// Ring buffer of decoded PCM, capacity = 3 seconds at current sample rate
private val pcmRing = PcmRingBuffer(capacityMs = 3_000)

// In consumer: before writing to MediaPlayerController, also write to ring
val pcm = audioDecoder.decode(frame.data)
pcmRing.write(pcm)
mediaPlayerController.writeRawPcm(pcm)

// After reconnect, if the queue is empty: play from ring while waiting
// for fresh frames
if (queue.size <= reorderDepth && pcmRing.available > 0) {
    // Feed the sink from ring buffer — old data is better than silence
    val chunk = pcmRing.read(chunkSize)
    mediaPlayerController.writeRawPcm(chunk)
}
```

**Effect:** Zero-gap recovery for short reconnects (<3s). For longer outages, the ring exhausts and the starve path kicks in.

**Risk:** Higher — adds a new buffer component, must be careful about memory (3s FLAC ≈ 250KB, 3s PCM 44.1kHz/16bit ≈ 260KB).

---

## 4. Implementation Plan

### Phase 1 — Grace Period (low risk, high impact)

| Step | File | Change |
|------|------|--------|
| 1a | `LocalPlayerController.kt` | Replace immediate teardown with 8s delayed teardown on `isStarved + Reconnecting` |
| 1b | `LocalPlayerController.kt` | Cancel the timer when any condition clears (`isStarved` flips false, transport recovers) |

**Verification:** Kill network while playing → pipeline stays alive for 8s → if network returns within 8s, playback continues without gap.

### Phase 2 — Auto-seek (medium risk)

| Step | File | Change |
|------|------|--------|
| 2a | `SendspinClient.kt` | After reconnect, check `audioPipeline.isStarved.value` before auto-resuming |
| 2b | `SendspinClient.kt` | If starved was true, send `seek_by: -3` to the server before resuming |
| 2c | `AudioStreamManager.kt` | On `processBinaryMessage` after a reconnect, reset `lastConsumedTs` to accept overlap frames |

**Verification:** Play → kill network 10s → restore network → audio resumes from ~3s before the gap, not from after it.

### Phase 3 — PCM Ring Buffer (highest complexity)

| Step | File | Change |
|------|------|--------|
| 3a | New file | Implement `PcmRingBuffer` — circular buffer with configurable ms capacity |
| 3b | `AudioStreamManager.kt` | Feed ring from consumer write path |
| 3c | `AudioStreamManager.kt` | On starve, attempt ring playback before declaring `isStarved = true` |
| 3d | `AudioStreamManager.kt` | Clear ring on explicit `stopStream`/`clearStream` |

**Verification:** Short network blip (1-2s) → no audible gap even without reconnect completing.

---

## 5. Test Scenarios

| # | Scenario | Expected | Phase |
|---|----------|----------|-------|
| T1 | Network drop 3s while playing | No gap, buffer covers it | 3 |
| T2 | Network drop 10s → reconnect within 8s | Playback resumes without teardown | 1 |
| T3 | Network drop 15s → reconnect | Starve timer fires → auto-seek back 3s → plays from gap position | 2 |
| T4 | Network drop 60s (all attempts exhausted) | Normal teardown, no auto-resume | 1 |
| T5 | Multiple rapid blips (1s each) | Ring buffer absorbs all, no starve reported | 3 |

---

## 6. Files

```
composeApp/src/commonMain/kotlin/io/music_assistant/client/data/LocalPlayerController.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/SendspinClient.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/audio/AudioStreamManager.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/audio/AudioPipeline.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/audio/PcmRingBuffer.kt  (new)
```
