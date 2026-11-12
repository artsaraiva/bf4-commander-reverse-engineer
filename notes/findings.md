# BF4 Commander (Android) – findings

Lab notebook, in the order things were learned. Paths under `apk/`, `captures/`, `scripts/`, `extension/` and `issues/` exist only on
the player's disk (gitignored): EA files, live captures, and phase 1 tools and issue files. Issue numbers here are the legacy ones:
001–011 are phase 1, and loader issues 012–021 are GitHub #2–#11. `<serial>` is the phone's adb serial (`adb devices`).

## APK
- File: `apk/com.ea.game.warsawcommander_row-1.apk` (from trashbox.ru)
- SHA-256: `5c51e4e957bd67798338842672e1b84386978b8469961db598ca85972fcea6ed`
- Package `com.ea.game.warsawcommander_row`, versionName 1.0.0 (versionCode 2) — launch build
- minSdk 15, targetSdk 18, tablet only (`supports-screens large xlarge`), landscape
- Native code: `armeabi` only (32-bit ARM). Will NOT install on 64-bit-only emulators (API 33+ arm64 images, 16k images).
- Launch activity: `com.frostbite.firstcontact.GameActivity`
- Engine: "FirstContact" — Frostbite mobile framework. Almost all logic in `lib/armeabi/libFirstContact.Main_android.so` (16.8 MB, stripped, but C++ dynamic symbols are readable).
- Build path leak: `C:\monkey\Warsaw.Deploy.Android\...` (Warsaw = BF4 codename)

## Architecture (from symbols/strings)
Namespace `fb::TabletCommander`:
- `TabletCommanderBlazeMain` — Blaze client: `setupDirtySock`, `setupBlazeHub`, `onAuthenticated`, `joinServer(...)`, `joinGameCb`, GameManager callbacks (`onGameStarted`, `onPlayerJoining`, …)
- `TabletCommanderNetworkMain` — Frostbite game network connection (via Blaze `NetworkMeshAdapter`)
- `TabletCommanderLogicMain`, `TCMapController`, `Minimap`, many widgets (Squad list, Assets, Scoreboard, Chat, Push-to-talk, Mutiny…)
- `SettingsManager::setBlazeArguments / getBlazeArg` — launch args

So the tablet app is a real (stripped-down) Frostbite client speaking Blaze + Frostbite netcode, not a simple REST client.

## Login / launch flow (hypothesis)
1. App opens WebView on `http://battlelog.battlefield.com/bf4/commanderwarsaw/`
2. User logs into Battlelog, picks a server, presses Commander
3. Battlelog hands the app Blaze args: `-authenticationCode <code> -blazeServiceName <name> -blazeEnvironment <env>` (+ server/game id). Custom scheme seen: `se.dice.bf4.battlelog://`
4. Native: redirector (`gosredirector.ea.com`, `https://gosca.ea.com:44125/redirector`) → Blaze login with auth code → GameManager join → Frostbite netcode to game server

## Live checks (2026-11-10)
- `battlelog.battlefield.com/bf4/commanderwarsaw/` → 302 to `/bf4/gate/?returnUrl=|bf4|commanderwarsaw|` (route still exists, needs login)
- `gosredirector.ea.com` resolves (159.153.51.18) — BF4 PC still uses it
- `gosca.ea.com` resolves (159.153.51.21); HTTPS on :44125 no response to plain curl (likely legacy TLS / Blaze-specific)
- `peach.online.ea.com` no DNS (telemetry, irrelevant)

## Running the app – blockers
- Apple Silicon has no AArch32 → arm64 emulator images (API 30 tested) report only `arm64-v8a` → `INSTALL_FAILED_NO_MATCHING_ABIS`
- 32-bit ARM images (`android-25;google_apis;armeabi-v7a`) refused by emulator 36.6: `FATAL | CPU Architecture 'arm' is not supported by the QEMU2 emulator`
- App needs expansion OBB `main.2.com.ea.game.warsawcommander_row.obb` (~670 MB, superbundles/`layout.toc`) in `/sdcard/Android/obb/<pkg>/`; we don't have it. Launch with any intent extra but no `-super` skips the downloader (GameActivity.onCreate), but native side probably still needs the bundles.
- Real fix: physical Android device with 32-bit support (most devices on Android ≤ 12) + matching APK/OBB pair (APKPure XAPK bundles both)

## Battlelog side (2026-11-10)
- Battlelog is still deployed: CDN build `develop-20260605`
- UA containing `BattlelogCommander` gets the commander gate (`<html class="is-touch is-commander is-android">`), old-style email/password form posting to `/bf4/gate/login/`
- Touch bundle has commander code: `touch.device.launchCommander`, `commanderPlatformString` (pc/x360/ps3/xbo/ps4), `/commanderwarsaw/muted/` endpoint
- The app's WebView exposes `window.TabletCommander` with: `onJavaScriptEvent(event, action)`, `joinServer(authCode, blazeServiceName, gameId, personaId, platform, gameProtocolVersion, blazeEnvironment)`, `launchBattlelog(path)`, `launchBrowser(url)`, `sendEmail(...)`
- `scripts/tc-shim.js` fakes that bridge in a normal browser (Playwright + UA override) to capture what Battlelog would hand the app

## Result of the Battlelog test (2026-11-10)
- Login with real account via commander gate (UA `BattlelogCommander`) **works**
- `/bf4/commanderwarsaw/` now renders a sunset page: "MOBILE COMMANDERS, YOU SERVED THE BATTLE WELL" (+ untranslated `ID_WEB_COMMANDER_SUNSET_PART1..4` keys)
- Page loads only `bundle_commanderwarsaw_topbundles_3860401463.js` = Modernizr, nothing else. Server list + `TabletCommander.joinServer` logic is gone
- Fake bridge got zero calls. **Battlelog no longer hands out commander auth codes / game ids**
- `launchCommander` in the touch bundle has no callers (dead code left over from the Battlelog mobile app)

## CORRECTION: the Battlelog *mobile app* path still works (2026-11-10)
Source: Reddit guide (2025) using modded `BattlelogBF4_mod_dunem.apk` + `CommanderBF4_mod_dunem.apk` + `main.11.com.ea.game.warsawcommander_row.obb`.
- Commander mod: versionCode 11 / 2.2.0, armeabi only → matches `main.11` OBB (1,057,470,817 bytes)
- Battlelog app: `com.ea.bf3bl.inc` 2.3.0, hybrid WebView app. Native libs only for SmartGlass/QR (not needed for commander)
- Battlelog app WebView: loads `http://battlelog.battlefield.com/bf4/`, UA suffix `BattlelogMobile/<ver> (Language: en; Capabilities: ...; Dimensions: WxH; Density: d; )`,
  JS bridge `window.BattlelogMobile.sendCommandJson('{"cmd":..,"params":{..}}')`
- On `launchCommander` the app starts `com.ea.game.warsawcommander_row/com.frostbite.firstcontact.GameActivity` with every param as string extra `"-" + key`
  (`com/ea/bf3bl/tablet/MainActivity.java:369`)

### Reproduced in a desktop browser
Playwright + UA override (`BattlelogMobile/2.3.0`) + `extension/bl-mobile-shim.js`:
- Battlelog treats the browser as the hybrid app (`touch.device.isHybridApp === true`)
- Server browser → server with commander slot → orange Commander button → shim receives:
```
cmd: launchCommander
params: {
  action: "joinServer",
  authenticationCode: <40 chars>,
  blazeServiceName: "battlefield-4-pc",
  gameId: "<17 digits>",
  personaId: "<persona>",
  platform: "pc",
  gameProtocolVersion: 4900,
  blazeEnvironment: "prod"
}
```
So the tablet client logs into the **PC** Blaze instance (`battlefield-4-pc`, prod, protocol 4900) with a Battlelog-minted auth code.
Battlelog app is not needed at all: these params can be passed straight to Commander:
```
adb shell am start -n com.ea.game.warsawcommander_row/com.frostbite.firstcontact.GameActivity \
  --es -action joinServer --es -authenticationCode <code> --es -blazeServiceName battlefield-4-pc \
  --es -gameId <id> --es -personaId <id> --es -platform pc --es -gameProtocolVersion 4900 --es -blazeEnvironment prod
```

