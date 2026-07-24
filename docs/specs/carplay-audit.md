# CarPlay — Control & Resilience Audit

> **Spec:** CarPlay implementation review
> **Status:** Draft
> **Date:** 2026-07-24
> **Source:** `CarPlaySceneDelegate.swift`, `CarPlayContentManager.swift`, `SiriIntentHandler.swift`, `NowPlayingCoordinator.swift`, `NativeAudioController.swift`, `KmpHelper.kt`, `LocalPlayerDispatch.kt`, field log analysis

---

## 1. Summary

Audit of the CarPlay integration covering the iOS scene delegate, content manager, Siri intents, the KMP bridge, and the local-player dispatch path. Found **16 issues**: 2 bugs, 7 control-flow defects, and 7 resilience gaps.

---

## 2. Problem Statements

### 2.1 Subscription Leak on Rapid Reconnect

**Severity:** Bug · **Impact:** Memory leak, duplicate callbacks

**Observed:** `CarPlaySceneDelegate.didConnect()` assigns three subscriptions (`readinessSubscription`, `trackSubscription`, `modesSubscription`) without cancelling any previously-held values. If `didConnect` fires twice without an intervening `didDisconnect` — possible during a rapid CarPlay connection bounce — the old subscriptions are orphaned and their callbacks continue processing alongside the new ones.

**File:** `CarPlaySceneDelegate.swift` lines 98, 213, 220

**Fix:** Cancel each subscription before reassigning:
```swift
readinessSubscription?.cancel()
readinessSubscription = KmpHelper.shared.observeReadiness { ... }
```
Same pattern for `trackSubscription` and `modesSubscription` in `setupNowPlayingButtons()`.

**Risk:** None — `?.cancel()` on nil is a no-op.

---

### 2.2 Template-Stack Overflow on Rapid Taps

**Severity:** Bug · **Impact:** CarPlay crash / template rejection

**Observed:** `safePushTemplate()` checks `interfaceController.templates.count >= 5` as a snapshot before pushing. Two rapid pushes from different call paths (e.g. a recommendation-row tap arriving during `setupTemplates`) can both see `count < 5` and push concurrently, exceeding CarPlay's 5-template limit.

**File:** `CarPlaySceneDelegate.swift` lines 462–471

**Fix:** Serialise template pushes with an in-flight flag:
```swift
private var templatePushInProgress = false
private let templateQueue = DispatchQueue(label: "carplay.template")
```
Check and set the flag before each push; release on completion callback.

**Risk:** Low — serialisation only changes the timing, not the semantics.

---

### 2.3 `isReady` Gate Tolerant of Half-Open Transport

**Severity:** Control · **Impact:** User taps land on a dead transport with no feedback

**Observed:** `isReady` mirrors `serviceClient.isReadyForCommands`, which the field log shows can return `true` while the WebSocket is half-open (probe-timeout window). Items tapped during this window dispatch a play request that fails with `"Not connected"` — CarPlay shows no error, the user waits for playback that never starts.

**Log evidence (field report, line 2221):**
```
sendRequest FAILED: kotlin.IllegalStateException: Not connected
// while isReadyForCommands == true
```

**File:** `CarPlaySceneDelegate.swift` line 19 (isReady), `KtorServiceClient.kt` (connection probe path)

**Fix:** Strengthen `isReady` to also require a confirmed-Connected transport state, not just `isReadyForCommands`. Options:
- Observe `DirectTransport.state` directly and gate on `== Connected`
- Add a ping-probe before reporting `isReady = true` after a reconnect

**Risk:** Low — tighter gating only rejects taps that would have failed anyway.

---

### 2.4 `playOnLocalPlayer` Returns `true` on Silent Failure

**Severity:** Control · **Impact:** Now Playing pushes without playback

**Observed:** `executeLocalPlayerDispatch()` in `LocalPlayerDispatch.kt` sends `sendRequest()` and calls `.onFailure` to log it, but never propagates the failure to the caller. `KmpHelper.playOnLocalPlayer()` returns `true` regardless of whether the request actually reached the server. CarPlay pushes `CPNowPlayingTemplate.shared` in response — the user sees "Now Playing" while nothing plays.

