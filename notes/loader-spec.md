# Loader phase — spec

Written 2026-11-11 after a design interview with the player. This is the plan for the phase that follows issues 001–011. The work is
split into issues 012–021 (GitHub #2–#11, under #1); this file holds what they share. The decision itself and the options rejected
are in `docs/adr/0001-run-the-original-library-in-a-desktop-loader.md`. Facts about the binary live in `notes/findings.md` → "Loader
phase — binary inventory"; add new facts there, not here. Change this file only when a decision changes.

## Goal

A desktop program that runs the **original** Commander app's native library, so the player gets EA's own commander UI in a desktop
window with no phone and no Android.

- The player clicks the Commander button in Battlelog (existing Chrome extension). A window opens showing the real app, joined to
  that server.
- The program loads the player's own `libFirstContact.Main_android.so` (2.2.0) and `main.11.com.ea.game.warsawcommander_row.obb`
  from disk. It contains and ships no EA code or assets.
- It replaces Android for that one library: ELF loading, CPU translation (32-bit ARM → host), the C library, OpenGL ES, input, audio
  and the small Java surface the library talks to.

## Decisions (settled with the player, 2026-11-11)

| Decision | Choice | Why |
|---|---|---|
| Approach | Custom loader for the `.so` | The player wants the original UI, no phone and no Android in the result. The own UI (010) was judged unable to match it. |
| Rejected | x86 Android emulator, cloud Android, literal all-web | Emulator: still Android, and unusable on Apple Silicon. Cloud: puts EA binaries and auth codes on a server. All-web: needs everything below plus a WASM CPU emulator, and a browser can't open UDP or TLS 1.1 sockets. |
| Host | macOS on Apple Silicon first | The phone and the Frida setup are here. Windows is a later port, not a place to work. Keep the code portable (no macOS-only APIs outside one file) but don't build for Windows yet. |
| Spike | Python + Unicorn, throwaway | Fits the existing Python tooling and answers "does the library boot outside Android, and what does it ask for" in days. Too slow to play on. |
| Real loader | C++, dynarmic (ARM32 JIT), SDL2 (window, input), ANGLE (OpenGL ES on Metal) | dynarmic is the fastest ARM32 JIT and is a C++ library. macOS has no OpenGL ES, so ANGLE is required. |
| Reference | The real app on the phone, traced with Frida | Every shim is checked against what Android actually answered. Observation only. |
| Network | Offline until a frame renders. The player approves the first live connection. | The traffic is the official client's own code, but it is the player's real account. |
| Audio | Silent stubs until a live match works | Not needed to play. |
| Audience | The player. No release planned. | The loader ships no EA files, so a release stays possible later. |
| Old work | Own Blaze client, companion, game page, mirror, map-art reader stay on disk, no further work | Insurance, and the protocol notes help debug the loader's network stage. |
| Launcher | The existing Chrome extension | It already delivers the launch parameters to `127.0.0.1:47615`. |

## Stop rule

If the spike (012) or the reference trace (013) shows the library can't get past startup without real Java code running — for
example it calls into Java classes that do substantial work we would have to reimplement — stop, record why in findings, and fall
back to the phone mirror (`scripts/mirror.py`). Don't start emulating Android piece by piece. The Java surface looks small
(findings → "Java surface"), so this is not expected.

Each milestone below is also a point to reassess with the player before the next one starts.

## Milestones and issues

| Milestone | Issues | Reached when |
|---|---|---|
| 1. Spike | 012, 013 | The library is loaded and its startup functions run under Unicorn, or we know exactly what stops them. A reference trace of the real app's startup exists. |
| 2. First frame | 014, 015, 016, 017 | The C++ loader draws the app's first screen in a desktop window, reading the OBB, with no network. |
| 3. Launch | 018, 019 | A Battlelog click starts the loader with the launch parameters, the mouse works, and the library logs in to Blaze (player-approved). |
| 4. Live match | 020 | The player commands a full round with the mouse. |
| After | 021 | Sound. |

Dependencies: 012 and 013 are independent of each other. 014 needs 012. 015 and 016 need 014 and use 013. 017 needs 015 and 016.
018 needs 017. 019 needs 016 and 017. 020 needs 018 and 019. 021 needs 020.

## Architecture

```
Battlelog tab ── extension ──► 127.0.0.1:47615 (launcher, Python, existing checks)
                                    │ launch parameters as JSON on stdin
                                    ▼
                    loader (one C++ process)
   ┌───────────────────────────────────────────────────────────────┐
   │ guest memory (32-bit address space, one host reservation)     │
   │   libFirstContact.Main_android.so   stacks   heap   stubs     │
   │                                                               │
   │ CPU: one dynarmic A32 JIT per guest thread, all sharing the   │
   │      guest memory; each guest thread is a host thread         │
   │                                                               │
   │ imports ─► trap ─► host shims                                 │
   │   libc / libm / pthread      files re-rooted under data dir   │
   │   sockets                    real host sockets                │
   │   EGL / GLES 1+2             ANGLE                            │
   │   libandroid (looper, input) SDL2 events                      │
   │   OpenSL ES                  silent                           │
   │   JNI (fake JavaVM/JNIEnv)   small table of Java methods      │
   └───────────────────────────────────────────────────────────────┘
        reads: player's .so, OBB, APK assets        window: SDL2
```

### Guest memory

- Reserve one contiguous host region for the whole 32-bit guest address space (or the part in use) so that
  `host pointer = base + guest address`. Every shim that takes or returns a pointer converts with that one rule. Guest address 0
  stays unmapped so null dereferences fault.
- The library is position independent (`ET_DYN`, link base 0). Load it at a fixed guest address and keep it fixed across runs, so
  addresses in logs and in Frida traces (module-relative) line up.
- A simple guest heap for `malloc`/`mmap` shims lives in the same region. The library's allocations must come from guest memory, not
  the host heap, because the guest stores and dereferences those pointers as 32-bit values.

### ELF loading

The 2.2.0 library needs very little (findings → "ELF layout"): two `PT_LOAD` segments, `REL` relocations of three types
(`R_ARM_RELATIVE`, `R_ARM_GLOB_DAT`, `R_ARM_JUMP_SLOT`), 111 `INIT_ARRAY` constructors, no TLS segment. Steps: map segments, apply
relocations, point each import at its stub, run `INIT_ARRAY` in order, then `JNI_OnLoad`. Honour `DT_SYMBOLIC`. The exception index
(`PT_ARM_EXIDX`) must be reachable through whichever lookup the library imports (`__gnu_Unwind_Find_exidx` or
`dl_unwind_find_exidx`): C++ exceptions unwind inside the guest and need it.

### Import traps

- Each imported **function** gets a stub in a reserved guest page: `svc #n` then return. The JIT's supervisor-call callback looks up
  shim `n`, reads arguments, calls the host function and writes the result to `r0` (and `r1` for 64-bit results).
- Each imported **data** symbol (`R_ARM_GLOB_DAT` to things like `__sF`, `__stack_chk_guard`, `tzname`, `timezone`) gets real guest
  memory with sensible contents.
- An import with no shim yet gets a default stub that logs its name and the guest return address, then aborts. Never return 0
  silently: a wrong answer deep in startup costs far more time than a crash at the call.
- **Calling convention.** The library is `armeabi` (ARMv5TE, soft float). Arguments are in `r0`–`r3`, then the stack. `float` and
  `double` travel in core registers and on the stack, not in VFP registers. 64-bit values (`long long`, `double`) are aligned to an
  even register pair and to 8 bytes on the stack. Write one argument reader that knows these rules and use it for every shim.
  Variadic functions (`printf` family, `open`, `ioctl`, `fcntl`) read the guest's argument area directly.
- **Callbacks into the guest** (thread start routines, `qsort`/`bsearch` comparators, `atexit`, `pthread_once`, `pthread_key`
  destructors, NativeActivity callbacks, JNI natives) run by re-entering the JIT on the current thread: save registers, set
  `r0`–`r3` and `lr` to a "return" stub, run until that stub traps, restore.
- **Kernel helpers.** ARMv5 code has no atomic instructions and reads the thread pointer through the kernel's fixed-address helpers
  at `0xffff0fa0`–`0xffff0fff` (`__kuser_cmpxchg`, `__kuser_memory_barrier`, `__kuser_get_tls`). Expect the library to call them;
  the spike confirms. The loader must provide them (as traps at those addresses) and `__kuser_cmpxchg` must be atomic across host
  threads.

### C library shims

The library imports about 250 libc, libm and pthread functions (findings → "Imports"). Most forward to the host's function after
pointer conversion. The ones that don't:

- **Struct layouts are 32-bit bionic's, not the host's.** `pthread_mutex_t` and `pthread_cond_t` are 4 bytes in the guest, so a
  host mutex can't live inside them. Keep host objects in a side table keyed by guest address, created on first use (statically
  initialised guest mutexes are just zero bytes). `struct stat`, `struct dirent`, `struct timeval`/`timespec` (32-bit `time_t`),
  `struct tm`, `struct sockaddr*`, `struct addrinfo`, `struct hostent` and `fd_set` all need explicit conversion.