## MILESTONE: Commander playing a live match (2026-11-10 ~17:40)
Setup — no Battlelog app needed:
1. Galaxy S23 (Android 16, still has `armeabi-v7a,armeabi`): `adb install --bypass-low-target-sdk-block CommanderBF4_mod_dunem.apk`
   (the "mod" is EA's untouched 2.2.0 release: v1 signature verifies with EA cert SHA-256 `5bff7d61…f0d2`)
2. `adb push main.11.com.ea.game.warsawcommander_row.obb /sdcard/Android/obb/com.ea.game.warsawcommander_row/`
3. First launch: Android "review permissions" screen for legacy apps → continue
4. `python3 scripts/commander_bridge.py <serial>` (127.0.0.1:47615)
5. Playwright Chrome via `scripts/open-battlelog-as-app.js` (Playwright MCP `browser_run_code_unsafe`, `filename`):
   UA `... BattlelogMobile/2.3.0 (...)` + `extension/bl-mobile-shim.js`; player logs into Battlelog by hand,
   server browser → server with commander slot → orange Commander button
6. Shim POSTs launchCommander params → bridge → `adb shell am start ... --es -<key> <value>` → Commander joins the server
   (Chrome asks once for local-network access for the page → allow)
Result: full commander UI in a live PC server (map, squads, scans, cruise missile, scoreboard). Phone screen works fine (landscape).
Screenshot: `captures/s23-commander-ingame.png`
Gotcha: phone dropped off USB once → bridge got `adb: device not found`; reconnect fixed it.

## Chrome extension launcher (issue 001, 2026-11-10)
`extension/` (Manifest V3, unpacked) replaces the Playwright browser in step 5 above. It uses the player's everyday Chrome.
- `rules.json`: a declarativeNetRequest `modifyHeaders` rule sets the Battlelog-app User-Agent (the same string as the Playwright script)
  on `main_frame`, `sub_frame` and `xmlhttprequest` requests to `battlelog.battlefield.com`. `resourceTypes` must list `main_frame`
  explicitly, because a DNR rule without `resourceTypes` matches everything *except* main_frame.
- Why the header alone is enough: `touch.device.init` in `bundle_touch.js` reads `isHybridApp`, `isCommanderApp`, `appVersion` and
  `appCapabilities` from `S.globalContext.staticContext.detectedBrowser`, which the server fills in from the request UA. It only falls back to
  `navigator.userAgent` when `S.globalContext` is missing. `isAndroid` comes from `window.BattlelogMobile` existing. So `navigator.userAgent`
  is left alone.
- `bl-mobile-shim.js` (the only copy of the shim): a MAIN-world content script at `document_start`, so `window.BattlelogMobile` exists before
  Battlelog's scripts run. On `launchCommander` it POSTs straight to the bridge from the page, as before. There is no service-worker relay,
  so the bridge needs no change (it already accepts origin `https://battlelog.battlefield.com`). The cost is Chrome's one-time
  local-network-access prompt for battlelog.battlefield.com.
- No on/off toggle: for normal (PC) Battlelog, switch the extension off in `chrome://extensions`.

Verified live 2026-11-10 in the player's everyday Chrome: hybrid UI, Commander button, the phone launched Commander. Bridge log had one launch with the auth code masked to 4 chars. Battlelog's `launchCommander`/`sendCommand` log nothing, and the extension has no background context, so the shim (masked) is the only logger.

Install (once): `chrome://extensions` → turn on Developer mode → Load unpacked → pick the `extension/` folder.
Use: phone on USB → `python3 scripts/commander_bridge.py <serial>` → open https://battlelog.battlefield.com/bf4/servers/ in Chrome
(log in by hand) → server with a commander slot → orange Commander button → allow local network access the first time → Commander
joins on the phone.

## Browser-tab mirror (issue 002, 2026-11-10)
`scripts/mirror.py` + `scripts/mirror.html`: the phone's screen in a Chrome tab, with mouse input sent back. Python stdlib only, plus Homebrew `scrcpy`.
- **Engine:** the official scrcpy server (`brew install scrcpy`, which ships `/opt/homebrew/share/scrcpy/scrcpy-server`; 5.0.1 tested). The server only
  starts if its first argument equals its own version, so `mirror.py` reads the version from `scrcpy --version`. A `brew upgrade` keeps the two in step.
- **Flow per tab:** the page opens `ws://127.0.0.1:47616/ws`. `mirror.py` wakes the screen, sets up `adb reverse localabstract:scrcpy_<scid>`, and
  starts the server (`audio=false clipboard_autosync=false stay_awake=true send_device_meta=false`). The server connects back twice: video
  first, then control. Video packets go to the page one per WebSocket message. Control messages from the page go to the control socket unchanged.
  Closing or reloading the tab ends the session, and the server exits when its sockets close.
- **scrcpy 5.x stream format** (changed since 3.x): 4-byte codec id, then 12-byte headers. MSB set = *session packet* (new capture size or rotation,
  u32 width + u32 height at bytes 4..11, no payload). MSB clear = media packet: bit 62 config, bit 61 key frame, u61 PTS, u32 size, then H.264 Annex B.
- **Page:** WebCodecs `VideoDecoder` without `description` (= Annex B). The SPS/PPS config packet is prepended to the next key frame, and the codec string
  `avc1.PPCCLL` is read from the SPS. Frames are drawn to a canvas. The session packet's width×height is the size touch positions must use: the server
  ignores touches generated for a different size.
- **Input:** left button = one finger (scrcpy `INJECT_TOUCH_EVENT`, 32 bytes, buttons 0 so the server injects `TOOL_TYPE_FINGER` / touchscreen).
  Wheel or trackpad pinch = two virtual fingers (ids 100/101) around the cursor that spread or close, lifted after 150 ms idle (`ZOOM_SPEED` in the
  page is the tuning knob). Right click = Android Back (`INJECT_KEYCODE` 4).
- **Security:** listens on 127.0.0.1 only. The WebSocket accepts only Origin `http://127.0.0.1:47616` or `http://localhost:47616`, so other
  sites can't watch or drive the phone.
- **Phone settings:** none persistent. `stay_awake=true` keeps the screen on while the server runs and the phone is plugged in, and restores the
  setting on exit. Commander stays in the foreground, so Android has no reason to kill it. Battery-optimisation exemptions weren't needed.
- **ws-scrcpy (the first candidate) not used:** it needs an `npm install` with native builds (node-pty) and Appium post-install steps, and its server is a
  fork of scrcpy 1.19. The official server plus about 250 lines does the job.

Use: phone on USB → `python3 scripts/mirror.py <serial>` (also runs the commander bridge on :47615, so don't start that separately) → open
http://127.0.0.1:47616 in Chrome → launch Commander from Battlelog as in the section above → play in the mirror tab.
Native fallback: `scrcpy -s <serial>` (a desktop window, same server).

Verified live 2026-11-10: the player played a commander round in the mirror tab. The picture switched to landscape when Commander started,
abilities fired, wheel/pinch zoom worked on the map, and the delay felt fine over USB.

Known limitations:
- **Resolution is soft.** The app (targetSdk 18) is letterboxed inside the 2340×1080 frame: the game uses about 82% of the width, with black bars and
  the gesture-bar strip around it. The whole frame is then scaled to the window and sent as H.264 at scrcpy's default 8 Mbps. Untried fixes: the
  server's `crop=W:H:X:Y` option to drop the bars, and `video_bit_rate=16M`. Follow-up: issue 007. The page now crops to Commander's window
  (`GET /crop` → `dumpsys window com.ea.game.warsawcommander_row` → `frame=[l,t][r,b]`, polled every 2 s; touch positions are mapped back
  through the crop).
- **Latency** was not measured; the player reports it as fine on USB/localhost.
- **Gestures:** the mouse gives one finger. Pinch is emulated as a symmetric horizontal two-finger spread at the cursor, so there's no rotation and
  no two-finger pan. Long-press = hold the button. There's no keyboard passthrough (chat means tapping the phone's on-screen keyboard).
- **No audio, no push-to-talk** (`audio=false`; out of scope).
- **The phone must stay on USB with its screen on** (`stay_awake`), which costs battery and heat. If USB drops, the tab shows "disconnected,
  retrying…" and reconnects once the phone is back. Each extra tab starts its own server session.

## Conclusion so far (superseded by the correction above)
Battlelog handoff is dead, so the official app can't start a session even on real hardware.
Whether Blaze/game servers still accept a tablet client is untested: that needs an auth code, and minting one
would mean impersonating the commander app against EA's live login (client id from the APK) — not done.

## Native symbols: 1.0.0 vs 2.2.0
- 1.0.0 `apk/unzipped/lib/armeabi/libFirstContact.Main_android.so`: ~76k exported dynamic symbols (e.g. `ProtoSSLSend/Recv/Connect`, all `Blaze::` and `fb::TabletCommander::` names)
- 2.2.0 `apk/mod/unzipped/lib/armeabi/libFirstContact.Main_android.so` (the build that works with `main.11` OBB): 215 exported symbols, class-name strings gone
- So: use 1.0.0 as the symbol reference and diff into 2.2.0 to find functions to hook
- Decompiled Java: `apk/jadx` (1.0.0), `apk/mod/commander-jadx` (2.2.0), `apk/mod/battlelog-jadx` (Battlelog app 2.3.0)

## Next
Tracked as GitHub issues (see `AGENTS.md` → "Issues").

## Blaze session — static map (issue 003, 2026-11-10)
Derived from the 1.0.0 symbol-rich build; confirmed by the live capture (next section).

### Hook points (verified, module-relative in 2.2.0 `libFirstContact.Main_android.so`)
Found by masked-byte diffing 1.0.0 into stripped 2.2.0 (`scripts/fmap.py`). ARM, even addresses:
- `ProtoSSLSend`  @ **0x9a5bec** — `(r0=ssl, r1=buf, r2=len)`, buf = plaintext about to be encrypted (OUT)
- `ProtoSSLRecv`  @ **0x9a5cc4** — `(r0=ssl, r1=buf, r2=len)` -> n decrypted bytes into buf (IN)

Prologues are byte-identical to 1.0.0 (confirmed by disassembly). The semantic layer
`Blaze::ProtoFire::send(connId, componentId, commandId, MsgType, msgNum, Tdf*)` and
`Blaze::BlazeConnection::OnProtoFirePacketReceived(FireFrame*, payload)` are the nicer
targets (IDs already parsed) but **drifted too much between 1.0.0 and 2.2.0 to locate by
byte pattern** (C++, recompiled). Not worth a BinDiff/Ghidra hunt: ProtoSSLSend/Recv give
the whole plaintext Fire2 stream, and Fire2 headers are self-describing, so
`scripts/blaze_decode.py` re-frames on the host. Hook = `scripts/blaze_hook.js`.

### Client session flow (from `fb::TabletCommander::TabletCommanderBlazeMain` + LoginManager symbols)
Order the app drives (symbol names, so this is what to expect on the wire):
1. `setupDirtySock(env)` — DirtySDK / ProtoSSL startup
2. `setupBlazeHub(serviceName, env)` — `battlefield-4-pc`, `prod`
3. `setupConnectionManager()` → **Redirector**: look up the Blaze server for the service (over ProtoSSL to `gosredirector.ea.com` / `https://gosca.ea.com:44125/redirector`). `Blaze::Redirector::ServerAddress`/`ServerInstanceInfo` TDFs returned. Note: a `Blaze::Redirector::SunsetData` TDF exists in the SDK — watch for it (EA may return sunset info).
4. `setupLoginManager(...)` → **Authentication**: login with the Battlelog auth code. Path is `LoginStateBase::doAuth2Login(Authentication2::LoginRequest)` / `onStartOriginLoginProcess(token, TOKENTYPE)` — i.e. Nucleus/Origin token login, not user+password. Replies carry `Authentication::SessionInfo` / `AccountInfo`.
5. `onAuthenticated(userIndex)` — session established; `UserSessions` component active.
6. `setupGameManagerApi(..., HostedPlatform)` ; `toHostedPlatform(fb::Platform)` maps platform → **GameManager**.
7. `joinServer(authCode, serviceName, gameId, personaId, platform, env)` → GameManager join **as commander** → `joinGameCb` → `onGameStarted` / `onNetworkCreated`.
8. `onNetworkCreated` → `Blaze::BlazeNetworkAdapter::NetworkMeshAdapter` connects to the game server (issue 004); game-server connection info comes from the GameManager join reply.

### Components the client registers (createComponent, stock Blaze ids in the decoder dict)
Authentication, GameManager, Redirector, Stats, Util, CensusData, Clubs, Messaging,
AssociationLists, GameReporting, Rooms, Achievements, ByteVault, Inventory, UserSessions,
plus Playgroups, NucleusIdentity, Mail, Packs, Quarter, Locker, Rsp, CommerceInfo,
DynamicInetFilter, GpsContentController, MigrateData, XBLSystem (not all used in a session).

### Transport / framing
- EA **ProtoSSL** (DirtySDK custom TLS) over TCP. Redirector first, then the assigned Blaze server.
- **Fire** frames (12-byte header, live-confirmed; not Fire2): length + component + command + error + msgType + msgNum, then a **Heat1 TDF** body. `Blaze::ProtoFire` / `Blaze::FireFrame` / `Blaze::TdfEncoder`/`TdfDecoder` confirm it.
- Underneath the game-server connection: **ProtoTunnel** UDP (strings `prototunnel-global-send/recv`) — issue 004.

### Live capture decoded (2026-11-10)
The live capture is decoded below in "Blaze session — live capture". All four items that were open here are answered there:
- Frames: `captures/blaze/blaze_cap.ndjson`, with `AUTH`/`KEY` masked at write time by `blaze_capture.py`. The full RPC sequence is listed below.
- Header: **Fire, 12 bytes** (not "Fire2"). TDF: **Heat1**.
- TLS: offered set known; negotiated version and cipher still open.

## Blaze capture runbook (issue 003 — BUILT & VERIFIED 2026-11-10, player approved main account; phone back on the EA-signed app since 23:40)
Runs a **re-signed, Frida-instrumented Commander on the player's real EA account against
EA's live service.** Hooks only observe (no crafted traffic). Player gave go-ahead for the
main account.

### Gadget build (done — reproducible)
- Host frida 17.26.2 in `scripts/.venv` (gitignored): `python3 -m venv scripts/.venv && scripts/.venv/bin/pip install frida-tools`.
- Gadget: `frida-gadget-17.26.2-android-arm.so` (ARMv7 32-bit) — matches the host frida version and the app's `armeabi` ABI.
- Injection (apktool 3.0.3, no ELF surgery — leaves the 13 MB native lib untouched):
  1. `apktool d CommanderBF4_mod_dunem.apk`
  2. In `smali/com/frostbite/firstcontact/GameActivity.smali`, insert at the top of
     `loadFrostbiteLibrary()` (right after `.prologue`, uses v0):
     `const-string v0, "gadget"` / `invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V`
     — loads the gadget **before** `System.load(libFirstContact)`, so hooks are in place before Blaze connects.
  3. `lib/armeabi/libgadget.so` = the gadget; `lib/armeabi/libgadget.config.so` = JSON
     `{"interaction":{"type":"listen","address":"127.0.0.1","port":27042,"on_port_conflict":"fail","on_load":"wait"}}`
     (`.config.so` name so Android packs it; `wait` blocks the app at load until the host attaches).
  4. `apktool b` → `zipalign -p 4` (build-tools 34.0.0) → `apksigner sign --ks <throwaway> --v1-signing-enabled true`.
- Verified live: gadget loads, listens on in-app `127.0.0.1:27042`, `frida-ps -U` shows `Gadget`,
  and the host script hooks cleanly: `base=0xe4d0e000 send=…bec recv=…cc4 / hooks installed` (no crash).

### Install (done)
Re-signing breaks EA's signature, so **uninstall the EA app first (deletes the OBB)**:
`adb uninstall com.ea.game.warsawcommander_row` → `adb install --bypass-low-target-sdk-block commander-gadget.apk`
→ `adb push apk/mod/main.11.*.obb /sdcard/Android/obb/com.ea.game.warsawcommander_row/`.
First launch shows Android's one-time **REVIEW_PERMISSIONS** screen (blocks `GameActivity.onCreate`
→ gadget until dismissed); it's already been cleared on this phone.

