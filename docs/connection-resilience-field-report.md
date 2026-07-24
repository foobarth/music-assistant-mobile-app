# Connection Resilience — Field Report

> **Datum:** 2026-07-24
> **Quelle:** MA Mobile App Client Log (iPhone, 3944 Zeilen, ~2,3h Session)
> **Build:** Post-PR #785 (upstream/main, commit `e14bb03`)
> **Szenario:** Podcast-Wiedergabe (Binärgewitter Talk), mehrfach Pause gedrückt, App backgrounded/foregrounded, Netzwerk-Wechsel (WiFi/Cellular/WireGuard)

---

## TL;DR

**Pause funktioniert einwandfrei.** Das Problem liegt im Zusammenspiel von **Play nach Pause + iOS Background + Netzwerkwechsel**. Die PR #785 Patches decken 3 von 6 identifizierten Problemen ab. Drei Issues bleiben offen.

---

## Timeline

| Zeit | Event | Code-Stelle |
|------|-------|-------------|
| +0s | Session-Start, WebSocket verbunden, "80s80s Rock" Radio | |
| +8,5s | **Audio output device disconnected** → WebSocket Error Code=53 | iOS System |
| ~+30s | iPhone im Tiefschlaf (40 min kein Netzwerk) | |
| +26min | Verbindung wieder da → Podcast "Die Nachrichten" startet | |
| +28:55 | **Pause #1** — sauber, 71ms | ✅ `NativeAudioController.pauseSink` |
| ~+60min | Ping Timeout → WS tot → DNS-Fehler (Code=-1003) × 8 | ❌ Reconnect-Loop |
| +48min | Play-Taste während Disconnect → **gequeued**, nach Reconnect ausgeführt | ✅ `Draining 1 queued commands` |
| +56:35 | **Pause #2** — sauber, 31ms | ✅ |
| +60:20 | **Pause #3** — sauber | ✅ |
| +64min | Play aus Pause → **gleichzeitig WS disconnect** → Background → Queue → Timeout | ⚠️ `Pending play timed out` |
| +67min | **play_pause (Toggle!)** nach Foreground + Reconnect → **resumed** | ⚠️ `play_pause` statt `play` |
| +73min | **Pause** (letzte Aktion) → sauber | ✅ |
| Log-Ende | Foreground probe (1s) → timeout → force reconnect | ❌ |

---

## Befunde

### 1. ✅ HttpClient Rotation (PR #785 — `3af3a9a`)
**Log-Evidenz:** Nach Ping Timeout 8× `Code=-1003 "A server with the specified hostname could not be found."` — DNS-Ergebnis aus dem alten NSURLSession-Pool.

**Patch:** `KtorServiceClient.kt` — `currentClient` als rotierbares `var`, `NetworkMonitor` beobachtet `nw_path` und rotated auf neue Network-Availability. `DirectTransport.kt` — `clientProvider`-Lambda statt festem `client`.

**Status:** ✅ **Im Log war die alte Version (ohne Rotation) aktiv.** Der Patch adressiert genau das Problem. Sobald deployed, sollten die 8 sinnlosen DNS-Retries entfallen.

---

### 2. ✅ 20-Attempt Backoff (PR #785 — `3af3a9a` + `752cac8`)
**Log-Evidenz:** Alter Code: 10 Attempts mit 60s-Deckel → ~3 Minuten Fenster. Im Log war nach 10 Attempts die Verbindung noch nicht zurück, danach `TransportState.Failed` (implizit).

**Patch:** `ReconnectBackoff.kt`
- `DEFAULT_MAX_RECONNECT_ATTEMPTS`: 10 → **20**
- Cap Delay: 60s → **120s**
- Drei-Phasen-Backoff: 0→4s (Blips), 8→120s (Ausfälle), 120s (steady)
- Total Window: **~24 Minuten**

**Status:** ✅ **Adressiert.** Der log zeigte den alten 10er-Limit. Mit 20 Versuchen + 120s-Deckel wird selbst ein längerer WireGuard-Tunnel-Drop überlebt.

---

### 3. ✅ Auto-Resume nach Reconnect (PR #785 — `eed8c20` + `bb89338`)
**Log-Evidenz:** Nach Reconnect wurde der gequeuede Play-Befehl gedrained (`Draining 1 queued commands`). Aber: das war nur der Fall wenn ein `players/cmd/play` in der Queue war. Die App resumed NICHT automatisch wenn sie vor Disconnect aktiv gespielt hat — der User musste Play tippen.

**Patch:**
- `SendspinClient.kt`: `wasStreaming`-Flag aus `Reconnecting`-State → `mediaPlayerController.resume()` nach reconnect
- Begrenzung auf `attempt < 9` (~4 Minuten). Danach kein Auto-Resume mehr (Server könnte rebootet sein)
- `resume()` = `resumeSink()` + `onRemoteCommand?.invoke("play")` — **explizit "play", nicht Toggle**

**Status:** ✅ **Klare Verbesserung.** Was im Log als "User tippt Play → queued → drained" funktionierte, wird jetzt automatisch für kurze Unterbrechungen (<4min) gemacht. Die 4-Minuten-Grenze ist sinnvoll (review-Feedback formatBCE).

---