**Log evidence (MA server log):**
```
Error calling tool 'play_media' — Tool execution timed out after 30.0s
```
CarPlay had already pushed Now Playing at this point.

**File:** `LocalPlayerDispatch.kt` lines 51–78, `KmpHelper.kt` line 434

**Fix:** Surface the failure back to the caller:
```kotlin
suspend fun executeLocalPlayerDispatch(…): Boolean {
    var ok = true
    plan.detachFrom?.let { syncedToId ->
        serviceClient.sendRequest(…).onFailure { ok = false }
    }
    serviceClient.sendRequest(…).onFailure { ok = false }
    return ok
}
```
Then gate `pushNowPlayingTemplate` on the response:
```swift
let ok = await KmpHelper.shared.playOnLocalPlayer(item: item, option: .play)
if ok { pushNowPlayingTemplate(animated: true) }
```

**Risk:** Medium — changes return type contract. Existing callers must handle `false`.

---

### 2.5 `pushNowPlayingTemplate` — Unsafe `pop(to:)` on Singleton

**Severity:** Control · **Impact:** Potential CarPlay runtime exception

**Observed:** `pop(to: CPNowPlayingTemplate.shared)` requires the template to already be in the navigation stack. A `contains()` check runs before the `pop()`, but the stack can change between the two calls (e.g. `safePushTemplate`'s `popToRootTemplate` fires concurrently). On some CarPlay versions, `pop(to:)` with a target not in the stack throws.

**File:** `CarPlaySceneDelegate.swift` lines 479–486

**Fix:** Use `firstIndex` instead of `contains`:
```swift
if let _ = interfaceController.templates.first(where: { $0 === CPNowPlayingTemplate.shared }) {
    interfaceController.pop(to: CPNowPlayingTemplate.shared, animated: animated, completion: logTemplateError)
}
```

**Risk:** Low — defensive check only, same semantics.

---

### 2.6 Template Update After User Has Popped

**Severity:** Control · **Impact:** Silent — update fires on detached template

**Observed:** `pushCategoryTemplate()` and `pushDrilldown()` push a loading template and then fire an async fetch. If the user taps Back before the fetch completes, the completion closure runs `template.updateSections(...)` on a template no longer in the navigation stack. No crash (the `CPListTemplate` object persists), but results are invisible.

**Same pattern in:** `loadRecommendations()` at lines 379–454

**File:** `CarPlaySceneDelegate.swift` lines 537–551, 647–660, 379–454

**Fix:** Capture the template weakly and verify stack presence before updating:
```swift
fetcher { [weak self, weak template] items in
    guard let self = self, let template = template,
          self.interfaceController?.templates.contains(where: { $0 === template }) == true
    else { return }
    // update sections…
}
```

**Risk:** Very low — guard only skips unnecessary work.

---

### 2.7 Empty Browse Grid on `carBrowseCategories` Timeout

**Severity:** Control · **Impact:** User sees a grid with zero buttons

**Observed:** `carBrowseCategories()` calls the KMP bridge which has a 5-second timeout. If the bridge times out (possible during initial connect), it returns an empty array. `categories` becomes empty → `CPGridTemplate` is created with 0 buttons. No error is surfaced.

**File:** `CarPlaySceneDelegate.swift` lines 513–514

**Fix:** Fall back to the default category set when the configured list is empty:
```swift
let configuredNames = manager.carBrowseCategories()
let categories: [CategoryEntry] = {
    let names = configuredNames.isEmpty ? Array(allCategories.keys) : configuredNames
    return names.compactMap { allCategories[$0] }
}()
```

**Risk:** Very low — only changes the fallback from "empty grid" to "full grid".

---

### 2.8 No Retry for Siri Affinity (Favorites)

**Severity:** Resilience · **Impact:** "I love this song" silently lost during reconnect

**Observed:** `KmpHelper.shared.setFavorite()` is fire-and-forget. If the transport is reconnecting when the user says "I love/like/dislike this song", the request is sent into a dead connection and silently dropped. Siri responds "Done" but MA never records the favourite.

**File:** `SiriIntentHandler.swift` (affinity handler), `KmpHelper.kt`

**Fix:** Add a bounded retry (max 2 attempts, 1s delay) in the affinity handler:
```swift
let dispatched = KmpHelper.shared.setFavorite(item, favorite: isLike)
if !dispatched {
    DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
        let retry = KmpHelper.shared.setFavorite(item, favorite: isLike)
        completion(INUpdateMediaAffinityIntentResponse(code: retry ? .success : .failure, userActivity: nil))
    }
}
```

**Risk:** Low — bounded retry, deferred to next runloop.

---

### 2.9 First CarPlay Play Never Donated to Siri

**Severity:** Resilience · **Impact:** First play in session has no Siri donation

**Observed:** `SiriIntentHandler.donatePlayed()` bails when `getServerId()` returns nil. On a cold CarPlay connect, the server handshake may not have completed by the time the user taps an item. The first play is never donated, so Siri does not learn from it.

**File:** `SiriIntentHandler.swift` lines 71–74

**Fix:** Defer the donation instead of skipping — retry when `serverId` becomes available:
```swift
guard let serverId = KmpHelper.shared.getServerId(), !serverId.isEmpty else {
    deferDonation(item)  // observes sessionState, fires on Connected
    return
}
```

**Risk:** Low — no behavioural change for the normal case.

---

### 2.10 Search Queries All Media Types Irrespective of Siri's Type Hint

**Severity:** Resilience · **Impact:** Unnecessary load on large libraries

**Observed:** `KmpHelper.shared.search(query:)` searches all 6 media types (artist, album, track, playlist, audiobook, radio) regardless of `INPlayMediaIntent`'s `preferredType`. If the user says "play album X", the search still queries all types; `bestMatch` then filters out non-album results. Wasteful for libraries with 10k+ tracks.

**File:** `SiriIntentHandler.swift` lines 382, 450, `KmpHelper.kt` (search)

**Fix:** Add a type-hint parameter to the search API:
```kotlin
fun search(query: String, typeFilter: MediaType? = null, completion: (List<AppMediaItem>?) -> Unit)
```
Pass through to the MA server's `mediaType` parameter when set.

**Risk:** Low — additive API change; existing callers pass `null` and keep current behaviour.

---

### 2.11 `connectionGen` Guard Only at Completion Entry

**Severity:** Resilience · **Impact:** Theoretically possible stale execution

**Observed:** The `connectionGen` guard runs once at the top of `loadCarPlayStrings`' completion closure. Between the guard check and the execution of `setupNowPlayingButtons()` / `setupTemplates()`, a rapid connect → disconnect → connect cycle could invalidate the generation. All operations after the guard then run on the old generation's state.

**File:** `CarPlaySceneDelegate.swift` lines 95–103

**Fix:** Check `connectionGen` before each state-modifying operation:
```swift
private func isCurrentGen(_ gen: Int) -> Bool { connectionGen == gen }
// Usage:
guard isCurrentGen(gen) else { return }
```

**Risk:** Very low — defensive pattern only.

---

### 2.12 Bulk Action Lacks True Feedback

**Severity:** Resilience · **Impact:** Same silent-failure pattern as 2.4

**Observed:** `playBulkAction()` inherits the same `Boolean` problem from `playOnLocalPlayer()`. The bulk-action path also pushes Now Playing based on a synchronous `true` that may have actually failed server-side.

**File:** `CarPlaySceneDelegate.swift` lines 682–690, `CarPlayContentManager.swift` lines 175–179

**Fix:** Share the same fix as 2.4 — propagate the actual dispatch outcome:
```swift
let ok = CarPlayContentManager.shared.playBulkAction(parent, actionName: actionName)
if ok { pushNowPlayingTemplate(animated: true) }
```

**Risk:** Same as 2.4.

---

### 2.13 `loadCarPlayStrings` Has No Timeout

**Severity:** Resilience · **Impact:** CarPlay stays stuck on loading state

**Observed:** `CarPlayStrings.load()` calls Compose resource resolvers that are normally sub-millisecond, but could theoretically hang on first app start after an update (resource cache rebuild). The `didConnect` closure never fires → `setupTemplates` never runs → CarPlay shows nothing.

**File:** `CarPlaySceneDelegate.swift` line 94, `CarPlayStrings.kt` line 65

**Fix:** Add a timeout with a hardcoded English fallback:
```swift
let gen = connectionGen
KmpHelper.shared.loadCarPlayStrings(timeoutMs: 5_000) { loaded in
    guard self?.connectionGen == gen, self?.interfaceController != nil else { return }
    self?.strings = loaded ?? CarPlayStrings.fallback
    self?.setupTemplates()
}
```

**Risk:** Very low — only affects the pathological case.

---

### 2.14 CarPlay Stuck on "No Connection" After Permanent Transport Failure

**Severity:** Control · **Impact:** App restart required to recover CarPlay

**Observed:** When `DirectTransport` exhausts its 20 reconnection attempts (~24 min), it enters `TransportState.Failed`. `isReadyForCommands` stays `false` permanently. `CarPlaySceneDelegate.handleReadinessChange()` sets `isReady = false`, which triggers the "disconnected row" affordance — but there is **no code path** that calls `forceReconnect()` or `onAppForeground()` from CarPlay. The transport stays dead until the user manually kills and restarts the app.

**Root cause:** The transport's `Failed` state is final — it only recovers through an explicit `connect()` or `onAppForeground()`. CarPlay never triggers either. The phone's screen may be off (CarPlay alone is active), so `onAppForeground` never fires.

**File:** `CarPlaySceneDelegate.swift` lines 136–153 (handleReadinessChange), `KtorServiceClient.kt` (reconnect entry points)

**Fix:** When `handleReadinessChange` observes `isReady` staying `false` after it was previously `true` (connection lost), schedule a reconnection probe:
```swift
private func handleReadinessChange(_ ready: Bool) {
    ...
    if !ready && wasReady {
        // Transport failed while CarPlay is active — schedule a reconnect probe
        DispatchQueue.main.asyncAfter(deadline: .now() + 5.0) { [weak self] in
            guard let self = self, !self.isReady else { return }
            KmpHelper.shared.reconnectFromCurrent()
        }
    }
}
```
Add `fun reconnectFromCurrent()` to `KmpHelper.kt` that calls `serviceClient.onAppForeground()` to kick the reconnection loop.

Alternatively: make the `Failed` state auto-retry with a long backoff (5 min → 15 min → 60 min) instead of being terminal.

**Risk:** Low — scheduled retry is bounded and only fires when CarPlay is active.

---

### 2.15 Now Playing Title/Position Desync on Track Change

**Severity:** Control · **Impact:** User sees wrong title or position on CarPlay screen

**Observed:** Two compounding causes:

**A) Artwork deferral:** `NowPlayingCoordinator.handleTrack()` defers the metadata write until artwork is loaded. If artwork takes 2-5 seconds (see 2.16), the lock screen and CarPlay show the **previous track's** title during that window. For short tracks or podcast segments with frequent transitions, the label is permanently behind.