### Capture round (NEEDS THE PLAYER — human does the Battlelog login; gadget waits so nothing is missed)
1. Phone on USB. `adb forward tcp:27042 tcp:27042` (host script also uses USB directly).
2. Start the bridge: `python3 scripts/mirror.py <serial>` (runs the commander bridge too) — or `commander_bridge.py`.
3. Start the capture, it waits for the gadget: `scripts/.venv/bin/python scripts/blaze_capture.py`
4. In Chrome (extension on): log into Battlelog, pick a server with a commander slot, click **Commander**.
   The app launches, the gadget blocks, the script attaches → `hooks installed` → resumes → Blaze login/join runs.
5. Play ~1 min, leave. Ctrl-C the capture. Frames land in `captures/blaze/blaze_cap.ndjson` (gitignored; auth code and
   session key masked at write time, lengths kept) and `captures/netcode/netcode_cap.ndjson` (game server; secrets masked by the hook).
6. Decode: `python3 scripts/blaze_decode.py captures/blaze/blaze_cap.ndjson [--brief]` (Blaze) and
   `python3 scripts/tc_decode.py captures/netcode/netcode_cap.ndjson` (game server, issue 004). A new run renames the
   previous capture files to `*.<timestamp>.ndjson` instead of overwriting them.
7. **Restore normal play:** `adb uninstall …` the re-signed app, reinstall EA-signed
   `CommanderBF4_mod_dunem.apk`, re-push the OBB, verify a normal browser launch still works.
   Gotcha (2026-11-10, Android 16): the first OBB push right after the reinstall reported success but the file vanished.
   The package's obb folder was gone, probably cleaned up after the uninstall. Fix: `adb shell mkdir -p /sdcard/Android/obb/com.ea.game.warsawcommander_row`,
   push again, then `adb shell ls -l` that folder to check the size (1,057,470,817 bytes).
   (Or keep the gadget build for 004/005 and restore later.)

## Game-server protocol (issue 004, 2026-11-10) — static map, confirmed by a live capture
Source: 1.0.0 symbols + reflection data, checked against the stripped 2.2.0 code. Tools:
`scripts/tc_protocol.py` (enums / member names / readFrom bit layouts straight from the libs),
`scripts/netcode_hook.js` (capture hooks, loaded by `blaze_capture.py`), `scripts/tc_decode.py` (decoder).

### It is NOT Frostbite ghost replication
The tablet speaks its own small protocol ("TabletCommander protocol", `TabletCommanderProtocol_Version`
= **38** in 2.2.0, 28 in 1.0.0). The server condenses game state into a few packet types. Every message class
has `readFrom`/`writeTo` built from named `fb::IBitStreamRead` primitives, and the classes and enums keep Frostbite
reflection tables (`c_fieldinfodata`), which gives field names, order, bit widths and enum values.
2.2.0 is stripped of symbols, but it keeps the enum/field name strings.

### Stack (what to undo, top to bottom)
1. `fb::TabletCommander::TabletCommanderNetworkMain` (`connectToServer(ConnApiClientT*, String, u64, char*)`,
   `sendData(buf, len, reliable)`, `receiveData()`, `handleReceivedPacket()`)
2. DirtySDK **ConnApi → NetGameLink → CommUDP → ProtoTunnel → UDP socket**. The game connection is a
   NetGameLink stream that ConnApi sets up from the Blaze join (live: ProtoTunnel is on, see "Live capture" below).
3. Online path (`byte [this+2140] != 0`): `sendData` builds a `NetGamePacketT` = `+4 u16 len, +6 u8 kind
   (5 = reliable, 6 = unreliable), +8 body` and calls `NetGameLinkSend(link=[this+2152], pkt, 1)`.
   **body = 8-byte tag + payload**. Outgoing tag is copied from `this+2156`. Incoming packets are accepted only
   if the tag is ASCII `tab.init`, and the payload is copied to `this+8` (max 1264 B).
   LAN path: raw `CommUDPSend/Recv`, no tag.
4. Payload = one TabletCommander bitstream packet (below).

Hook points, module-relative in 2.2.0 (fmap.py, verified by disassembly; `receiveData` 0x480004 is byte-identical
to 1.0.0 and calls 0x992e7c): `NetGameLinkSend` **0x992b94**, `NetGameLinkRecv` **0x992e7c**, `ProtoTunnelAlloc`
**0x9a6ec0**, `CommUDPSend` 0x984e04, `SocketSendto` 0x98aeac, `SocketRecvfrom` 0x98b32c, `ConnApiConnect` 0x98fba4.
Bitstream primitives in 2.2.0: `readIntLimit` 0xc66a4, `readUnsignedLimit` 0xc6744, `readUnsigned` 0xc673c,
`readFloat` 0xc6c60, `readBool` 0xc6cdc, `readUnsigned64` 0xc67dc, `readInt` 0xc6da0, `readString` 0xcc28c.

### Bit encoding (fb::InBitStream)
Big-endian, MSB first (the reader loads 32-bit words and byte-swaps them). `readUnsigned(n)` = n bits.
`read{Int,Unsigned}Limit(lo,hi)` = `bitlen(hi-lo)` bits, then + lo. `readFloat` = raw IEEE f32.
`readBool` = 1 bit. String = u10 length + that many bytes, unaligned. Array = u32 count + elements.

### Packets
`PacketBase` = `PacketType u2` (0 HighFrequency, 1 LowFrequency, 2 MessageList) · `ServerTime f32` ·
`PacketFrequency u8`. Unchanged in 2.2.0 (byte-identical readFrom).
- **MessageList** = PacketBase · u32 n · n × `MessageType u6` · the n bodies back to back (**no lengths**,
  so an unknown body stops parsing). The client sends one message per packet (`sendTryLogin`, `sendPing`, …).
- **HighFrequency** (server → tablet, positions) = PacketBase ·
  `FriendlySquadList[] FriendlyVehicleList[] FriendlyTomahawkList[] SpottedEnemySoldierList[]
  SpottedEnemyVehicleList[] SpottedEnemyTomahawkList[] ObliterationBomb[]`
  - `NetworkState` = `Id u32 · X u13 · Y u13 · Rotation u8` (byte-identical in 2.2.0); Tomahawk lists use it bare
  - `SoldierState` = NetworkState · `InVehicleId u32 · SoldierKit u2 · IsSquadLeader b · IsOnKillStreak b ·
    IsHighValueTarget b · IsFiring b` (1.0.0 kit was 8 bits)
  - `VehicleState` = NetworkState · `IconType u32` · **bool** (added in 2.2.0, name unknown)
  - `FriendlySquad` = `SquadId u6 (0..33) · NormalizedSquadScore f32 · IsSquadWipeProtected b ·
    IsSquadReinforced b · Soldiers[SoldierState]`
  - `BombState` = NetworkState · `TeamId u5`
- **LowFrequency** = PacketBase · `FriendlyDeadSoldierList[SoldierState] · ObjectiveList[Objective] ·
  OrderList[Order] · SquadRequestList[SquadRequest] · MapMarkerList[{MapMarkerId u32, X u13, Y u13}] ·
  Tickets{Friend, FriendMax, Enemy, EnemyMax: u32} · Mutiny{Treshold f32, Duration u32, ActiveSquads u8,
  NegativeRatings u8} · RoundTimeSeconds, SquadPoints, PreRoundPlayerReadyCount, PreRoundRestartTime: u32 ·
  Obliteration Armed/Destroyed bitmasks i32 · CurrentCratesCount u8` · then one extra struct in 2.2.0 (unknown)
  - `Objective` = `ObjectiveName s · LocationTextSid s · ObjectiveId u32 · ObjectiveType u3 · ActionType u5 ·
    ControllingTeamId u5 · CapturingTeamId u5 · X u13 · Y u13 · ProgressVelocity lim(-1000,1000) ·
    ProgressValue u8 · Blink b · IsControlled b`
  - `Order` = `OrderType u2 (None/Attack/Defend) · ReplyType u2 (Pending/Accepted/Denied) · IssuedToPlayerId u32 ·
    ControlPointId u32 · ControllableId u32 · TargetX u13 · TargetY u13`
  - `SquadRequest` = `SquadId u6 · ActionType u5 · ControlPointId u32 · TargetX u13 · TargetY u13`

