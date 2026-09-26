# CURRENT — Canonical Active Checkpoint

**Active subsystem:** Hunting System  
**Status:** PARTIAL  
**Checkpoint source before migration:** Project_Map commit `f154bc23dbf9d2aa45c0419fcb89f959f14f0117`  
**Last updated:** 2026-09-26

## Read set for the next "ilerleyelim"
Read only:
1. `CURRENT.md`
2. `systems/hunting.md`
3. `bugs/hunting.md` when validating/recording a bug
4. `tests/hunting.md` only for runtime-test work
5. exact source snippets needed from the read-only source repos

Do **not** read `archive/`, other subsystem files, or legacy root maps unless the active subsystem specifically depends on them.

## Current Hunting state
Mapped:
- server runtime: `game/src/char_hunting.cpp`
- CG action dispatch: `input_main.cpp::ReciveHuntingAction`
- kill progress hook: `char_battle.cpp::UpdateHuntingMission`
- login/level bootstrap: `input_login.cpp::CheckHunting` + `char.cpp::CheckHunting`
- client UI/send: `root/uihunting.py` + `m2netm2g.SendHuntingAction`
- persistence: `hunting_system.*` quest flags

Verified bugs currently: `BUG-HUNT-001..004`.

## Next work
1. Validate Hunting static mission/reward tables and exact dimensions.
2. Finish client parser + packet-info/sequence audit.
3. Audit reward item grant failure/ground fallback and claim crash atomicity.
4. Verify quest-flag save/relogin behavior at reward boundaries.
5. Close final mission level-90 behavior end-to-end.
6. When complete: update `systems/hunting.md`, `bugs/hunting.md`, `tests/hunting.md`, `INDEX.md`, then overwrite this file with the next subsystem.

## Write rule
This file is **overwrite-only**. Never append historical checkpoints here.
