# dungeon info — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Recovered test plan — Ticket / Dungeon Info / Battle Pass

### Ticket
- TICKET-T01 foreign valid ticket ID -> PAGE_REPLY.
- TICKET-T02 SQL metacharacters/apostrophe in create/reply/reason on disposable DB.
- TICKET-T03 deterministic ticket-ID collision.
- TICKET-T04 staff sort modes 0/5/255 and high pages.
- TICKET-T05 client log request id == vector size under ASan.
- TICKET-T06 crafted non-NUL fixed-char subpacket.

### Dungeon Info
- DUNGEON-T01 WARP/RANK 255 and vector-size boundary.
- DUNGEON-T02 GC index 255 on client under ASan.
- DUNGEON-T03 multi-entry reload then inspect stale client slots.
- DUNGEON-T04 mismatch LEVEL_LIMIT vs ENTRY_BASE_POSITION counts.
- DUNGEON-T05 exceed required-item/boss-drop config capacities.
- DUNGEON-T06 POINT_MAX_NUM+1 bonus entries.
- DUNGEON-T07 put a >QUEST_NAME_MAX_LEN token into an isolated dungeon_info.txt and reload under ASan; covers BUG-DUNGEON-007.
- DUNGEON-T08 call Python getters with bonus index/type 255/65535 and required/boss slot 255 under ASan; covers BUG-DUNGEON-008.
- DUNGEON-T09 request a valid ranking and capture the GAME SQL error; verify the missing whitespace before LEFT JOIN; covers BUG-DUNGEON-009.
- DUNGEON-T10 — **RETRACTED / DO NOT RUN**. The underlying BUG-DUNGEON-010 was a false-positive indentation reading; current code creates list buttons for nonzero counts.
- DUNGEON-T11 inspect `dragon_lair_access dragon_lair_time 1`; compare PC quest flag vs global event flag selection; covers BUG-DUNGEON-011.
- DUNGEON-T12 with an unset/expired configured quest cooldown flag, open Dungeon Info and verify the wrapped huge cooldown; covers BUG-DUNGEON-012.

### Battle Pass
- BP-T01 complete mission, then invoke any reachable SetExt caller again at threshold; check duplicate reward.
- BP-T02 capture MISSION_UPDATE packets; verify bMissionType nondeterminism/mismatch.
- BP-T03 long season/config name against RequestOpen under ASan.
- BP-T04 open pass repeatedly to expose dangling season_name behavior under ASan/UBSan.
- BP-T05 remove playerindex row then request final reward.
- BP-T06 Event Manager start/stop season with configured ID >1 and inspect active ID.
- BP-T07 process boot before Event Manager population; inspect uninitialized array values.
- BP-T08 season rollover/reload with existing mission/playerindex rows.


### Runtime preflight — 2026-09-26

#### DUNGEON-T10 — RETRACTED / DO NOT RUN
Fresh 2026-09-29 source verification shows the list-button creation loop is outside the zero-count `else` branch. The former deterministic prediction was incorrect.

#### DUNGEON-T09 — LIVE PENDING, FIRST LIVE GATE
Normal trigger after list population is restored:
`select dungeon -> ranking button -> dungeonInfo.Ranking(index,type) -> CG RANK -> CDungeonInfoManager::Ranking`.

The current SQL literal boundary produces `dungeon_ranking\`LEFT JOIN` with no whitespace. Expected live evidence:
- GAME DB/SQL error when ranking is requested;
- ranking window receives no real rows.


#### DUNGEON-T11 — LIVE PENDING, current-data mismatch confirmed
Current config line:
`QUEST dragon_lair_access dragon_lair_time 1`

Quest implementation uses `game.get_event_flag("dragon_lair_time")` / `game.set_event_flag("dragon_lair_time", ...)`, proving this is a global event flag.

The DungeonInfo parser recognizes GLOBAL only when token 3 is literal `GLOBAL`; numeric `1` falls into the PC-flag branch. Thus the current config/quest pair is semantically mismatched.

Expected live evidence after T10 is fixed/bypassed:
- Dungeon Info for map 208 derives cooldown from per-character `dragon_lair_access.dragon_lair_time` instead of global `dragon_lair_time`;
- changing the real global flag will not be reflected correctly by this path.

#### DUNGEON-T12 — LIVE PENDING, arithmetic path confirmed
All 4 QUEST-backed current dungeon blocks have no explicit `COOLDOWN` line, leaving configured cooldown at 0.

For unset/expired flags, `SendInfo` evaluates:
`uint32_t dwRemainSec = (dwFlagValue + 0) - get_global_time();`

A negative result wraps to a very large positive uint32 and passes `dwRemainSec > 0`.

Expected live evidence after T10 is fixed/bypassed:
- a QUEST-backed dungeon with zero/expired flag displays an abnormally huge cooldown rather than zero/available.

#### Dependency decision — corrected 2026-09-29
Runtime normal-path order is now:
1. DUNGEON-T09 — first future live gate.
2. DUNGEON-T11.
3. DUNGEON-T12.

DUNGEON-T10 is retracted and must not be executed. Do not mark T09/T11/T12 runtime PASS before the client/server is actually run.


### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no runtime test executed.
- Normal-path order is fixed as T10 -> T09 -> T11 -> T12.
- Isolated ASan/debug order is T01 -> T08.
- T10 remains the first live gate for the entire runtime phase.
- This cluster is ready to resume without rereading unrelated subsystem files.


### DUNGEON-T10 canonical handoff — 2026-09-28
Full future execution checklist: `../DUNGEON_T10_HANDOFF.md`.

Current status:
- preflight/documentation: **COMPLETE**;
- runtime execution: **LOCKED / NOT RUN**;
- canonical bug: `DINFO::BUG-DUNGEON-010`;
- expected current snapshot: 9 configured dungeons;
- allowed future trigger: normal login + minimap Dungeon Info button only;
- result classes: `REPRODUCED | NOT REPRODUCED | INCONCLUSIVE`;
- mandatory stop: after first evidence capture, before any fix/bypass.

No later Dungeon Info live test may be marked executed before the DUNGEON-T10 result is recorded.


### 2026-09-29 preflight correction
- BUG-DUNGEON-010 retracted.
- DUNGEON-T10 retracted.
- Current first future live gate: **DUNGEON-T09**.
- Execution remains locked.


### T11/T12 source revalidation — 2026-09-29
**T11:** current parser uses `strcmp(szValue3, "GLOBAL") == 0`; numeric `1` becomes `QUEST_FLAG_PC`. Current Blue Dragon config uses `QUEST dragon_lair_access dragon_lair_time 1`, while the quest itself uses `game.get_event_flag("dragon_lair_time")` and `game.set_event_flag("dragon_lair_time", ...)`. Preflight remains valid.

**T12:** current QUEST-backed config contains no explicit `COOLDOWN` lines, leaving `dwCooldown=0`. In the expired branch, `(dwFlagValue + dwCooldown) - get_global_time()` is assigned to `uint32_t`; negative results wrap positive. Preflight remains valid.

Canonical handoffs: `../DUNGEON_T11_HANDOFF.md`, `../DUNGEON_T12_HANDOFF.md`.