**Positions → world** (`NetworkParser`; conversion info comes from LoadLevel `WorldCenterX/Y, WorldSize, WorldRotation`):
`p = center + (n / 8191) * size - size/2` per axis, then rotated by `WorldRotation` around center
(`x' = cx + dx·cos − dy·sin`, `y' = cy + dy·cos + dx·sin`). Heading = `Rotation / 255 · 2π + WorldRotation`.
Resolution: WorldSize / 8191 per step (e.g. 2 km map ≈ 0.25 m).

### Messages (MessageType id, 2.2.0; † = not seen in the capture, layout from code only)
0 TryConnect (tablet → server) = `TCClientProtocolVersion u32 · PlayerId u64 · PlayerName s · Platform u2
(0 Unknown, 1 iOS, 2 Android) · 3 more strings (2.2.0, unnamed)` · 1 ConnectionApproved = `TCServerProtocolVersion u32` ·
2 ConnectionDenied = `TCServerProtocolVersion u32 · DeniedReason u3` (None, ProtocolMismatch, CommanderSlotTaken,
ServerFull, RankRequirement, UnknownError) · 3 Ping = `PingTime f32` · 4 Pong = `PingTime f32 · PongTime f32` ·
5 JoinServer† = `ServerName s · ServerVersion s` · 7 LoadLevel = `LevelName s · GameMode lim(-1,3) · CommanderName s ·
CommanderEmblem s · CommanderPlayerId u32 · WorldSize f32 · WorldCenterX f32 · WorldCenterY f32 · WorldRotation f32 ·
RoundTimeLimit u32 · PreRoundPlayerRequiredCount u32 · RoundInPreRoundState b · Max/CurrentPlayerCount u8 ·
Max/CurrentCommanderCount u8 · LevolutionMiniMaps[{Tag s, Size f32, X u13, Y u13, IsPatch b}] · bool (new in 2.2.0)` ·
8 LoadLevelPlayerData = `[Id u32 · Name s · u8 (new) · u64 (new) · TeamId u5 · SquadId u6 · SoldierKit u2 ·
IsSquadLeader b · IsAlive b]` · **16 TriggerAction** (tablet → server, every commander ability) =
`ActionType u5 · PlayerId u32 · MapMarkerId u32 · X u13 · Y u13` · 17 AbortAction / 18 AcknowledgedAction /
19 DeniedAction = `ActionType u5` · 20 PlayerKilled = `KillerId u32 · VictimId u32 · WeaponName s · bool (new)` ·
23 PlayerLeft = `PlayerId u32` · 30 ChangedSelectedSquad = `SelectedSquad u6` · 33 ChatMessage = `ChatMessage s ·
SenderPlayerId u32 · Channel u2` · 35 ClientHasLoadedLevel = `Loaded b` · 29 MutinyCompleted = `b`.
15 ActionEnteredState = ActionState `{ActionType u5 · ActionState u3 (Disabled/Idle/WarmUp/Running/CoolDown/Frozen) ·
TeamId u5 · StateDuration f32 · Radius f32 · TargetX u13 · TargetY u13 · IsExcluded b}` · 22 PlayerJoined = PlayerData ·
24 ScoringEvent = `Description s (string id, e.g. ID_SCORE_COMMANDER_HVT_MARKED) · Score f32 · RawScore f32 · StatEvent u8 ·
VisibilityType u2 · IsEnemyCommander b` · 26 ScoreBoard = `[Id u32 · Score u32 · Kills u16 · Deaths u16 · Rank u16]` ·
9 LoadLevelActionStates = `[ActionState] · InfantryScanDir u2 · VehicleScanDir u2 · EarlyWarningDuration u32` ·
10 LoadLevelLevolutionStates = `[{LevolutionType u3 · LevolutionId s · X u13 · Y u13 · Activated b}]` ·
11 LoadLevelSquadBarData = `[ActionType u5] · PointsPerSegment u32` · 12 LoadLevelRushData† = `u8 · u8 · b` ·
31 CombatAreaUpdated = `[{X u13 · Y u13}]` (combat-area polygon) · 32 EnemyCommanderChanged = `CommanderName s ·
CommanderEmblem s · CommanderPlayerId u32` · 34 CommanderSwitchedTeam = `TeamId u5 · EnemyTeamId u5 · Faction u32 ·
EnemyFaction u32`. The 3 extra TryConnect strings are device model, Android version and connection type
(`Wifi`). The `u8` added to PlayerData is close to the player Id but not equal (meaning open); the `u64` was always 0.
6 LeaveServer, 13 StartMatch, 14 EndMatch have no message class (assumed empty)†. Not decoded yet: 25 RoundOver and
37 Assignment (end of round only); 36 CarrierState† (Titan).
ActionType (2.2.0): 0 SectorScan, 1 SectorJam, 2 Order, 3 SupplyDrop, 4 VehicleSpawn, 5 DefenseDrop, 6 PromoteSquad,
7 LaunchTomahawk, 8 RadarScan, 9 EarlyWarning, 10 InfantryScan, 11 VehicleScan, 12 SatelliteEMP, 13 GunshipDeploy,
14 HighValueTarget, 15 ReinforceSquad, 16 SquadWipeProtection, 17 ProxyAttack, 18 Communication, 19 ParaDrop.
Full enum lists: `python3 scripts/tc_protocol.py enums22`.

### Live capture (2026-11-10, Conquest on MP_Journey, 64 players, ~170 s, ended by round over)
`captures/netcode/netcode_cap.ndjson` (gitignored; tunnel key masked). Decode: `python3 scripts/tc_decode.py
captures/netcode/netcode_cap.ndjson [--hf N] [--udp]`. All 2650 NetGameLink packets decode, and every packet ends with
≤ 7 padding bits, except RoundOver/Assignment. The same session regenerated `captures/blaze/blaze_cap.ndjson` (Blaze, issue 003).

**Transport and ports.** UDP between the phone and the **game server's own address, port 25200** (the normal BF4
server port; here 104.153.105.x). There is no relay and no TCP. ProtoTunnel runs over that one socket. Tunnel info
(`ProtoTunnelInfoT`): remote client id, remote addr, port 25200, port list `[25200, 6000]` (game, DirtySDK VoIP),
port flags `[3, 1]`. The app also sends SSDP M-SEARCH to 239.255.255.250:1900 (DirtySDK UPnP) and 384-byte LAN
broadcasts on :9999 (they carry the string `EAO/Blaze/GameManager`; it hears its own copies back). Neither is needed for play.
Rates: in ≈ 17 datagrams/s, mean 499 B (max 945), ≈ 8.5 kB/s. Out ≈ 5/s, mean 33 B. HighFrequency every 100 ms
of ServerTime, LowFrequency every 500 ms (corrected in 009: ServerTime steps by 0.5 in every capture), Ping once a second (Pong ≈ 70–200 ms later).

