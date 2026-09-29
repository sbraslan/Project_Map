# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static Mapping Coverage Complete  
**Status:** CLASSIC PET STATIC COMPLETE / 4 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** none — latest completed: Classic Pet System  
**System:** `systems/pet.md`  
**Bugs:** `bugs/pet.md`  
**Tests:** `tests/pet.md`  
**Last completed subsystem:** Classic Pet System  
**Effective completed/readiness-covered subsystems:** 29  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Classic Pet progress
Promoted:
- `BUG-PET-001` — Bruce pickup range ignores Y distance.
- `BUG-PET-002` — Bruce caches a raw ground-item `LPITEM` across update ticks; owner manual pickup can destroy the object before the pet dereferences it again.
- `BUG-PET-003` — REAL_TIME PET_PAY expiry removes the summon item without PetUnsummon; the missing-item actor branch returns before Unsummon and the update event keeps scheduling.
- `BUG-PET-004` — forced summon-item loss prevents elapsed TYPE_SUMMON_PET time from being credited; logout later clears the stale summon-time flag without credit.

Deferred tests:
- `PET-T01`
- `PET-T02`
- `PET-T03`
- `PET-T04`

Also closed:
- normal PET_PAY toggle uses explicit `PetUnsummon`;
- forced lower-level item removal does not call `PetUnsummon`;
- missing summon-item branch in `CPetActor::Update` returns false before `Unsummon`;
- pet-system event ignores that Update return and continues scheduling.

Newly closed:
- deployed PET_PAY REAL_TIME reachability is confirmed from current decoded item proto, including Bruce 53233;
- `petsystem_update_event` ignores the false update result and reschedules;
- owner destruction eventually tears down the pet system and stale actor.

## Exact next work
1. locate/exclude active `pet.summon()` quest producers;
2. map remaining PET_PAY item/race/client coverage;
3. close Achievement TYPE_SUMMON_PET abnormal cleanup consequence;
4. audit remaining death/warp/login restoration edges.

Do not execute `PET-T01`, `PET-T02` or `PET-T03`.
GitHub state is canonical.


## Latest checkpoint
- active tracked quest package contains no `pet.summon()`/related producer; legacy Lua signature mismatch remains dormant and is not promoted;
- Achievement 60 actively tracks 30 days of `TYPE_SUMMON_PET` time;
- REAL_TIME item loss can discard the current uncommitted summon interval -> `BUG-PET-004`.

## Exact next work
1. finish PET_PAY item/race/client coverage;
2. close death/warp/login restoration edges;
3. audit remaining auto-pickup ownership/re-target lifetime boundaries;
4. decide Classic Pet STATIC COMPLETE.

Do not execute `PET-T01`..`PET-T04`.


## Classic Pet final closure
- 143 current PET_PAY rows -> 135 unique pet races.
- 135/135 race VNUMs exist in server mob_proto.
- 135/135 race VNUMs exist in client npclist.
- raw client asset packs are only partially tracked; missing raw folders are deployment validation, not a promoted defect.
- owner death cleanup, same-process warp relocation, teardown and login `CheckPet` reconstruction are statically closed.
- PET_AUTO_PICKUP final collection revalidates item VID, sectree and ownership.
- no new bug beyond `BUG-PET-001..004`.

**Classic Pet System: STATIC COMPLETE.**

Runtime remains locked. First future live gate is `DUNGEON-T09`.


## Runtime-gate correction — 2026-09-29
Fresh preflight found the previous first-gate premise was wrong:
- `BUG-DUNGEON-010` retracted;
- `DUNGEON-T10` retracted / do not run;
- current `uidungeoninfo.py` creates list buttons for nonzero dungeon counts;
- `BUG-DUNGEON-009` malformed ranking SQL remains statically verified.

**New first future live gate: `DUNGEON-T09`.**
Execution remains locked.


## Dungeon normal-path handoff closure — 2026-09-29
Fresh source/data verification:
- `DUNGEON-T09`: malformed ranking SQL remains verified and is the first future live gate.
- `DUNGEON-T11`: config token `1` is parsed as PC because parser only accepts literal `GLOBAL`; Blue Dragon quest actually reads/writes `dragon_lair_time` via global event flags.
- `DUNGEON-T12`: QUEST-backed entries have no explicit COOLDOWN lines; expired/zero values can underflow into a large `uint32_t` cooldown.

Canonical handoffs are now prepared for T09, T11 and T12.
Runtime remains locked; no live test was executed.


## Ticket normal-path preflight — 2026-09-29
- `TICKET-T07` current 40(server) / 10(C++ cache) / 20-per-page(Python UI) mismatch reverified.
- direct BUG-TICKET-005 OOB is not part of normal T07; UI only requests C++ rows 0..9.
- canonical handoff: `TICKET_T07_HANDOFF.md`.
- global first future live gate remains `DUNGEON-T09`; Ticket follows the Dungeon normal-path cluster.


## Hunting normal-path preflight — 2026-09-29
- `HUNT-T04` current terminal boundary reverified.
- mission 90 claim unconditionally stores hunting level 91.
- valid table domain remains 0..90.
- reproduction requires character level >=91 before reopening Hunting.
- canonical handoff: `HUNT_T04_HANDOFF.md`.
- global first live gate remains `DUNGEON-T09`; no Hunting runtime test executed.


## Battle Pass normal-path preflight — 2026-09-29
- `BP-T02` mission-update packet omission reverified across all mapped send paths.
- `bMissionType` is never assigned before send.
- client forwards it unchanged; progress may still update because HaveMission matches by index only.
- selected mission detail can consume the undefined mission type via SetMissionInfo.
- canonical handoff: `BP_T02_HANDOFF.md`.
- global first live gate remains `DUNGEON-T09`; no Battle Pass runtime test executed.


## Achievement normal-path preflight — 2026-09-29
- `ACH-T03` missing gameplay-hook families reverified against current config: 16 mount / 5 search-shop spend / 3 shop spend / 5 withdraw tasks.
- `ACH-T04` reverified: 29 explore tasks; progression hook is reached from login, not mapped normal map transitions.
- canonical handoffs: `ACH_T03_HANDOFF.md`, `ACH_T04_HANDOFF.md`.
- global first live gate remains `DUNGEON-T09`; no Achievement runtime test executed.


## Biolog normal-path preflight — 2026-09-29
- `BIO-T01` completion bridge reverified.
- C++ sets `biolog_manager.*` completion flags and client requests event quest `biolog_manager`.
- current Project_Game quest_list contains only legacy collect_* Biolog quests; no `biolog_manager` implementation was found.
- canonical handoff: `BIO_T01_HANDOFF.md`.
- actual biolog mission/reward values remain live-DB dependent.
- global first live gate remains `DUNGEON-T09`; no Biolog runtime test executed.


## Inventory / Item normal-path preflight — 2026-09-29
- `ITEM-T01` destroy-count semantics reverified.
- normal Binary UI forwards the selected partial `dropCount` into `SendItemDestroyPacket`.
- server receives count but `CHARACTER::RemoveItem` does not use it; full item stack is destroyed.
- adjacent `BUG-ITEM-001` UAF may interrupt the same flow and remains owned by `ITEM-T02`.
- canonical handoff: `ITEM_T01_HANDOFF.md`.
- global first live gate remains `DUNGEON-T09`; no Inventory/Item runtime test executed.
