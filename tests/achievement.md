# achievement — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

## Achievement System — static-complete runtime matrix

### ACH-T01 — remote Achievement Shop
Use an isolated test client to send Achievement OPEN_SHOP while far from the intended NPC / on another allowed map.
Verify shop 104 opens and a purchase can complete.
Covers BUG-ACH-004.

### ACH-T02 — shop point rollback after crash
1. record Achievement points;
2. buy a shop-104 item;
3. confirm item row is durable after `FlushDelayedSave`;
4. hard-kill GAME before clean logout;
5. relog.

Bug indicator: item remains while pre-purchase Achievement points return.
Covers BUG-ACH-005.

### ACH-T03 — missing task-family progression
Trigger legitimate gameplay for:
- summon/use mount,
- normal shop spending,
- private-shop-search spending,
- safebox money withdraw (only where the relevant feature is enabled).
Observe configured task IDs for types 5/19/20/22.

Bug indicator: task state never changes.
Covers BUG-ACH-006.

### ACH-T04 — EXPLORE transition
Login outside target map, then warp normally into a map referenced by TYPE_EXPLORE without relogging.
Check task state/UI immediately, then relog on the map.

Bug indicator: progress appears only after relog.
Covers BUG-ACH-007.

### ACH-T05 — repeated GM force finish
On disposable data, as GM_IMPLEMENTOR run `force_finish_achievement` twice for an achievement with a visible point/title reward.

Bug indicator: reward applies twice.
Covers BUG-ACH-008.

### ACH-T06 — repeated quest force finish
In isolated dev quest only, invoke `pc.finish_achievement(id)` twice for the same current character.
Verify repeated reward.
Covers BUG-ACH-008 trusted-script surface.

### ACH-T07 — reward/crash completion replay
Complete a normal achievement, allow item reward to become durable, then hard-kill GAME before Achievement OnLogout.
Relog and repeat the final trigger.
Covers BUG-ACH-001.

### ACH-T08 — stale task migration
Copy a disposable DB state, then remove/renumber one task from a max_value achievement XML and restart.
Load the player under ASan/debug.
Covers BUG-ACH-003.


### ACH-T09 — achievement cache rebuild atomicity
Use only a disposable DB snapshot. Trigger an Achievement-state flush while simulating/intercepting a DB failure after the destructive clear phase but before the full achievement/task rebuild completes. Restart and reload the same player.

Bug indicator: achievement/task rows are missing or only partially rebuilt.
Covers BUG-ACH-002.


### Readiness consolidation — 2026-09-28
- Documentation-only pass completed; no Achievement runtime test executed.
- ACH-T01 -> BUG-ACH-004.
- ACH-T02 -> BUG-ACH-005.
- ACH-T03 -> BUG-ACH-006.
- ACH-T04 -> BUG-ACH-007.
- ACH-T05/ACH-T06 -> BUG-ACH-008 through privileged/trusted surfaces.
- ACH-T07 -> BUG-ACH-001.
- ACH-T08 -> BUG-ACH-003.
- ACH-T09 added to cover previously untested BUG-ACH-002.
- Primary legitimate normal-path candidates: ACH-T03 and ACH-T04.
- Overall first live runtime gate remains DUNGEON-T09.


### ACH-T03/T04 preflight — 2026-09-29
Current config/source reverified. Current XML contains TYPE_SUMMON_MOUNT=16, TYPE_SPEND_SEARCH_SHOP=5, TYPE_SPEND_SHOP=3, TYPE_WITHDRAW=5 and TYPE_EXPLORE=29 task rows. No mapped gameplay caller exists for the four BUG-ACH-006 families. OnLogin explicitly calls OnVisitMap, while mapped normal warp/map-transition paths do not. Canonical handoffs: `../ACH_T03_HANDOFF.md`, `../ACH_T04_HANDOFF.md`.