**Handshake (observed, seconds from the first datagram).** 3.05 `ProtoTunnelAlloc` (key comes from the Blaze join) → 3.10–3.27
CommUDP connect over the tunnel (24/24/20/20/25/25-byte datagrams) → 3.34 tablet **TryConnect** (protocol 38,
PlayerId = persona id (u64), persona name, Platform 2, `SM-S911B`, `16`, `Wifi`) → 3.65 server EnemyCommanderChanged (empty) +
**ConnectionApproved(38)** + **LoadLevel** (`MP_Journey`, GameMode 0 Conquest, WorldSize 3478.9, center (224.4, 708.3),
WorldRotation π, the commander's own PlayerId 30) → 3.67 tablet **ClientHasLoadedLevel(true)** → server
LoadLevelLevolutionStates, PlayerJoined (self), first HighFrequency (3.69), CommanderSwitchedTeam (team 1 vs 2),
8 × LoadLevelPlayerData (64 players), CombatAreaUpdated, LoadLevelActionStates, LoadLevelSquadBarData (PromoteSquad,
SupplyDrop, ReinforceSquad, VehicleSpawn; 100 pts/segment), first LowFrequency (4.09) → steady state.
Client packets are NetGame kind 5 (reliable) for messages, kind 6 for Ping. Server: kind 6 for High/LowFrequency, 5 for events.

**Tunnel layer and how to undo it.** The wire is encrypted: the `tab.init` tag never shows up in the UDP bytes. In 1.0.0,
`ProtoTunnelAlloc` calls `CryptArc4Init(state, tunnelKey, strlen(tunnelKey), 12)`: RC4 keyed with the
110-character tunnel key string from Blaze, key schedule run 12 times. The packet code calls `CryptArc4Advance(state,
(Δseq & 0x7fff) * 8)` and `CryptArc4Apply`, so the keystream position follows a 15-bit packet counter. No HMAC
or other Crypt* call comes from the tunnel code. Every datagram has `0x80` in byte 1 and a counter-like byte 0.
Decoding from the wire would need the key (from the Blaze join reply) plus the exact framing, which is still open (see open questions).
**Our hooks sit above the tunnel** (NetGameLink), so analysis doesn't need to undo it. An own client (005/006)
must implement ProtoTunnel + CommUDP + NetGameLink framing.

**Positions (a), confirmed.** HighFrequency, 10 Hz: 11 friendly squads with their soldiers, 11 friendly vehicles,
3–4 spotted enemy soldiers, 1 spotted enemy vehicle. World coordinates from the formula fall inside the map bounds.

**Commander actions (b), confirmed.** Every ability the player used produced one tablet → server **TriggerAction** and,
150–360 ms later, a server **AcknowledgedAction{ActionType}** followed by **ActionEnteredState** updates
(e.g. SectorScan → `ActionState 3 Running, StateDuration 30 s, Radius 70, Target = the tapped point`). Field use:
| ActionType | PlayerId | MapMarkerId | X, Y |
|---|---|---|---|
| Order | leader of the selected squad | ControlPointId of the objective | objective position |
| HighValueTarget | targeted enemy's PlayerId | 0xFFFFFFFF | target position |
| PromoteSquad | leader of the selected squad | 0xFFFFFFFF | tap position |
| SectorScan / SectorJam / InfantryScan / VehicleScan | unused (0xFFFFFFFF or stale value) | 0xFFFFFFFF | target |
Selecting a squad sends **ChangedSelectedSquad{SquadId}**. An Order then appears in LowFrequency `OrderList` as
`{OrderType 2 Defend, ReplyType 0 Pending, IssuedToPlayerId, ControlPointId, TargetX/Y}`. Not used this round:
LaunchTomahawk, SupplyDrop, VehicleSpawn, GunshipDeploy, EMP (layout is the same; values not exercised).

### Open questions (next step)
- ~~ProtoTunnel/CommUDP/NetGameLink framing on the wire~~: answered in "ProtoTunnel / CommUDP / NetGameLink on the wire" (issue 008).
- ~~Where the tunnel key and game-server address come from~~: answered in "Blaze session — live capture". Address = `GAME.HNET`
  of `NotifyJoiningPlayerInitiateConnections`. Key = `GAME.UUID-` followed by the sorted pair (`GAME.UUID`, own `PROS.UUID`).
- **Does the server accept a non-app client?** It checks TCClientProtocolVersion (38) and Platform; the persona comes
  from the Blaze session (005).
- Not decoded: RoundOver, Assignment (end-of-round stats), CarrierState (Titan). Meaning of the new 2.2.0 fields
  (VehicleState bool, PlayerData u8/u64, PlayerKilled bool, LoadLevel bool). `ControlPointId` ≠ `ObjectiveId` in
  ObjectiveList: map the two (needed to draw orders on objectives).
- `InVehicleId` holds small values (0/1) that don't look like vehicle Ids; compare with FriendlyVehicleList Ids.
- Map art: positions are in world units, and the minimap textures must be read from the player's OBB (006).

## ProtoTunnel / CommUDP / NetGameLink on the wire (issue 008, 2026-11-11)
Worked out offline from the captures, no re-capture. The tunnel key is rebuilt in memory from the matching Blaze capture
(`GAME.UUID`, own `PROS.UUID`). Three sessions can be decrypted: `netcode_cap.20261110-230540` with `blaze_cap.20261110-232321`
(the same 172 s session, round over and leave, first records 0.6 s apart), `…-232932` with `…-232931`, and `netcode_cap` with
`blaze_cap`. `python3 scripts/prototunnel.py demo` decrypts all 6303 datagrams. On the wire it finds exactly the NetGameLink
payloads the hook logged, in order, both directions. It rebuilds all 1444 tablet datagrams byte for byte, re-encrypted.
Derived values (ids, ident) were checked against the 1.0.0 code: `CommUDPConnect`, `CommUDPSend`, ConnApi's game-connect,
NetGameLink's send. `prototunnel.py` is written from these notes.

**Datagram.** `u16 LE (0x8000 | c)` in clear, then the RC4-encrypted tunnel payload. `c` is a 15-bit block counter.
- **RC4:** keyed with the 110-character tunnel key (ASCII). The key schedule runs **once**, then the first **3072** keystream
  bytes are dropped. `CryptArc4Init(…, 12)` means 12 × 256 discarded bytes, not 12 schedule passes.
- **Keystream position:** payload byte *i* of a datagram uses keystream byte `8c + i`. The sender adds `ceil(len/8)` to `c` after
  each datagram. Each direction keeps its own `c`, starting at 0. Both directions are keyed the same, so they run the same
  keystream; that is the app's design. Inbound `c` wraps about every 30 s at 8.5 kB/s, so the receiver unwraps it.

**Tunnel payload.** n × `u16 BE (len << 4 | port index)`, then the n bodies in header order. The receiver reads headers until
their lengths add up to the remaining bytes. Port index follows the `ProtoTunnelInfoT` port list:
- 0: the game, :25200, CommUDP
- 1: VoIP, :6000 (DirtySDK voice). Seen only when someone used push-to-talk. Not decoded, not needed.
- 7: tunnel control

**Tunnel control (port 7, 6 bytes):** `00 · u32 BE tunnel client id · state` (0 = hello, 1 = heard the peer). It appears only in
each side's first two datagrams, next to CommUDP INIT/CONN.
- Client id = **NetHash(persona name)** (ConnApi `'clid'`). Ours is NetHash of our Blaze persona name. The server's is
  NetHash(`"bf4-pc"`) = `0xb3ae3e68`, the server's own Blaze user.
- It was the same in every session. It is how the server finds our tunnel without knowing our NAT address
  (the app never learns its external IP).

**NetHash** (DirtySDK): `h = Σ byte_i · W[i mod 16]`, with `h *= 0x1a042e5b` at every 16-byte boundary, mod 2³². W is 16
constants, listed in `prototunnel.py`.

**CommUDP (port 0):** `u32 BE seq · u32 BE ack · body`.
- **Connect, ~130 ms:**
  1. tablet → `00000001 · connident · own client id` (INIT, with control state 0)
  2. server → `00000005 · connident · server client id` (with control 0)
  3. tablet → `00000002 · connident` (CONN, with control 1)
  4. server → the same CONN (with control 1)
- **Disconnect:** tablet → `00000003 · connident` (DISC), sent once, no reply. The 172 s session ends with it, 2.4 s before Blaze
  `leaveGameByGroup`.
- **connident** = NetHash of ConnApi's game name `"%x-%x"`: the topology host's MACI, then our MACI, in hex. That is the server's
  `GAME.HNET[0].MACI` and our `PNET.VALU.MACI`. The app sends its LAN IP as MACI, so the ident changes with the phone's IP:
  `5f0b…` twice, `5fe9…` in the third session. Code: CommUDP hashes the text after `#` in ConnApi's `"%a:%d:%d#%s"` address.
- **Reliable:** 24-bit seq `0x100…0xffffff`, wrapping to `0x100`, per direction. The first reliable packet is `0x100`.
  `ack` = last in-order reliable seq received (`0xffffff` = none yet).
- **Unreliable:** seq = `0x80 + n mod 128`.
- **Pure ack:** empty body, with seq = the *next* reliable seq (not consumed).
- **Bundles:** the top nibble of the seq field counts older, still-unacked reliable packets. They are appended after the newest
  one, newest first, each followed by its length byte. Bits 24–27 are a metadata type, always 0 here. The server does this
  often. The tablet did it 5 times in 170 s (a new TriggerAction plus an unacked sync-only packet). GameLink resends instead.
- **Timers:** an unacked packet is resent after ~100–115 ms. The tablet acks reliable data within ≤ 100 ms, with a pure ack
  or the next packet.

**NetGameLink:** `payload · [sync, 12 bytes] · kind`.
- **kind:** 5 = reliable, 6 = unreliable (7 = unreliable broadcast, not seen). `| 0x40` = sync present.
- **0x40 alone** = a sync-only packet, sent **reliable**. These are what use up reliable seqs while idle: `0x100` right after
  CONN, then about once a second.
- **sync** = `u32 echo · u32 own ms tick · u16 late · u8 sent · u8 12`.
  - echo = the peer's last tick plus the ms since we got it.
  - late = the rounded mean of two internal values, probably latency estimates, in ms. The tablet sent 0x21–0x61; the server sends 1.
  - sent = datagrams sent since the previous sync.
  - It is attached at most every 250 ms: TryConnect went without it, ClientHasLoadedLevel and Ping carried it.
- **payload** = tag `tab.init` + one TabletCommander bitstream packet. The **outgoing tag is also `tab.init`** (`this+2156`).
  The app leaves stale bits in a packet's final padding byte; ours are zero.

**Shape of a session on the wire** (seconds from the first datagram):
1. 0.00 INIT ↔ 0.05–0.11 INIT reply, then CONN ↔ 0.13 CONN
2. 0.18–0.27: both sides send sync-only `0x100`
3. 0.25–0.33: TryConnect (`0x101`)
4. server: EnemyCommanderChanged, ConnectionApproved, LoadLevel (`0x101`–`0x103`, partly bundled)
5. ClientHasLoadedLevel (`0x102`), then the load burst
6. Steady state:
   - server: ~17 datagrams/s; High/LowFrequency unreliable, events reliable
   - tablet: Ping once a second (unreliable, with sync), sync-only once a second (reliable), pure acks

**Ports.** The server is at `HNET`:25200. The hook doesn't log the tablet's source port; `updateNetworkInfo` announces 3659.
GameLink binds 3659 and falls back to any free port. The server answers whatever address it hears from (NAT).

**Blaze around the link.**
- Right after the CommUDP CONN: `updateMeshConnection{FLGS 0, GID, SCG = own CGID, STAT 2, TCG = (30722, 2, GAME.THST.CONG)}`.
- At leave, after `leaveGameByGroup`: the same with `STAT 0`.
- `blaze_client.py demo` rebuilds both byte-identical.
- **The app never sends `LeaveServer`** (TC message 6): 1.0.0 has no sender, and `TabletCommanderNetworkMain::disconnect` is
  empty. The app leaves with CommUDP DISC, then `leaveGameByGroup`, then `updateMeshConnection{STAT 0}`.

**TryConnect's 3 extra strings.** Ours sends what the player's own app sent: `SM-S911B`, `16`, `Wifi`
(`blaze_client.CLIENT_STRINGS`). The one live run should change a single thing, who speaks the protocol. Whether other values
pass can be tested later.

**Live run (player-approved):** command `blaze_client.py --link --hold 45`.
- Attempt 1 (2026-11-11 15:04, `captures/blaze-client/run-20261111-150420.log`) stopped before login. The redirector TLS handshake
  timed out. Cause: the work VPN was up (full tunnel, `utun4`), and through it EA's redirector never answers the
  TLS 1.1 hello (TCP connects; `--probe` fails the same way). Without the VPN (005 runs, 00:08 probe) it works. Nothing reached
  the game server, and the auth code was not sent. **Run with the VPN off**; the game UDP must not go through it either.
- **Attempt 2 (2026-11-11 15:23, `captures/blaze-client/run-20261111-152317.log`, VPN off): the game server accepts our own client.**
  Server: Metro-only Conquest, 34/64 players, 0 commanders before us. Seconds from the click:
  - 3.40 join notification. Link opened from local port 3659.
  - 3.46 CommUDP connected (60 ms), then `updateMeshConnection{STAT 2}`.
  - 3.56 `NotifyGamePlayerStateChange{STAT 4}`: our roster entry is active. TryConnect sent.
  - 3.70 **`ConnectionApproved(38)`**, then **`LoadLevel`** (MP_Metro, GameMode 0, WorldSize 1018.8, CommanderPlayerId 16).
    `ClientHasLoadedLevel` sent.
  - Then the full load: LoadLevelLevolutionStates, PlayerJoined, CommanderSwitchedTeam (team 1 vs 2), 5 × LoadLevelPlayerData,
    CombatAreaUpdated, LoadLevelActionStates, LoadLevelSquadBarData.
  - Steady state for 45 s: 220 HighFrequency, 44 LowFrequency, 44 Pong for our 1/s Pings, 21 PlayerKilled, 8 ScoreBoard.
    Every packet decoded, and the link logged no error.
  - 48.41 DISC, then `leaveGameByGroup` → `NotifyPlayerRemoved{PID own, REAS 6}` (player left, not the 30 s timeout), then
    `updateMeshConnection{STAT 0}`, then close.

  Nothing in the handshake checks for the app beyond what we send: protocol 38, Platform 2 and the three device strings.
  We sent the app's own values for these, so whether other values pass is still open.

## Blaze session — live capture (issue 003, 2026-11-10)
One session: Battlelog Commander click → join a 64-player Conquest server → round end. Decoder: `scripts/blaze_decode.py`
(`--brief` = one line per frame). It masks token-like strings and blobs to 4 chars, so its output is safe to paste.
RPC, notification and error names come from `scripts/blaze_names.json`, which `scripts/blaze_names.py` extracts by running
each component's `getCommandName`/`getNotificationName`/`getErrorName` from the 1.0.0 lib in unicorn (installed in the
gitignored venv). Component ids come from the error codes, which are `(code << 16) | componentId`.
323 frames, no misframes. TDF parsing has one gap: 2 bytes after `RRST` in `joinGame`, reported as `?skipped`.

