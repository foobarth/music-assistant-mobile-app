# Reconnect Resilience — Edge Cases

> **Spec:** Post-PR #785 follow-up — implemented in `pr/reconnect-edge-cases`
> **Status:** Implemented (2026-07-24)
> **Date:** 2026-07-24
> **Source:** `docs/connection-resilience-field-report.md` (field log analysis)

---

## 1. Summary

PR #785 ("fix: improve reconnect resilience, HttpClient rotation, 20-attempt backoff, auto-resume") addressed three core issues:

- HttpClient rotation on network change
- 20-attempt backoff with ~24 min window
- Auto-resume after reconnect (attempt < 9)

Field testing with a ~2.3h real-world log revealed **three remaining gaps** that prevent full recovery in common edge-case scenarios. This spec documents those gaps and proposes targeted fixes.

---

## 2. Problem Statements

### 2.1 Foreground Connection Probe Timeout

**Observed behaviour (multiple instances in log):**

```
Direct connection probe timed out: probeReason=app_foreground timeoutMs=1000
  messageCounterBefore=50 messageCounterAfter=50 state=Connected
→ Direct reconnect initiated: reason=probe_timeout:app_foreground
```

After any foreground transition (app switch, notification reply, quick lock/unlock), `DirectTransport` performs a 1-second connection probe. When no message counter change is observed within that window — which happens reliably when the WebSocket is still warming up from iOS background suspension — the probe times out and triggers a **full reconnect cycle**: teardown → auth → queue refresh → state sync.

**Impact:** Every brief interruption (5-30s background) costs ~2-5 seconds of reconnect overhead. High churn on auth, queue state, and clock sync.

**Root cause:** The 1-second probe window is too tight for iOS background → foreground transitions where the underlying NSURLSession/WebSocket may take 500-1500ms to become responsive again.

**Proposed solution:** Increase probe timeout from 1s to **3s**, and add a 500ms grace delay before the first probe message to let the socket settle.

**File:** `composeApp/src/commonMain/kotlin/io/music_assistant/client/api/DirectTransport.kt`

**Risk:** Very low — a 3-second delay before declaring a reconnect is still imperceptible to the user (they already waited through the foreground transition).

---

### 2.2 `play_pause` Toggle Instead of Explicit `play`

**Observed behaviour:**

```
Z. 1941: ServiceClient: #5a59c30a players/cmd/play_pause target=[REDACTED_ID] command sent
```

iOS Control Center, headset play button, and CarPlay all deliver a **toggle** (`play_pause`) rather than an explicit `play` or `pause`. When the auto-resume feature (`eed8c20`) also fires during the same reconnect, the system can send two conflicting commands:

1. Auto-resume from `SendspinClient` → sends explicit `play`
2. User-triggered `play_pause` from `NativeAudioController` → toggles → could pause

Even without auto-resume: if the client state is `playing` but the network state is `paused` (or vice versa), a toggle sends the wrong direction.

**Root cause:** `NativeAudioController` maps iOS `MPRemoteCommandCenter.playCommand` → `"play_pause"` instead of `"play"`.

**Proposed solution:** Replace the `play_pause` mapping with explicit `play` for the play command and explicit `pause` for the pause command. The toggle concept exists for the play/pause button in custom UI, but the iOS remote control sends separate `playCommand` and `pauseCommand` events — they should stay separate.

**File:** `composeApp/src/iosMain/kotlin/io/music_assistant/client/player/NativeAudioController.kt`

**Risk:** Low — `play` on an already-playing player is idempotent on the server side (no-op). Same for `pause` on paused.

---

### 2.3 Background Ping Timeout / `wasStreaming` Lost

**Observed behaviour (two compounding issues):**

**A) Ping kills WebSocket in background:**
```
kotlinx.io.IOException: Ping timeout
```
The 10-second ping interval is healthy during active use, but iOS suspends network access for backgrounded apps (except voip/audio/background-fetch). When the app is backgrounded for more than ~40 seconds, the ping timeout fires and the WebSocket connection dies. This triggers an unnecessary reconnect cycle on next foreground.

**B) `wasStreaming` is not preserved through background disconnect:**
SendspinClient → `Transport→Reconnecting while backgrounded → Disconnected.Backgrounded`. The `Reconnecting` state (which carries `wasStreaming`) is skipped entirely, transitioning directly to `Disconnected.Backgrounded`. After foreground reconnect, the auto-resume feature cannot fire because `wasStreaming` is `false`.

```
Z. 1122: Transport→Reconnecting while backgrounded → Disconnected.Backgrounded
Z. 1124: Backgrounded — preserving 4 players as Stale(RECONNECTING)
...
Z. 1150: Seamless recovery - reusing cached data
// No auto-resume — wasStreaming was never set
```

**Impact:** After background → disconnect → reconnect, playback never resumes automatically. The user must manually tap play, even for a 2-minute coffee break.

**Proposed solutions:**