**B) Lost transport anchor after artwork-less update:** When artwork is skipped (no URL, nil) or cache-hit (same URL), `applyTrackKeys` runs with `rebuildingGroup = true`. This rebuilds the info center keys — including position — by calling `setTransport` from `lastTransport`. But `lastTransport` is only available if the transport channel already emitted for this track. If the track channel fires before the transport channel (normal ordering), `lastTransport.mediaItemId != track.mediaItemId`, the restore is skipped, and the position stays at the old value.

**Log evidence (field report, line 1233):**
```
Info center assign: 8 keys title=Finish Line rate=1.00 elapsed=0.4
// Correct — but only after artwork arrived. Previous seconds showed old title.
```

**File:** `NowPlayingCoordinator.swift` lines 155–226 (track handler), 280–322 (transport handler), 235–263 (applyTrackKeys)

**Fix for A:** Set a timeout on artwork deferral (max 2s), then present with placeholder artwork:
```swift
// In handleTrack(), line 193:
let deferWrite = isNewTrack || artworkWaitTrackId == track.mediaItemId
if deferWrite {
    artworkWaitTrackId = track.mediaItemId
    // Timeout: present metadata even without artwork after 2 seconds
    DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) { [weak self] in
        guard let self = self, self.artworkWaitTrackId == track.mediaItemId else { return }
        self.applyTrackKeys(track, artwork: nil, rebuildingGroup: true)
        self.artworkWaitTrackId = nil
    }
}
```

