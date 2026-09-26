# CURRENT — Canonical Active Checkpoint

**Active subsystem:** Hunting System
**Status:** PARTIAL
**Machine state:** `STATE.json`
**Last updated:** 2026-09-26

## Startup read set
For a normal "ilerleyelim" turn read only:
1. `STATE.json`
2. `CURRENT.md`
3. `systems/hunting.md`

Conditional:
- `bugs/hunting.md` only when validating/recording a bug.
- `tests/hunting.md` only for runtime/fault-injection work.
- source repos: search first, then fetch only exact files/ranges needed.

Do **not** reconstruct state from old chats. GitHub state is canonical.
Do **not** read `archive/`, completed subsystem files, or legacy root maps unless recovery is required.

## Current Hunting state
Mapped:
- server runtime: `game/src/char_hunting.cpp`
- CG action dispatch: `input_main.cpp::ReciveHuntingAction`
- kill progress hook: `char_battle.cpp::UpdateHuntingMission`
- login/level bootstrap: `input_login.cpp::CheckHunting` + `char.cpp::CheckHunting`
- client UI/send: `root/uihunting.py` + `m2netm2g.SendHuntingAction`
- persistence: `hunting_system.*` quest flags

Verified bugs: `BUG-HUNT-001..004`.

## Exact next work
1. Validate all Hunting static mission/reward tables and exact dimensions.
2. Finish client parser + packet-info/sequence audit.
3. Audit reward item grant failure/ground fallback and claim crash atomicity.
4. Verify quest-flag save/relogin behavior at reward boundaries.
5. Close final mission level-90 behavior end-to-end.

## End-of-turn write rule
When meaningful progress is made:
1. update `systems/hunting.md`;
2. update `bugs/hunting.md` / `tests/hunting.md` only if affected;
3. overwrite this file with the new short cursor;
4. update `STATE.json`.

Keep this file short and overwrite-only. Git history is the checkpoint history.
