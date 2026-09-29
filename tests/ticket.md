# ticket — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Recovered test plan — Ticket / Dungeon Info / Battle Pass

### Ticket
- TICKET-T01 foreign valid ticket ID -> PAGE_REPLY.
- TICKET-T02 SQL metacharacters/apostrophe in create/reply/reason on disposable DB.
- TICKET-T03 deterministic ticket-ID collision.
- TICKET-T04 staff sort modes 0/5/255 and high pages.
- TICKET-T05 client log request id == vector size under ASan.
- TICKET-T06 crafted non-NUL fixed-char subpacket: for OPEN/CREATE/REPLY/ADMIN fill one fixed char field completely without a NUL and run GAME under ASan/UBSan. Covers BUG-TICKET-006.
- TICKET-T07 normal-user pagination mismatch: create >=25 tickets, open the Ticket UI, verify server sends up to 40 while client cache contains only 10; inspect page-1 rows 10..19 and pages 2+. Covers BUG-TICKET-007.

### Dungeon Info
- DUNGEON-T01 WARP/RANK 255 and vector-size boundary.
- DUNGEON-T02 GC index 255 on client under ASan.
- DUNGEON-T03 multi-entry reload then inspect stale client slots.
- DUNGEON-T04 mismatch LEVEL_LIMIT vs ENTRY_BASE_POSITION counts.
- DUNGEON-T05 exceed required-item/boss-drop config capacities.
- DUNGEON-T06 POINT_MAX_NUM+1 bonus entries.

### Battle Pass
- BP-T01 complete mission, then invoke any reachable SetExt caller again at threshold; check duplicate reward.
- BP-T02 capture MISSION_UPDATE packets; verify bMissionType nondeterminism/mismatch.
- BP-T03 long season/config name against RequestOpen under ASan.
- BP-T04 open pass repeatedly to expose dangling season_name behavior under ASan/UBSan.
- BP-T05 remove playerindex row then request final reward.
- BP-T06 Event Manager start/stop season with configured ID >1 and inspect active ID.
- BP-T07 process boot before Event Manager population; inspect uninitialized array values.
- BP-T08 season rollover/reload with existing mission/playerindex rows.


### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no Ticket runtime test executed.
- TICKET-T01..T07 map one-to-one to BUG-TICKET-001..007.
- TICKET-T07 is the normal-path UI validation target.
- TICKET-T01..T06 remain isolated/adversarial tests requiring controlled conditions.
- Overall first live runtime gate remains DUNGEON-T09; Ticket does not preempt it.
