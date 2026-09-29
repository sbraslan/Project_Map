# battle pass — Runtime Tests

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

### Battle Pass
- BP-T01 complete mission, then invoke any reachable SetExt caller again at threshold; check duplicate reward.
- BP-T02 capture MISSION_UPDATE packets; verify bMissionType nondeterminism/mismatch.
- BP-T03 long season/config name against RequestOpen under ASan.
- BP-T04 open pass repeatedly to expose dangling season_name behavior under ASan/UBSan.
- BP-T05 remove playerindex row then request final reward.
- BP-T06 Event Manager start/stop season with configured ID >1 and inspect active ID.
- BP-T07 process boot before Event Manager population; inspect uninitialized array values.
- BP-T08 season rollover/reload with existing mission/playerindex rows.

## Battle Pass — static-complete runtime matrix

- BP-T09: complete a mission, wait until the reward item has left ITEM_MANAGER delayed-save queue / is visible in DB, then hard-kill GAME before clean logout; relog and verify whether mission can reward again. Covers BUG-BPASS-008.
- BP-T10: inject crash/failure after `battlepass_playerindex.battlepass_completed=1` UPDATE but before/during `BattlePassReward`; relog and retry final claim. Covers BUG-BPASS-009.
- BP-T11: repeated login/logout with many historical `battlepass_missions` rows under ASan/LSan or RSS monitoring; verify leaked `TPlayerExtBattlePassMission` objects. Covers BUG-BPASS-010.
- BP-T12: fresh CHARACTER first ranking request before any setter; instrument/read `m_dwLastExtBattlePassOpenRankingTime`. Covers BUG-BPASS-011.
- BP-T13: run `battlepass_set_mission` as GM_IMPLEMENTOR twice on an already completed mission with value >= target; verify duplicate mission reward. Covers BUG-BPASS-002.
- BP-T14: temporarily configure BattlePassID 2 in isolated environment, start through Event Manager and verify active ID remains 1. Covers BUG-BPASS-007.


### Readiness consolidation — 2026-09-28
- Documentation-only classification completed; no Battle Pass runtime test executed.
- BP-T01..BP-T14 are now treated as the canonical deferred Battle Pass validation set.
- Multiple tests may cover the same verified bug from different angles.
- BP-T08 remains a broad lifecycle/regression check rather than a unique bug mapping.
- BP-T02 is the primary normal-path observational candidate.
- Overall first live runtime gate remains DUNGEON-T09.


### BP-T02 preflight — 2026-09-29
Current source reverified: every TPacketGCExtBattlePassMissionUpdate construction path leaves bMissionType unset. Client forwards it unchanged. UI impact refined: HaveMission currently matches only missionIndex, so progress can still update; selected missions pass the undefined missionType into SetMissionInfo. Canonical handoff: `../BP_T02_HANDOFF.md`.