**For A (Background ping):** Either:
- Disable the WebSocket ping interval when the app backgrounds, or
- Increase the accepted pong timeout window from ~40s to ≥120s for backgrounded sessions, or
- Accept that the WebSocket dies in background and rely on foreground reconnect + auto-resume (leaning on fixing B)

**For B (wasStreaming persistence):** Store the streaming state before background disconnect:
- In `MainDataSource.SessionState` transition to `Backgrounded`, check if playback was active (`Playback active` in `ServiceClient.state`)
- Pass that flag through `Disconnected.Backgrounded` → next `Seamless recovery` so `SendspinClient` knows to auto-resume
- Alternative: store `wasStreaming` as a simple boolean field on the `SendspinClient` that is set on `stream/start` and cleared on `stream/end` / `playback inactive` — survives state transitions

**Files:**
- `composeApp/src/commonMain/kotlin/io/music_assistant/client/api/KtorServiceClient.kt` (ping interval)
- `composeApp/src/commonMain/kotlin/io/music_assistant/client/api/ServiceClient.kt` (playback state)
- `composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/SendspinClient.kt` (wasStreaming persistence)

**Risk:** Medium — changing ping behaviour could affect other platforms (Android). Keeping `wasStreaming` as a field is low-risk but requires testing the re-entry case (what if stream/end arrived while backgrounded?).

---

## 3. Implementation

### Phase 1 — Probe Timeout ✅ Implemented

| Step | File | Change |
|------|------|--------|
| 1a | `TransportState.kt` | Default timeout `1000` → `3000` |
| 1b | `DirectTransport.kt` | Added 500ms `delay()` before first probe ping |

### Phase 2 — `play`/`pause` Split ✅ Implemented

The iOS side already maps `playCommand` → `"play"` and `pauseCommand` → `"pause"` correctly
(`NowPlayingCoordinator.swift` lines 441-442). The fix was in the KMP command factory:

| Step | File | Change |
|------|------|--------|
| 2a | `PlayerRequestFactory.kt` | `TogglePlayPause` now resolves to `"play"` or `"pause"` based on `player.isPlaying` instead of always sending `"play_pause"` |
| 2b | `MainDataSource.kt` | Added clarifying comment for the server-player path (unchanged — server handles toggle correctly) |

### Phase 3 — `wasStreaming` Persistence ✅ Implemented

Low-risk approach: stored as a simple boolean field on `SendspinClient`.

| Step | File | Change |
|------|------|--------|
| 3a | `SendspinClient.kt` | Added `private var wasStreaming: Boolean = false` field |
| 3b | `SendspinClient.kt` | Set `true` on `stream/start`, `false` on `stream/end` |
| 3c | `SendspinClient.kt` | Post-reconnect handler uses the **field** instead of `reconnecting?.wasStreaming` (which is lost during `Backgrounded` transition). Field survives transport state changes |
| 3d | `SendspinClient.kt` | Cleared after successful auto-resume to prevent double-fire |

**Verification:** Start podcast → lock phone for 2 min → unlock → playback resumes automatically without user tap.

---

## 4. Test Scenarios

| # | Scenario | Expected | Phase |
|---|----------|----------|-------|
| T1 | App switch (5s background) → return | No reconnect, playback continues | 1 |
| T2 | Lock phone (30s) → unlock | Probe succeeds (3s), no reconnect | 1 |
| T3 | Control Center: tap play while playing | Log: "play", not "play_pause", server no-op | 2 |
| T4 | Control Center: tap play while paused | Log: "play", playback resumes | 2 |
| T5 | Listen to podcast → background 5 min → foreground | Auto-resume fires within attempt < 9 | 3 |
| T6 | Listen to podcast → background 30 min → foreground | Auto-resume **skipped** (attempt ≥ 9) | 3 |
| T7 | Track ends while backgrounded → foreground | No auto-resume (wasStreaming = false) | 3 |
| T8 | Multiple rapid foreground/background cycles | No crash, no reconnection storm | 1+3 |

---

## 5. Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Phase 1: 3s probe delays actual disconnect detection when network is genuinely gone | Low | The probe only applies to `probeReason=app_foreground` — network-loss detection uses the ping interval (10s), unaffected |
| Phase 2: iOS version quirk where playCommand and pauseCommand aren't separate | Very low | Standard since iOS 7.0. Verified per HIG |
| Phase 3: wasStreaming persists across session boundaries (app kill → restart) | Low | Field is in-memory only — reset on class init. App restart starts fresh |
| Phase 3: Race between stream/end and background transition | Medium | Check `ServiceClient.playbackActive` flag as secondary source of truth |

---

## 6. Related Code

```
composeApp/src/commonMain/kotlin/io/music_assistant/client/api/DirectTransport.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/api/KtorServiceClient.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/api/ServiceClient.kt
composeApp/src/commonMain/kotlin/io/music_assistant/client/player/sendspin/SendspinClient.kt
composeApp/src/iosMain/kotlin/io/music_assistant/client/player/NativeAudioController.kt
```

Reference: PR #785 commit `e14bb03` (5 squashed commits: `3af3a9a`, `752cac8`, `eed8c20`, `4af070e`, `089ca90`, `bb89338`).