### Framing
- **Fire header, 12 bytes, big-endian:** `u16 size · u16 component · u16 command · u16 error · u16 (msgType << 12 | flags) ·
  u16 msgNum`. msgType: 0 request, 1 reply, 2 notification, 3 error reply. If flag `0x0010` is set, a u16 size-high follows.
  Replies echo the request's msgNum. Notifications use msgNum 0.
- **Heat1 TDF:** each field is a 3-byte tag (4 chars × 6 bits, +0x20) and a type byte. Types:
  - 0 int (varint: bit 7 = more, bit 6 = sign in the first byte)
  - 1 string (varint length incl. NUL)
  - 2 blob
  - 3 struct (ends with 0x00)
  - 4 list (element type, count)
  - 5 map (key type, value type, count)
  - 6 union (member byte, 0x7f = unset, then one field)
  - 7 int list
  - 8 pair
  - 9 triple (BlazeObjectId)
  - 0xA float
- **Connections:** ProtoSSL calls from every TLS connection land in one log. Telemetry (a separate connection, payload `&lgr…`)
  is told apart by its first bytes. The hook now tags each record with the ProtoSSL ref (`ssl`), and the decoder frames
  per connection when that tag is present.

### Sequence (seconds; → = tablet to server)
**Redirector** (ProtoSSL to the redirector):
- 0.00 → `Redirector.getServerInstance`: `NAME battlefield-4-pc · ENV prod · CLNT "Battlefield 4 Commander" · CLTP 4 ·
  CPLT 5 · CSKU/PLAT Android · BSDK/DSDK 13.3.1.2.1 · BTIM "Jun 2 2014" · CVER "TabletCommanderProtocol_Version" ·
  PROF standardSecure_v3 · LOC enUS`
- 0.13 ← reply: `ADDR{HOST diceprodblapp-08.ea.com, IP 159.153.81.52, PORT 10237} · SECU 1 · CERT[10 PEM CA certificates] ·
  XDNS 0`. The redirector hands out the Blaze server **and the CA certificates for it**.

**Blaze server** diceprodblapp-08.ea.com:10237 (TLS). It reports `Blaze 13.3.1.8.0 (CL# 2015647)`, instance
`battlefield-4-pc`, platform `pc`.
- 0.47 → `Util.preAuth` `{CDAT{SVCN battlefield-4-pc, TYPE 4, LANG}, CINF{client info as above}, FCCR{CFID BlazeSDK}}`
  ← server version, config (`pingPeriod 30s`, `connIdleTimeout 90s`, nucleus URLs), QoS servers per region
  (`qos-prod-*.gos.ea.com:17502`), component id list.
- 0.53 → `Util.ping` ← `STIM` (server time).
- 0.58 → **`Authentication2.login`** `{AUTH: the Battlelog authenticationCode (42 chars on the wire), EXTB empty, EXTI 0}`. That is the whole login.
  - 2.14 ← `UserSessions.UserAuthenticated` (persona name `DSNM`, persona id `PID`, nucleus `UID`, session `KEY` (59 chars),
    `PLAT 4`, email masked by the server)
  - 2.15 ← login reply `SESS{KEY, PDTL{DSNM, PID, PLAT 4}, UID}`. The login takes about 1.6 s.
- 2.15 → `Util.postAuth` ← ticker `{ADRS <private IP>, PORT 8999, SKEY "<pid>,addr,service,…"}`; telemetry and PSS empty.
- 2.20 → `UserSessions.updateNetworkInfo` `{ADDR union[2] VALU{INIP <phone LAN ip>:3659, EXIP 0}, NQOS{NATT 5}}`.
  The app doesn't learn its external address.
- 2.20 → `Util.getTelemetryServer`. Telemetry then goes over its own TLS connection from 2.51.
- 2.20 → **`GameManager.joinGame`** `{GID <Battlelog gameId> · GVER "4900" · JMET 1 · GENT 2 · BTPL <own connection group> ·
  PNET (as updateNetworkInfo) · RRST {"commander": {}} (role) · SLOT 0 · TIDX 65535 (any team) · USER{0} · XSES empty}`
  ← reply `{GID, JGS 0}`
- 2.35 ← `UserSessions.UserAdded` + `UserUpdated` for **every player in the server** plus the server's own user
  `bf4-pc`. Each carries persona, region and **external/internal IP**. Privacy: those IPs stay in the gitignored captures, never in notes.
- 2.46 ← **`GameManager.NotifyJoiningPlayerInitiateConnections`** (20.7 kB) — the game-server connection info:
  - `GAME.HNET = [{EXIP, INIP} = <game server IP>:25200]`: where the tablet connects (issue 004)
  - `GAME.UUID`: game UUID. Its prefix matches the first part of the ProtoTunnel key.
  - `GAME.ATTR`: server attributes (name, level `MP_Journey`, `ConquestLarge`, punkbuster/fairfight, settings strings)
  - `GAME.RNFO.CRIT = {commander: {CRIT {commanderRank: "stats_rank >= 10"}, RCAP 2}, soldier: {RCAP 64}}`
    (the commander rank gate and 2 commander slots)
  - `VSTR "4900"`, `NTOP 1`, `VOIP 0`; `THST/PHST`: topology/platform host = the server user `bf4-pc`
  - `PROS[65]`: roster (64 soldiers plus our own entry `ROLE commander` with our `UUID`, `SLOT 0`, `TIDX`, `STAT 2`)
- 2.47 ProtoTunnelAlloc → 2.51 first UDP to :25200 (netcode capture on the same clock).
- 2.65 → `GameManager.updateMeshConnection` `{GID, SCG own group, TCG host group (THST.CONG), STAT 2 connected}` ← empty reply.
- 2.75 ← `NotifyGamePlayerStateChange{PID own, STAT 4 active}` and `NotifyPlayerJoinCompleted`. At the same moment the
  tablet sends TryConnect to the game server (004).
- Steady state: `Util.ping` every 30 s. GameManager notifications as players join, claim reservations or leave
  (`NotifyPlayerJoining/ClaimingReservation/Removed`, `UserAdded/Removed`). At round end (157 s):
  `NotifyGameStateChange` ×3, `NotifyGameReportingIdChange`, then a burst of `UserSessionExtendedDataUpdate`.

### ProtoTunnel key = three UUIDs
ConnApi's tunnel setup in 1.0.0 (just before `ProtoTunnelAlloc` at 0xe37dc8) runs
`ds_snzprintf(key, 128, "%s-%s-%s", <string at ConnApi+184>, uuidA, uuidB)`. uuidA and uuidB are the local and remote client
UUIDs, swapped into strcmp order. Live (110 chars = 3 × 36 + 2; the hook logs only a 4-char prefix of each part), two sessions:
- own UUID `bc94…` → key `5b32…-5b32…-bc94…`
- own UUID `24ed…` → key `5b32…-24ed…-5b32…`

So **key = `GAME.UUID + "-" + join("-", sorted([GAME.UUID, own PROS.UUID]))`**. The dedicated server's client UUID is the game
UUID, and our UUID is the `PROS` entry with `ROLE commander` in `NotifyJoiningPlayerInitiateConnections`. `GAME.UUID` stayed the
same across sessions on the same server; our UUID changes on every join.

### TLS (ProtoSSL) — for issue 005
- **Offered:** ClientHello `client_version` **3.2 (TLS 1.1)**. Cipher suites come from an enable bitmask:
  `0x0005 RSA_RC4_128_SHA`, `0x0004 RSA_RC4_128_MD5`, `0x002F RSA_AES_128_CBC_SHA`, `0x0035 RSA_AES_256_CBC_SHA`. RSA key exchange only,
  no (EC)DHE, null compression. The code has a TLS-1.1 explicit-IV branch (`version > 0x301`). Found in 1.0.0 at 0xe4b0e0/0xe4b254
  and byte-identical in 2.2.0 (0x9a1b24/0x9a1c98). This is why a modern `curl`/OpenSSL gets no answer.
- **Certificates:** the redirector reply ships the CA certificates that the Blaze connection is then checked against
  (Redirector also has `getCACertificates`/`findCACertificates`). The redirector's own certificate presumably
  chains to ProtoSSL's built-in CAs (`ProtoSSLSetCACert`, `ProtoSSLValidateAllCA`).
- **Negotiated (live, `k='tls'` handshake records, `tc_decode.py` prints them):**
  - redirector: **TLS 1.1 + `RSA_AES_256_CBC_SHA`**
  - Blaze server (diceprodblapp): **TLS 1.1 + `RSA_RC4_128_SHA`**

  Full handshake both times: ClientHello → ServerHello, Certificate, ServerHelloDone → ClientKeyExchange, ChangeCipherSpec,
  Finished. No client certificate, no session resumption. ProtoSSL reads each record's 5-byte header on its own, so the
  hook joins it with the next read.

### What an own client must send (input for 005)
redirector `getServerInstance` (as the tablet: `CLNT`, `CSKU`/`PLAT Android`, `CVER`, `NAME battlefield-4-pc`, `ENV prod`) →
`preAuth` → `Authentication2.login{AUTH}` → `postAuth` → `updateNetworkInfo` → `joinGame{GID, GVER 4900, RRST commander, …}`
→ wait for `NotifyJoiningPlayerInitiateConnections` (server address, UUIDs) → ProtoTunnel/CommUDP to `HNET:25200` →
`updateMeshConnection{STAT 2}` → TabletCommander `TryConnect` (004). All over TLS ≤ 1.1 with RSA ciphers, to a server that
takes the `AUTH` code from Battlelog.