### 4. ❌ App-Foreground Probe zu aggressiv
**Log-Evidenz (mehrfach):**
```
Direct connection probe timed out: probeReason=app_foreground timeoutMs=1000 messageCounterBefore=50 messageCounterAfter=50 state=Connected
→ sofort: Direct reconnect initiated: reason=probe_timeout:app_foreground
```
Der 1-Sekunden-Probe killt die Verbindung jedesmal wenn er kein Pong bekommt. Bei 40-80ms RTT sollte 1s eigentlich reichen — aber wenn das iPhone gerade aus dem Tiefschlaf kommt und der WebSocket noch nicht responsive ist, schlägt er fehl.

**Problem:** Kurzes App-Wechseln (z.B. Nachricht lesen, 5s weg) → 100% Force-Reconnect. Das ist teuer (auth + queue refresh + state sync).

**Vorschlag:**
- Probe-Timeout auf 2-3s erhöhen, oder
- Grace Period von 500ms vor dem eigentlichen Probe, oder
- Auf den Probe ganz verzichten und direkt den vorhandenen WebSocket nutzen — wenn er tot ist, fliegt der nächste Ping eh raus

**Datei:** `composeApp/src/commonMain/kotlin/io/music_assistant/client/api/DirectTransport.kt`

---

### 5. ❌ play_pause als Toggle (iOS Remote Control)
**Log-Evidenz (Z. 1941):**
```
ServiceClient: #5a59c30a players/cmd/play_pause target=[REDACTED_ID] command sent
```
Nach Background + Reconnect hat iOS den Play-Button als `play_pause` (Toggle) gemapped. Der reconnectete Client hat den gequeueden Toggle ausgeführt → **hat resumed, weil Client-State "paused" war.** Aber wenn Client-State "playing" wäre (weil Auto-Resume aus 3. schon gefeuert hat), würde der Toggle **pausieren**.

**Problem:** `NativeAudioController.kt` mappt iOS Remote Commands. Der Play-Button in Control Center / Kopfhörer liefert `play_pause`, nicht `play`.

**Vorschlag:**
- `NativeAudioController` sollte nach reconnect/background immer `play` senden, nicht `play_pause`
- Oder den Toggle durch State-Check ersetzen: wenn `isSinkActive || isStreaming → "play"`, sonst Sonderfall

**Datei:** `composeApp/src/iosMain/kotlin/io/music_assistant/client/player/NativeAudioController.kt`

---

### 6. ❌ Ping-Timeout im iOS Background
**Log-Evidenz (mehrfach):**
```
kotlinx.io.IOException: Ping timeout
```
Nach ~40s im Background stirbt der WebSocket durch ausbleibende Pongs. iOS pausiert Netzwerk-Sockets im Background (ausser für voip/audio/background-fetch).

**Problem:** Der 10s Ping-Intervall ist sinnvoll für aktive Verbindungen. Aber iOS kann keine WebSocket-Pings beantworten wenn der Prozess suspended ist.

**Das machen andere Apps:**
- **Spotify:** Verwendet `URLSessionWebSocketTask` mit `URLSessionConfiguration.waitsForConnectivity = true` + längerem Timeout
- **Overcast (Podcasts):** Schaltet bei Background in einen "low-power polling mode" statt WebSocket am Leben zu halten

**Vorschlag:**
- Ping-Timeout im Background deaktivieren oder auf 120s+ verlängern
- Oder bewusst reconnecten beim Foreground statt den Ping im Background zu halten (aktuell tut die App ja genau das — der Ping killt den WS → reconnect, das kostet nur extra Runden)

**Datei:** `composeApp/src/commonMain/kotlin/io/music_assistant/client/api/KtorServiceClient.kt` (Zeile: `pingInterval = 10.seconds`)

---

## Cross-Reference: PR #785 Commits

| Commit | Änderung | Deckt Befund |
|--------|----------|--------------|
| `3af3a9a` | HttpClient Rotation + 20 Attempts + NetworkMonitor | ✅ **1** (DNS-Stale) + **2** (Backoff) |
| `752cac8` | Infinite Retry Mode (Option), Backoff-Index-Cap | ✅ **2** (Backoff) |
| `eed8c20` | Auto-resume via `wasStreaming` + `mediaPlayerController.resume()` | ✅ **3** (Auto-Resume) |
| `bb89338` | Auto-Resume auf <4min begrenzt (attempt < 9) | ✅ **3** (Auto-Resume Guard) |
| `4af070e` | Lint-Fixes | — |
| `089ca90` | KMP Compile-Fixes (@Volatile, distinctUntilChanged, client ref, minOf) | — |

## Offene Issues (nicht gepatched)

| # | Problem | Schwere | Datei |
|---|---------|---------|-------|
| **4** | App-Foreground Probe zu aggressiv (1s) | Mittel | `DirectTransport.kt` |
| **5** | `play_pause` statt `play` nach Reconnect | Niedrig | `NativeAudioController.kt` |
| **6** | Ping-Timeout killt WS im Background | Niedrig | `KtorServiceClient.kt` |

---

## Empfehlungen

1. **Probe-Timeout von 1s → 3s erhöhen** (Punkt 4) — einfach, risikoarm, spart ~80% der unnötigen Reconnects
2. **`play_pause` durch `play` ersetzen** in `NativeAudioController.remoteCommand` nach reconnect/foreground (Punkt 5) — verhindert versehentliches Pausieren
3. **Background Pong-Timeout evaluieren** (Punkt 6) — entweder Ping deaktivieren oder Timeout auf >60s setzen