**Fix for B:** In `applyTrackKeys` with `rebuildingGroup = true`, always set the elapsed time — even without a matching `lastTransport` — from the track's own startup position (0.0 for new track, or the server-provided `elapsedSec`):
```swift
if rebuildingGroup, let last = lastTransport, last.mediaItemId == track.mediaItemId {
    // restore from cached transport…
} else if rebuildingGroup {
    // New track, no transport anchor yet — set position to 0 / start
    infoStore.setTransport(elapsedSec: track.startPosition?.doubleValue ?? 0.0, rate: 1.0)
}
```

**Risk:** Low — presenting metadata earlier can only improve the user experience.

---

### 2.16 Artwork Cache Misses on Every Load

**Severity:** Resilience · **Impact:** Images reload from network on every appearance

**Observed:** `CarPlayImageLoader` (line 10) uses `NSCache<NSString, UIImage>` with **no disk cache** and **no cost limit**. Three compounding causes:

**A) Cache volatility:** `NSCache` is pure in-memory. Under memory pressure — common in a car with CarPlay + navigation + music running — iOS evicts the entire cache. Reload from scratch on next appearance.

**B) URL variance:** The cache key is `urlString` from `item.image(type: .thumb)?.url`. MA server URLs may include auth tokens, expiry timestamps, or session-bound paths. If the URL changes between sessions (e.g. token refreshed), the cache key changes too — even for the same artwork.

