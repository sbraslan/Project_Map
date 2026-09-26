# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Dungeon Core
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/dungeon_core.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Dungeon Core checkpoint
Mapped:
- CDungeonManager create/destroy;
- CHARACTER::SetDungeon enter/leave bookkeeping;
- IncMember/DecMember;
- IncPartyMember/DecPartyMember/QuitParty;
- party destruction ordering against dungeon raw party keys;
- warp/login destination SetDungeon binding;
- quest new_jump/new_jump_all/new_jump_party/join entry flows;
- dead/exit/jump event cancellation basics.

Verified bugs:
- `BUG-DUNGEON-001` — `d.join` / `d.new_jump_guild` can create a private dungeon before rejecting caller eligibility, leaving a memberless dungeon with no dead-event cleanup.

## Closed candidates
- normal party destruction cleans dungeon `m_map_pkParty` before CParty deletion;
- JoinParty map-null ordering lacks a normal live-dungeon/missing-map path;
- exit/jump event null ordering remains incorrect but destructor event cancellation currently prevents promotion.

## Exact next work
1. Audit JumpParty one-party ownership across nested dungeon creation.
2. Close event lifetime/ID reuse.
3. Audit participant registration across warp/core transition.
4. Audit dungeon item-group lifecycle.
5. Audit spawn/unique/regen pointers.

GitHub state is canonical.
