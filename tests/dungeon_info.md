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
- DUNGEON-T10 load the current 9-dungeon config and open the UI; verify no list buttons are created despite GetCount()>0; covers BUG-DUNGEON-010.
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