**C) WebRTC proxy latency:** `loadArtworkBytes` routes through the KMP WebRTC HTTP proxy (`webrtcHttpClient`). In CarPlay mode the phone is typically on cellular, adding 200-500ms per request. Combined with A and B, every grid row reloads artwork from the server.

**File:** `CarPlayImageLoader.swift` lines 12–29, `KmpHelper.kt` lines 82, 460+

**Fix:** Three-tier cache strategy:
```swift
class CarPlayImageLoader {
    // Tier 1: In-memory (fast, volatile)
    private let memCache = NSCache<NSString, UIImage>()
    // Tier 2: File-system (persistent across sessions)
    private let diskCache = DiskCache(name: "carplay-images", limitMB: 50)
    // Tier 3: Network

    func loadImage(from urlString: String, completion: @escaping (UIImage?) -> Void) {
        // 1. Normalise URL: strip query parameters beyond the path (remove auth tokens)
        let cacheKey = Self.normalisedKey(urlString)

        // 2. Check memory cache
        if let cached = memCache.object(forKey: cacheKey as NSString) { completion(cached); return }

        // 3. Check disk cache (async)
        if let data = diskCache.read(key: cacheKey), let image = UIImage(data: data) {
            memCache.setObject(image, forKey: cacheKey as NSString)
            completion(image); return
        }

        // 4. Network — load through KMP proxy, then write to both caches
        KmpHelper.shared.loadArtworkBytes(urlString: urlString) { [weak self] data in
            guard let data = data as Data?, let image = UIImage(data: data) else {
                DispatchQueue.main.async { completion(nil) }
                return
            }
            self?.memCache.setObject(image, forKey: cacheKey as NSString)
            self?.diskCache.write(key: cacheKey, data: data)
            DispatchQueue.main.async { completion(image) }
        }
    }

    /// Strip session-bound tokens from the URL so the same artwork always
    /// produces the same cache key.
    private static func normalisedKey(_ url: String) -> String {
        guard let components = URLComponents(string: url) else { return url }
        // Keep only path + essential query items; drop auth/session params
        var clean = URLComponents()
        clean.scheme = components.scheme
        clean.host = components.host
        clean.path = components.path
        return clean.string ?? url
    }
}
```

Set `memCache.totalCostLimit = 5_000_000` (≈5 MB, ~200 thumbnail-sized images) to prevent unbounded growth.

**Risk:** Low — additive caching, no behavioural change for the already-working first load.

---

## 3. Implementation Plan

### Phase 1 — Bug Fixes (low risk, high correctness)

| # | File | Change |
|---|------|--------|
| 2.1 | `CarPlaySceneDelegate.swift` | Add `?.cancel()` before each subscription reassignment in `didConnect` |
| 2.2 | `CarPlaySceneDelegate.swift` | Serialise template pushes with `templatePushInProgress` flag |

**Verification:** Rapid CarPlay connect/disconnect → no duplicate callbacks, no stack-overflow crash.

### Phase 2 — Control Flow (medium risk)