- **Linux numbers, not Darwin's.** The guest was built for Linux: `errno` values, `open` flags, socket option levels and names,
  address families, `fcntl` and `ioctl` requests, `poll` events and signal numbers differ on macOS. Translate at the shim boundary in
  both directions. `sockaddr` on macOS has a length byte where Linux has a 16-bit family.
- **`FILE*`** is an opaque guest handle mapped to a host `FILE*`. `__sF` (stdin, stdout, stderr) is a data import.
- **`errno`** is per thread, reached through `__errno()`, which returns a guest pointer.
- **Files.** Every guest path is re-rooted under one data directory (default `~/Library/Application Support/bf4-commander-loader/root`),
  so `/sdcard/Android/obb/com.ea.game.warsawcommander_row/main.11….obb` resolves to a file or symlink the player put there, and
  anything the library writes stays inside. No path escapes the root.
- **Threads.** `pthread_create` starts a host thread with its own guest stack and its own JIT instance, then runs the guest start
  routine through the callback mechanism. Thread-local storage (`pthread_key_*`) is kept host-side per thread.
- **Time.** Use the real clock. The library's timers drive networking keep-alives, so no fake time.

### Fake Java (JNI and NativeActivity)

The app's activity is a `NativeActivity` subclass: the library exports `ANativeActivity_onCreate` and runs its own main loop. The
Java class around it (`GameActivity`, 658 lines decompiled at `apk/mod/commander-jadx/…/GameActivity.java`) loads the library, shows a
splash and a download screen, hosts a WebView popup and passes the launch parameters. The loader plays that role:

- Build a guest `JavaVM` and `JNIEnv`: structs of function pointers, each pointing at a trap stub. Implement an entry when the
  library first calls it; unimplemented entries log and abort like any other import.
- Java objects are small integer handles. Classes, methods and fields are looked up by name and signature in a host table. The
  expected contents are in findings → "Java surface": `SettingsManager` getters and setters (backed by one JSON file in the data
  directory), a few `GameActivity` statics (package name, device model, OS version, network type, WebView show/hide), asset access,
  display metrics, external storage path, class loader lookups.
- Build a guest `ANativeActivity` (callbacks struct, `vm`, `env`, `clazz`, data paths, `sdkVersion`, `assetManager`), call
  `ANativeActivity_onCreate`, then deliver lifecycle callbacks in the order Android does: start, resume, input queue created, native
  window created, window focus gained. The exact order and timing come from the reference trace (013).
- `libandroid` imports (`ALooper_*`, `AInputQueue_*`, `AInputEvent_*`/`AMotionEvent_*`/`AKeyEvent_*`, asset functions if present)
  are implemented on top of the host event queue.
- Calls the loader makes **into** the library, standing in for Java: `saveBlazeParameters` with the eight launch strings,
  `onJavaScriptEvent(95, 0)` ("Wi-Fi connected", which `ValidateConnection` sends), and the `EAIO`/`EAThread`/`EAAudioCore` init
  natives. When and in which order Java calls them is a question for 013.
- The WebView popup (`staticShowWebViewBegin(url)`) is logged, and reported back to the library as closed. If a real need shows up
  (for example an EA terms page that blocks login), open the URL in the default browser. Don't embed a web view.

### Graphics

- The library imports both `libGLESv1_CM` and `libGLESv2` plus EGL (177 `gl*`, 14 `egl*` functions). Record which ones it actually
  calls (012, 013) before writing shims; implement only those.
- SDL2 creates the window; EGL and GLES calls go to ANGLE. `eglGetDisplay`, `eglCreateWindowSurface` and friends return guest
  handles mapped to host objects; the guest's `ANativeWindow*` is a handle for the SDL window.
- Pointer rules for GL: client-side vertex arrays (`glVertexAttribPointer` and the ES1 `gl*Pointer` calls) pass a guest address
  when no buffer is bound and an offset when one is: convert only in the first case. `glShaderSource` takes an array of guest
  pointers. `glGetString` returns a host pointer that must be copied into guest memory once and cached.
