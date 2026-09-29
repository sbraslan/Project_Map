# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Fishing Renewal mapping in progress  
**Status:** 29 STATIC COMPLETE / FISHING ACTIVE / 5 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Fishing Renewal  
**System:** `systems/fishing.md`  
**Bugs:** `bugs/fishing.md`  
**Tests:** `tests/fishing.md`  
**Last completed subsystem:** Classic Pet System  
**Effective completed/readiness-covered subsystems:** 29  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Fishing Renewal initial checkpoint
Verified:
- `ENABLE_FISHING_RENEWAL` is active.
- client renewed UI exists in `Project_Binary/root/uifishing.py`;
- server dispatch: `HEADER_CG_FISHING_NEW -> CInputMain::FishingNew -> CHARACTER::fishing_new_*`;
- current rod names include 27400..27490 (+1..+10), 27500..27590 (+11..+20), and 27591 Carbon rod.

Promoted:
- `BUG-FISH-001` — second normal fish table has 5 entries but is indexed with `number(0,6)`; normal renewed fishing with +11..+20/Carbon rods reaches `second=true`.
- `BUG-FISH-002` — `fishing_new_start()` creates temporary item 50187 for inventory probing and never destroys it; repeated starts can accumulate ownerless registered items.
- `BUG-FISH-003` — renewed fishing successful-hit validation is client-authoritative; server accepts timed CATCH packets without verifying the UI hit test.
- `BUG-FISH-004` — renewed fishing does not server-lock movement or revalidate fishing position after start.
- `BUG-FISH-005` — Carbon rod 27591 special doubled chance branch is impossible because the same condition also requires `dwVnum <= 27490`.

Open candidate:
- Carbon-rod bonus branch is impossible as written: `dwVnum == 27591 && dwVnum >= 27400 && dwVnum <= 27490`. Intended effect still needs semantic closure.

## Exact next work
1. map renewed fishing packet registration and size validation;
2. audit catch/fail timing, replay and trust boundaries;
3. close bait + `POINT_FISHING_RARE` arithmetic;
4. audit success/log/Battle Pass/Achievement flow;
5. audit stop/death/warp/logout/equipment-change cleanup;
6. audit rod refine/current proto values;
7. promote only verified reachable additional findings.

Do not execute `FISH-T01`..`FISH-T05`. Global first live gate remains `DUNGEON-T09`.