| # | File | Change |
|---|------|--------|
| 2.4 | `LocalPlayerDispatch.kt`, `KmpHelper.kt`, `CarPlayContentManager.swift`, `CarPlaySceneDelegate.swift` | Propagate real dispatch outcome; gate Now Playing push on success |
| 2.3 | `CarPlaySceneDelegate.swift` | Strengthen `isReady` with transport-state confirmation |
| 2.14 | `CarPlaySceneDelegate.swift`, `KmpHelper.kt` | Auto-reconnect probe when CarPlay sees persistent `isReady = false` |
| 2.15 | `NowPlayingCoordinator.swift` | Artwork deferral timeout (2s); set track start position in `applyTrackKeys` |
| 2.5 | `CarPlaySceneDelegate.swift` | Safe `pop(to:)` — replace `contains` with `firstIndex` |
| 2.7 | `CarPlaySceneDelegate.swift` | Default-category fallback when configured list empty |
| 2.6 | `CarPlaySceneDelegate.swift` | Weak template capture + stack check before `updateSections` |
| 2.12 | `CarPlaySceneDelegate.swift` | Gate bulk-action Now Playing push on actual dispatch |

**Verification:** Tap while disconnected → offline alert, not silence. Template update after back-navigation → no-op.

### Phase 3 — Resilience (low risk, additive)

| # | File | Change |
|---|------|--------|
| 2.16 | `CarPlayImageLoader.swift` | Three-tier cache (memory + disk + network), URL normalisation, cost limit |
| 2.8 | `SiriIntentHandler.swift` | Retry on affinity-set failure |
| 2.9 | `SiriIntentHandler.swift` | Deferred donation when `serverId` not yet available |
| 2.10 | `KmpHelper.kt`, `SiriIntentHandler.swift` | Type-hint filter in search API |
| 2.11 | `CarPlaySceneDelegate.swift` | Per-operation `connectionGen` guard |
| 2.13 | `CarPlaySceneDelegate.swift`, `CarPlayStrings.kt` | Timeout + English fallback for string loading |

**Verification:** Affinity during reconnect → retries. First CarPlay play → eventual donation. Timeout → full default grid.

---

## 4. Test Scenarios

| # | Scenario | Expected | Phase |
|---|----------|----------|-------|
| T1 | Rapid connect → disconnect → connect | No duplicate callbacks, no leak | 1 |
| T2 | Rapid tap two items before first loads | Template stack ≤ 5, no crash | 1 |
| T3 | Tap play while transport half-open | Offline alert, not silent failure | 2 |
| T4 | Play dispatch fails server-side | Now Playing NOT pushed | 2 |
| T5 | Tap item, then immediately tap Back | No crash, no update on detached template | 2 |
| T6 | `carBrowseCategories` times out | Grid shows default full set | 2 |
| T7 | Siri "I love this song" during reconnect | Retries, eventually succeeds | 3 |
| T8 | Cold CarPlay connect, tap first item | Donation fires after `serverId` arrives | 3 |
| T9 | Siri "play album X" | Search limited to albums | 3 |
| T11 | Transport fails (20 attempts exhausted) while CarPlay active | Auto-reconnect probe fires after 5s, recovers | 2 |
| T12 | Rapid track changes with slow artwork (2-5s each) | Metadata visible within 2s, placeholder art until real art loads | 2 |
| T13 | Browse grid, go back, browse again same row | Artwork loads from cache (<50ms), no network request | 3 |
| T14 | Force memory warning → browse grid again | Artwork reloads from disk cache, not network | 3 |

---

## 5. Files

```
iosApp/iosApp/CarPlay/CarPlaySceneDelegate.swift
iosApp/iosApp/CarPlay/CarPlayContentManager.swift
iosApp/iosApp/CarPlay/CarPlayImageLoader.swift
iosApp/iosApp/CarPlay/SiriIntentHandler.swift
iosApp/iosApp/NowPlayingCoordinator.swift
composeApp/src/commonMain/kotlin/io/music_assistant/client/data/LocalPlayerDispatch.kt
composeApp/src/iosMain/kotlin/io/music_assistant/client/di/KmpHelper.kt
composeApp/src/iosMain/kotlin/io/music_assistant/client/carplay/CarPlayStrings.kt
```