- Window size: the app is a landscape tablet app. Start with a fixed 16:10 window (1280×800) and report matching display metrics.
  Resizing is out of scope.
- Risk: compressed texture formats (likely ETC1) may not be available through ANGLE on Metal. If not, decode to RGBA in the shim.

### Input

Mouse and keyboard from SDL2 become Android input events in the guest's input queue: left button = one finger (down, move, up),
wheel or a modifier-drag = two-finger pinch, Escape = Back. `scripts/mirror.html` already solved pinch emulation for the phone
mirror; reuse the idea.

### Network

Socket shims map to real host sockets. TLS is inside the library (ProtoSSL, TLS 1.1), so the host provides only TCP, UDP and DNS.
The library then speaks to EA exactly as the app does. Until issue 019, sockets are refused (`ENETUNREACH`) unless the loader is
started with an explicit `--online` flag. Run with any work VPN off (same as `scripts/blaze_client.py`).

### Launch

The extension already POSTs the Battlelog `launchCommander` parameters to `127.0.0.1:47615`, where `scripts/commander_bridge.py`
validates them and calls `adb`. For the loader, the same listener starts the loader process instead and writes the parameters to its
stdin as JSON. Never put the auth code in `argv` (visible in `ps`) or in logs. The eight parameters, in the order
`saveBlazeParameters` takes them: `authenticationCode`, `blazeServiceName`, `gameId`, `personaId`, `platform`,
`gameProtocolVersion`, `blazeEnvironment`, `mutedPlayers`.

## Method: the phone is the reference

The real app on the Galaxy S23 is ground truth. Issue 013 records a startup trace with Frida (imports called with arguments and
results, JNI traffic, files opened, GL configuration, thread creation). From then on, the loader can write the same trace format,
and a diff of the two shows the first point where the loader's world differs from Android's. Use that before guessing. The gadget
build for the app already exists (findings → "Blaze capture runbook"); the 1.0.0 build's 76k symbols and `scripts/fmap.py` map
names onto the stripped 2.2.0 library.

## Layout

```
loader/
  spike/        012: Python + Unicorn, throwaway
  trace/        013: Frida scripts and the trace differ
  src/          014 onward: the C++ loader
  CMakeLists.txt
```

- Traces and anything derived from EA binaries go under `captures/` (gitignored). The loader reads the library and OBB from `apk/`
  (gitignored) or the data directory; nothing from EA is copied into `loader/`.
- Python in `loader/spike` and `loader/trace` may use `scripts/.venv` (add `unicorn` and `frida-tools` there). C++ dependencies:
  dynarmic and ANGLE fetched at build time or installed, SDL2 from Homebrew. Record the exact way in findings when 014 and 017
  settle it.
- Each issue leaves one runnable check behind (a script or a `--self-test` mode) that needs no phone and no network where possible.

## Guardrails for this phase

In addition to `AGENTS.md`:

- **No EA binaries or assets in the repo or in any published artifact**, including traces that embed their contents. The loader
  works only with files the player supplies.
- **Don't patch the library on disk.** Change behaviour through shims and loader-side hooks only.
- **Auth codes** stay masked in every log and trace (first 4 characters). Frida traces must mask them before writing to disk.
- **No network** from the loader before issue 019, and the first online run needs the player's go-ahead in that session.
- **Leave a live match cleanly.** Closing the window must let the library run its own disconnect path, so the commander slot is
  freed the way the app frees it.

## Out of scope

Windows and Linux builds, packaging and installers, window resizing and high-DPI polish, voice chat and push-to-talk, the in-app
WebView content, in-app OBB download, any change to EA's UI, and further work on the own client, companion, game page or mirror.

## Risks, most likely first

1. **A subtle shim mismatch crashes the library far from the cause.** Mitigation: abort on unknown imports, and the trace diff.
2. **dynarmic on Apple Silicon** (JIT memory permissions, ARMv5 code paths). Verify with a ten-line test before building on it
   (014). If it can't work, decide the replacement in that issue with the player.
3. **Threading**: atomics through `__kuser_cmpxchg`, many engine threads, callbacks re-entering the JIT.
4. **Graphics**: ES1 fixed-function calls and compressed textures through ANGLE.
5. **Hidden Android dependencies** found late (a Java class doing real work, a system service). The stop rule covers the severe case.
6. **EA-side checks** on device or OS strings the Java side reports. Answer with the same values the phone gives (from 013).
