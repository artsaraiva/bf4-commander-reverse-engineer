# BF4 Commander reverse engineering

Battlefield 4 had a tablet **Commander** mode: a separate Android/iOS app that let one player per team command the battle from a
map. EA discontinued the app, but the PC servers still accept commanders. This project aims to make Commander playable again with
EA's original UI, through a desktop program that runs your own copy of the original app's native library, with no phone and no
Android.

**Status:** research. The plan is in [ADR 0001](docs/adr/0001-run-the-original-library-in-a-desktop-loader.md) and
[the loader spec](notes/loader-spec.md). Progress is tracked in [issue #1](../../issues/1).

- Only for people who own BF4. Every session is authorised by your own Battlelog login.
- This repository contains no EA code or assets and never will. The loader works only with files you supply from your own copy of
  the app.
- Not affiliated with or endorsed by Electronic Arts or DICE.