## Own Blaze client (issue 005, 2026-11-11): EA accepts a non-app client
Scripts: `scripts/blaze_client.py` (Blaze client), `scripts/tls11.py` (ProtoSSL-compatible TLS 1.1), `commander_bridge.py --save`.
Both have offline self-checks (`… demo`). The client's requests are **byte-identical to the app's**: `blaze_client.py demo`
rebuilds every captured getServerInstance, preAuth, login, postAuth, updateNetworkInfo, joinGame and leaveGameByGroup from
the 003 captures and compares them byte for byte.

Run it (player approval required, live service):
1. `python3 scripts/commander_bridge.py --save`: no adb. The bridge writes the params to `captures/launch.json` (mode 0600, atomic replace).
   Stop `mirror.py` first, because it holds the same port.
2. `python3 scripts/blaze_client.py --hold 20`: waits for that file, reads it and deletes it, then runs the session.
3. Click Commander in Battlelog **once**. A double click mints two auth codes, and the second one is wasted.
`--probe` does the redirector lookup and the Blaze TLS handshake (both verified) without logging in. No click needed.
It saves the server certificates to `captures/blaze-client/<host>.pem`.

### Result (run 3, `captures/blaze-client/run-20261111-001043.log`)
redirector (verified, `OTG3`) → Blaze server `diceprodblapp-06.ea.com:10057` (verified, `GOS 2013`) → preAuth → ping →
**`Authentication2.login` OK (1.5 s)** → postAuth → updateNetworkInfo → **`joinGame` accepted (`JGS 0`)** →
`NotifyJoiningPlayerInitiateConnections`, where our roster entry has `ROLE commander`, `STAT 2` (connecting) and `SLOT 0`
(commanders in the roster: 1 of `RCAP 2`). No game-server link was opened. **Exactly 30.0 s after the join notification**, the server sent
`NotifyPlayerRemoved{PID own, REAS 0}`.
- **Run 4 (`run-20261111-001447.log`, hold 15 s): clean leave.** Same flow, then `leaveGameByGroup{REAS 7}` at +15 s → empty reply →
  `NotifyPlayerRemoved{PID own, REAS 6}`. Then the connection closes (TLS close_notify).
- `REAS` values seen: 0 = join timeout (us, no game connection), 6 = player left (others, and our own leave in run 4),
  7 = group left (what the leave request sends).
- **Battlelog's server view kept showing the commander slot as free** during run 4 (player watched it). Our entry stays at `STAT 2`
  (connecting) and never becomes active (`STAT 4`, which the app reaches after the game-server link). So Battlelog counts only
  players who reached the game server, or it lags. The Blaze roster (`ROLE commander`, 1 of 2) is the evidence that the join was accepted.
  These match Blaze's PlayerRemovedReason order (PLAYER_JOIN_TIMEOUT, …, PLAYER_LEFT, GROUP_LEFT).
- **Join timeout = 30 s.** A real client must bring up ProtoTunnel, send TryConnect (004) and `updateMeshConnection{STAT 2}`
  within 30 s of `joinGame`.
- The answer to 006's open question: **the Blaze backend accepts our own client.** There is no client attestation: the auth code
  plus app-shaped requests are enough. Whether the game server accepts a non-app client is still open (TryConnect, 004).
- `BTPL` in joinGame/leaveGameByGroup is the session's connection group, `CGID` from `UserSessions.UserAuthenticated`.
- Each run gets a different Blaze server (seen: diceprodblapp-06:10057, -07, -08:10237, -10:10221). Always ask the redirector.

