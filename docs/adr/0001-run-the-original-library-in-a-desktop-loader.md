---
status: accepted
date: 2026-11-11
---

# Run the original Commander library in a custom desktop loader

EA discontinued Battlefield 4's tablet Commander app, but the PC servers still accept a commander. Phase 1 (2026-11-10/11, legacy
issues 001–011) found two ways to play. The first runs the original app on a 32-bit-capable Android phone, launched from Battlelog
and mirrored into a browser tab. The second is our own client (Blaze, ProtoTunnel, game netcode), which joins as commander but needs
its own UI. The player judged that an own UI can't match EA's. **We build a desktop program that loads the player's own copy of the
original app's native library (`libFirstContact.Main_android.so`, 2.2.0) and stands in for Android around it**: ELF loading,
ARM32-to-host CPU translation, and shims for libc, JNI, EGL/OpenGL ES, input and audio. The player gets EA's own commander UI with no
phone and no Android. The design is in `notes/loader-spec.md`. The work is tracked under GitHub issue #1.

## Considered options

- **Phone + browser mirror** (works today): keeps a phone and Android in the loop, which the player wants gone. It stays as the
  fallback.
- **Own UI on our own client** (legacy issue 010): stopped because it can't match EA's UI.
- **Android emulator**: still Android. On Apple Silicon it can't run 32-bit ARM apps at all.
- **Cloud Android**: puts EA binaries and the player's auth codes on a server.
- **All in the browser** (WASM CPU emulator): needs everything the loader needs plus a WASM CPU, and a browser can't open the UDP and
  TLS 1.1 sockets the library uses.

## Consequences

- The repository contains no EA code or assets. The loader reads the library and OBB from the player's disk and never patches them.
- Lock-in: C++, dynarmic (ARM32 JIT), SDL2 and ANGLE (macOS has no OpenGL ES). A throwaway Python + Unicorn spike comes first.
- macOS on Apple Silicon first. Windows is a later port.
- The real app on a phone, traced with Frida, stays the reference during development: every shim is checked against what Android
  answered. A phone is needed to build the loader, not to play with it.
- Stop rule: if the library can't get past startup without substantial real Java code, stop and fall back to the phone mirror
  instead of emulating Android piece by piece.
- Phase 1 code stays on the player's disk as a fallback and protocol reference. It isn't developed further and isn't in this
  repository.
