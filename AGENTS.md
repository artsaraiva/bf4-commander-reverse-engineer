# BF4 Commander reverse engineering

Goal: make Battlefield 4's discontinued tablet **Commander** mode playable again with its **original UI**, through a **desktop loader** that runs the original app's native library with no phone and no Android. Only for people who own BF4 — every session is authorised by the player's own Battlelog login.

## Current phase: the loader (since 2026-11-11)

- **Why:** `docs/adr/0001-run-the-original-library-in-a-desktop-loader.md`.
- **How:** `notes/loader-spec.md` holds what all loader issues share: decisions, architecture, milestones, stop rule, phase guardrails. Read it whole before working on any loader issue.
- **Facts:** `notes/findings.md` is the single source of truth for everything learned (APK and binary facts, launch flow, Battlelog internals, protocols, dead ends). Read it before starting an issue; add new facts there, not in issues.

## Issues

Work lives in GitHub issues on `artsaraiva/bf4-commander-reverse-engineer`, under macro issue #1. Its sub-issues keep their legacy numbers 012–021 in the title, because the spec and findings cite those numbers: **GitHub number = legacy number − 10** (012 is #2, 021 is #11). Start with #2 and #3, which are independent.

- Use the player's personal GitHub account for every `gh` call: `GH_TOKEN=$(gh auth token --user artsaraiva) gh …`.
- Each issue lists its dependencies and a "Done when" checklist; it is finished only when every box is ticked. Tick boxes in the issue body as they are met, post progress as comments, and close the issue when the last box is ticked.

## Phase 1 (local only)

Phase 1 (legacy issues 001–011, 2026-11-10/11) got Commander playing on a live PC server two ways: the original app on a 32-bit-capable Android phone, launched from Battlelog through a Chrome extension and mirrored into a browser tab, and our own Blaze + game-server client. Findings describes both. Their code (`scripts/`, `extension/`) and issue files (`issues/`) stay on the player's disk, gitignored, as fallbacks and references. Leave them as they are; when a loader issue needs one of those files (013 the Frida tooling, 019 the extension and `scripts/commander_bridge.py`), bring that file into the repo as part of the issue.

## Guardrails

- **Public repository.** Commits, issues and comments are public: no secrets, no EA files, no personal data (IP addresses, persona names, device serials).
- **Auth codes are secrets.** A Battlelog `authenticationCode` logs into the player's EA account. Keep it masked in logs, console output and chat (first 4 chars max); keep raw values only in memory or in gitignored `captures/`.
- **The human logs in.** Credentials are typed by the player in the browser window; agents never handle passwords or 2FA.
- **EA files stay local.** `apk/`, `captures/` and `*.obb` are gitignored; EA binaries, assets and anything derived from their contents are never committed or published.
- **Live service, real account.** Everything online talks to EA's production servers with the player's account. Prefer observing (logging, hooking the official app) over sending crafted traffic. Ask the player before anything that sends non-official traffic to EA, and before the loader's first online run in a session (019). Turn any work VPN off first: EA's redirector doesn't answer through it.
- **Installs on the player's phone** of old APKs need `adb install --bypass-low-target-sdk-block`; the player runs that themselves (or grants a permission rule).

## Environment

- Mac (Apple Silicon). Android emulators can't run the app: no 32-bit ARM support (details in findings). Use the player's phone.
- Phone: Galaxy S23 with Commander 2.2.0 + OBB installed; `adb devices` shows its serial.
- Tools installed: Android SDK at `~/Library/Android/sdk` (adb, `cmdline-tools/latest`, build-tools incl. `aapt2`/`apksigner`, NDK `llvm-nm`/`llvm-readelf`), `jadx` (Homebrew), Python venv `scripts/.venv` (frida-tools, unicorn).