### The TLS findings (runs 1–2 and the probe)
- **Run 1, Python `ssl` (OpenSSL 3.6, TLS 1.1, AES suites, `@SECLEVEL=0`):** TCP connects, but the redirector never answers the
  ClientHello (15 s timeout). The app's hello is bare: record version **3.0**, client_version 3.2, empty session id,
  suites `0005 0004 002f 0035`, null compression, **no extensions**. That's 56 bytes (`captures/netcode/netcode_cap.ndjson`, `k=tls`).
  OpenSSL adds SNI, session_ticket, extended_master_secret, encrypt_then_mac and renegotiation_info. Two possible causes, not
  separated: EA's server stalls on extensions, or the player's endpoint filter (the work VPN's web filter, which reads SNI)
  holds the hello. The bare hello gets through either way.
- Python `ssl` can't produce the bare hello. Options `OP_NO_TICKET`, `1` (NO_EXTENDED_MASTER_SECRET) and `1<<19`
  (NO_ENCRYPT_THEN_MAC), plus no SNI, leave renegotiation_info, which can't be dropped. Homebrew OpenSSL 3.6 also has **no RC4 TLS suites**,
  even with the legacy provider loaded (LibreSSL 3.3.6 in `/usr/bin/openssl` still has them). tlslite-ng 0.8.2 still sends
  session_ticket and cert_type. Hence `tls11.py`: about 250 lines of stdlib code (RSA key exchange, TLS 1.0/1.1 PRF, RC4 in Python,
  AES-CBC through the `openssl enc` CLI because the stdlib has no AES). It's tested against local `s_server` for all 4 suites.
- Negotiated with our hello: redirector **AES-256-CBC-SHA**, Blaze server **RC4-128-SHA**, the same as the app.
- **Certificates:** the servers send only the leaf. The redirector's leaf is issued by **`OTG3 Certificate Authority`** and the Blaze servers'
  (`O=Electronic Arts, Inc., OU=Global Online Studio, CN=diceprodblappNN.ea.com`) by **`GOS 2013 Certificate Authority`**.
  Neither CA is in the redirector's `CERT` list (10 public roots: DigiCert, GeoTrust, Entrust, …). Run 2 checked against
  that list, so it stopped with "unable to get local issuer certificate" before the key exchange, without sending the auth code.
- **ProtoSSL's built-in CAs** (2.2.0 lib): a table at file offset **0xc579f8**, 3 entries of 0x1f0 bytes. Each entry holds inline subject
  strings (C, ST, L, O: 32 bytes each; OU: 256; CN: 64), then `u32 key type (1) · u32 modulus size · u32 modulus pointer
  (file offset = address, first LOAD segment) · u32 exponent size · exponent bytes`.
  OTG3 = 1024-bit, e = 3. GOS 2011 = 2048-bit, e = 17. GOS 2013 = 2048-bit, e = 65537. OU strings carry `/emailAddress=GOSDirtysockSupport@ea.com`.
  `blaze_client.py` reads these keys from the player's own APK at runtime, and `tls11.check_cert` checks a leaf against them:
  an exact PKCS#1 v1.5 block compare (OTG3's e = 3 makes lenient parsing forgeable), validity dates, and the hostname (SAN or CN).
  Both hops verify live. The probe also showed that the work VPN does not re-sign these connections.
- Redirector: `gosredirector.ea.com:42127` (prod host string, and port 42127 as `movw` immediates around 0x95c600 in the lib).

## What a browser client needs (issue 005 → 006)
**Transport.** Per session: one short TCP+TLS connection to the redirector, then one long-lived TCP+TLS connection to the assigned
Blaze server (`Util.ping` every 30 s, `connIdleTimeout` 90 s), then UDP to the game server's `HNET` address on :25200
(ProtoTunnel, issue 004). Pages can't open raw TCP or UDP, so a local companion has to own these sockets (006).

**TLS.** TLS 1.1, RSA key exchange, RC4-128-SHA (Blaze) or AES-256-CBC-SHA (redirector), sent with the app's extension-free hello.
The servers authenticate with EA's built-in CAs (OTG3, GOS 2013), whose public keys come from the player's own APK. Hostname checked.
No browser, Node `tls` or OpenSSL-3 stack can do this, so the companion carries its own TLS 1.1 (`tls11.py` is the reference).
RC4 is the Blaze-side cipher. The connection is only as strong as RC4, the same as the official app.

**RPCs, in order** (builders in `blaze_client.py`; field values as captured in 003):
`Redirector.getServerInstance` → [Blaze] `Util.preAuth` → `Util.ping` → `Authentication2.login{AUTH}` (wait for
`UserAuthenticated` → `CGID`, login reply → `PID`) → `Util.postAuth` → `UserSessions.updateNetworkInfo` →
`GameManager.joinGame{GID, GVER, BTPL = CGID, RRST commander, …}` → `NotifyJoiningPlayerInitiateConnections` (game-server address,
UUIDs for the tunnel key) → **within 30 s:** game-server link (004) + `GameManager.updateMeshConnection{STAT 2}` → session
(`Util.ping` / 30 s) → leave: `GameManager.leaveGameByGroup{REAS 7}` + `updateMeshConnection{STAT 0}`.
Timing: the auth code is single-use and short-lived, so the companion starts the moment the Commander click arrives.

**Where the WebSocket line goes.** Recommended: **the companion terminates everything** (TLS, Blaze, ProtoTunnel/netcode).
The localhost WebSocket then carries only game-level JSON:
- page → companion: `start` (session status request), commander actions (`TriggerAction{ActionType, PlayerId, MapMarkerId, X, Y}`,
  `ChangedSelectedSquad`, `AbortAction`, chat) and `leave`
- companion → page: session status (`redirecting`, `logged-in`, `joined`, `in-game`, `removed{reason}`, `error{rpc, code}`),
  LoadLevel/world info, High/LowFrequency state and event messages (004 field lists)

The launch params, including the auth code, go from the extension straight to the companion (`POST /launch`, the existing bridge
interface). They never cross the page's WebSocket.
The alternative is a dumb relay, with WebSocket frames `{conn, open host:port tcp|udp | data bytes | close}` and TLS 1.1, Blaze and netcode
written in page JS. It would put the auth code and session keys in the page and reimplement RSA/RC4/TLS in JS, for no gain. Rejected.

## Map art in the OBB (issue 006 → 011, 2026-11-11)
Looked at only; nothing extracted into the repo.
- `main.11.com.ea.game.warsawcommander_row.obb` is a plain **stored zip** (57 files, no compression) under `Data/`: `layout.toc`,
  `initfs_Android`, `bom.fb2`, and superbundle pairs `Data/Android/<name>.{toc,sb}`. `layout.toc` also names iOS superbundles, which aren't in the zip.
- Superbundles: `TCMapsSuperBundle` (12.9 MB sb, the map art), `Chunks0` (936 MB sb, probably the bulk chunk store), `UI`,
  `TabletCommanderMainLevel`, `Globals`, `defaultLevel`, eight `Background_*_Bundle`s, audio and per-language VO.
- `TCMapsSuperBundle.toc` is an **unencrypted Frostbite DbObject** (starts `82 f4 12 0f`, then `tag`, `bundles[{id, offset, size}]`,
  `chunks`, `totalSize`, `name Android/TCMapsSuperBundle`, `alwaysEmitSuperbundle`). It lists **32 per-map bundles**, e.g.
  `Android/levels/tcmap_journey_bundle` (MP_Journey), `Android/xp0/tcmap_xp0_002_oman_bundle`, `Android/xp4/tc_map_xp4_titan_bundle`.
- `TCMapsSuperBundle.sb` at offset 0 starts `00 00 b2 2c 9d 79 8e d5 …`: a u32 BE size (0xb22c) and then what looks like a bundle
  magic. The bundle format is not decoded yet (011).

## Companion (issue 009, 2026-11-11)
`scripts/companion.py` replaces `commander_bridge.py` on the browser path. It implements 006 → "Interfaces" on
`127.0.0.1:47615`: `POST /launch` (the bridge's checks, `bridge.check`), `GET /` (static files in `scripts/game/`), `GET /ws`
(JSON text frames). Python stdlib only. `mirror.py` and `commander_bridge.py` bind the same port, so run one of them at a time.

Use:
- **Replay (offline, no EA traffic):** `python3 scripts/companion.py --replay captures/netcode/netcode_cap.ndjson`, then open
  http://127.0.0.1:47615/. A replay starts when a page connects and none is running, so reloading after the end replays again. The
  captured server packets go out at their original timing. Page actions are checked, encoded and logged with their wire bytes, but
  not sent (`page: TriggerAction {...} -> 80000000… (not sent: no game link)`).
- **Live (player approval, EA's servers, VPN off):** `python3 scripts/companion.py`, then click Commander in Battlelog. The shim POSTs the
  params, and the companion opens the game page in the default browser unless a page is already connected. The log goes to `captures/companion/run-<time>.log`.
- **Self-check:** `python3 scripts/companion.py demo`. It covers the action range checks, the `/launch` and `/ws` Origin checks, a synthetic
  replay through a real WebSocket, late join, the page action → wire bytes path, and the live engine's error path without network.

How it works:
- **Live engine.** `blaze_client.Session` runs `join()` → `link(on_packet)` → `poll()` → `leave()`. `blaze_client.run` (the CLI) uses
  the same steps. `poll()` sends `Util.ping` every 30 s, as the app does. `GameLink` takes `on_packet` (every decoded server packet,
  called on the link thread) and `post(msg)` (a queue drained by the link thread once approved, so other threads never touch the
  RC4/CommUDP state).
- **Status.** Join steps report `blaze` (detail: `getServerInstance`, `login`, `joinGame`), then `connecting` (game link). A
  `ConnectionApproved` in the stream becomes `in-game`, in replay too. The session ends on page `leave`, `NotifyPlayerRemoved`
  (`removed` + `REAS`), `ConnectionDenied` (`denied` + `DeniedReason`), a link failure or exception (`error` + `rpc` + `code`), or
  60 s without a connected page (`left`). It leaves with DISC → `leaveGameByGroup` → `updateMeshConnection{STAT 0}`, as the app does.
  It never sends `LeaveServer` (006 updated).
- **Codec.** `tc_decode.decode()` returns enums as integers. Only `main()` names ActionType for printing. `tc_decode.message()`
  is the encoder. Page actions are checked against the 2.2.0 layout (ActionType 0..19, X/Y 0..8191, ids u32, SelectedSquad 0..33,
  Channel 0..3, strings ≤ 1023 UTF-8 bytes, exact field set; JSON booleans are not integers). Anything else is logged and dropped.
  `ChatMessage.SenderPlayerId` = `LoadLevel.CommanderPlayerId`; chat before `LoadLevel` is dropped.
- **Rates (replay of `netcode_cap.ndjson`, measured at the page):** HighFrequency 10.0/s, **LowFrequency 2.0/s**. The server sends
  LowFrequency every 500 ms of ServerTime (both captures). The earlier "1 s" in this file was wrong.
- **Live run (player-approved, 2026-11-11 15:44, `captures/companion/run-20261111-154431.log`, VPN off):** the companion plays live.
  The server was Conquest on MP_Siege, 42 players. A scratch driver connected to `/ws` as a page. Seconds from the Commander click:
  - 0.67 Blaze login; 2.63 `joinGame`; 2.80 in the roster (`ROLE commander`, 1 of 2).
  - 2.86 CommUDP connected. 2.96 roster `STAT 4`. 3.05 **`status in-game`** (ConnectionApproved + LoadLevel).
  - The page's **SectorScan `TriggerAction`** (centre of the combat area) got **`AcknowledgedAction` 94 ms later**. Before that,
    `ActionEnteredState{SectorScan, team 1, Running, 30 s}` arrived.
  - Page `leave` → DISC → `leaveGameByGroup` ok → `NotifyPlayerRemoved{own PID, REAS 6}` → `updateMeshConnection{STAT 0}` →
    `status ended left`. 106 HighFrequency and 21 LowFrequency messages reached the page in ~10 s.
  - **Game-server port:** this server's `HNET` port was **25300**, not 25200. The port comes from `HNET`, and `GameLink` already
    takes it from there.
  - **Game tab:** the shim's tab never connected; only the driver did (the extension hadn't been reloaded). A replay-mode retest with the
    reloaded shim still opened no tab from the Commander click, although the click's POST arrived each time. The cause wasn't pinned
    down (popup blocker, or Chrome refusing the jump to localhost). **Fix:** the companion opens the page itself (`webbrowser.open`)
    when a launch arrives and no page is connected. Verified with a real click in replay mode: the tab opened and loaded 1.1 s after
    the POST. The shim's `window.open` and the bridge's `GET /` redirect are gone.
- **Masking fix (found in this run):** the session `KEY` can contain `$`. `blaze_decode.TOKEN` didn't allow `$`, so the 59-char key was
  written unmasked into this log and into `captures/blaze-client/run-20261111-001447.log` (gitignored, local only; the sessions
  are closed). `TOKEN` now allows `$~!*`, both logs were re-masked in place, and `blaze_client.py demo` checks for this.

## Loader phase — binary inventory (2026-11-11)
Input for `notes/loader-spec.md` and issues 012–021. All from static inspection of the 2.2.0 library
(`apk/mod/unzipped/lib/armeabi/libFirstContact.Main_android.so`, 12,993,792 bytes) with the NDK's `llvm-readelf`/`llvm-nm`.

### ELF layout
- `ET_DYN`, ARM, link base 0, soname `libFirstContact.Main_android_retail`. Flags `SYMBOLIC BIND_NOW`.
- Two `PT_LOAD`: R+X `0x0`–`0xbcfee8` (file = memory size) and RW at `0xbd1418`, file size `0x93b69`, memory size `0x15f3ac`
  (the rest is zero-filled). `GNU_RELRO` covers the first `0x72be8` bytes of the RW segment.
- `PT_ARM_EXIDX` at `0xbcfdf8`, `0xe8` bytes (C++ unwinding). No `PT_TLS`.
- Relocations are `REL` (addend in place), three types only: 90,214 `R_ARM_RELATIVE`, 427 `R_ARM_JUMP_SLOT`, 60 `R_ARM_GLOB_DAT`.
- `INIT_ARRAY` at `0xbea5d0`, 444 bytes = 111 constructors. `FINI_ARRAY` one entry.
- Entry points: `JNI_OnLoad` @ `0x4e44b0`, `ANativeActivity_onCreate` @ `0xbdd3c` (both even addresses = ARM mode).

### Imports
- Needed libraries: `libEGL`, `libGLESv1_CM`, `libGLESv2`, `libOpenSLES`, `libandroid`, `libc`, `liblog`, `libm`, `libstdc++`.
- 478 undefined dynamic symbols: 177 `gl*`, 14 `egl*`, 37 `pthread*`, 7 OpenSL (`SL_IID_*` data), 6 `AInput*`, 3 `ALooper*`,
  2 `__android_log*`, the rest libc/libm (string, stdio, sockets, time, math).
- List them with `llvm-nm -D --undefined-only <lib>`.

### Java surface
- The app is a `NativeActivity` (`com.frostbite.firstcontact.GameActivity`); the library owns the main loop.
- JNI functions the library exports for Java to call: `JavaToNativeBridge.{loadPreferences, savePreferences, saveBlazeParameters,
  newBlazeParametersAvailable, onJavaScriptEvent}`, `EAIO.{StartupNativeImpl, Shutdown}`, `EAThread.Init`,
  `AndroidEAAudioCore.{Init, Release}`. `JavaToNativeBridge.getURL` is declared native in Java but not exported by name (so it is
  registered at runtime or unused).
- `saveBlazeParameters` / `newBlazeParametersAvailable` take eight strings: authCode, blazeServiceName, gameId, personaId,
  platform, gameProtocolVersion, blazeEnvironment, mutedPlayers. `GameActivity.parseBlazeArguments` reads them from the intent
  extras `-authenticationCode`, `-blazeServiceName`, `-gameId`, `-personaId`, `-platform`, `-gameProtocolVersion`,
  `-blazeEnvironment`, `-mutedPlayers` and calls `saveBlazeParameters` when the auth code is non-empty (in `onCreate` and again in
  every `onResume`).
- `GameActivity.ValidateConnection` reports connectivity to native as `onJavaScriptEvent(94|95|96, 0)` = lost | Wi-Fi | mobile.
- Java the library calls (from strings: class names and about 25 method signatures): `GameActivity` statics
  (`staticGetPackageName`, `staticGetDeviceModel`, `staticGetOSVersion`, `staticGetNetworkConnection`, `staticShowWebViewBegin(url)`,
  `staticShowWebViewEnd`, `staticHideWebView`, `staticValidateConnection`, `isExpansionFileAvailable`), `SettingsManager`
  `get/put{Bool,Float,Int,String}(key, default)` (Android SharedPreferences), plus framework calls: `getAssets`, asset
  `open`/`openFd`, `getClassLoader`/`loadClass`, `getPackageManager`/`getActivityInfo`, `getWindowManager`/`getDefaultDisplay`/
  `getMetrics`, `getExternalStorageDirectory`. Which are actually hit at runtime is not traced yet (issue 013).
- The WebView in `GameActivity` is a popup dialog (in-app Battlelog and web pages), not the commander screen. The Java `-super`
  intent extra only gates the OBB downloader.
- OBB: the manifest's `obbDataVersion` picks `main.<version>.<package>.obb` under `Environment.getExternalStorageDirectory()`
  `/Android/obb/<package>/`.

## Leftovers from issue 010 (closed 2026-11-11)
- `ControlPointId` in an Order `TriggerAction` is the `MapMarkerList` id of the marker at the objective's position (not `ObjectiveId`).
- `InVehicleId` is the vehicle's Id, 0 when on foot.
- The unfinished page is at `scripts/game/index.html` and `scripts/game/actions.js`.
