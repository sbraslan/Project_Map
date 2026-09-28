# Party Match — Deferred Runtime Tests

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only
**Created:** 2026-09-28

Do not execute runtime, sanitizer, crafted-client, or fault-injection tests unless the user explicitly changes project phase.

### PMATCH-T01 — same-channel cross-core queue fragmentation
Use two disposable characters on the same channel but different game cores. Both search for the same Party Match dungeon whose required player count is two.

Observe:
- each core's local SearchMap;
- whether either side sees the other queued player;
- whether a match forms without relocating both characters to the same core.

Covers BUG-PMATCH-001.

### PMATCH-T02 — exchange-listed required item lifetime
Isolated/dev only.

1. Put an exact-stack required Party Match item into an active Exchange offer.
2. Start Party Match while Exchange remains active.
3. Complete a local match so required-item consumption occurs.
4. Trigger normal Exchange cancellation/character teardown afterward.
5. Observe item lifetime and CExchange raw-pointer handling under ASan/debug.

Covers BUG-PMATCH-002.

### PMATCH-T03 — FAIL_NO_ITEM stale minimap state
Normal UI path:
1. start Party Match successfully with required items;
2. while queued, remove/consume one required item through an allowed normal path;
3. trigger a later CheckPlayers by having another player search;
4. receive PARTY_MATCH_FAIL_NO_ITEM.

Verify:
- server queue entry is removed;
- main Party Match state resets;
- minimap Party Match icon visibility matches server state.

Covers BUG-PMATCH-003.

### PMATCH-T04 — duplicate SEARCH / HOLD robustness
Isolated alternate/modified client only. Send SEARCH again while already queued and observe:
- server StopSearching(...PARTY_MATCH_HOLD...);
- queue removal;
- client main state and minimap icon state.

This is robustness coverage for an unpromoted alternate-client desync. No bug ID is assigned.

### PMATCH-T05 — WarpSet result robustness
Isolated test environment only. Force or instrument a target-warp failure after local match completion and inspect whether:
- required items were already consumed;
- party was already created;
- success was already sent;
- recovery/rollback exists.

This validates the documented non-atomic WarpSet-result weakness only. No bug ID is assigned because the current configured targets are valid and normal reachability was not established.

## Readiness consolidation — 2026-09-28
- No Party Match runtime test was executed.
- PMATCH-T01 -> BUG-PMATCH-001.
- PMATCH-T02 -> BUG-PMATCH-002.
- PMATCH-T03 -> BUG-PMATCH-003.
- PMATCH-T04/PMATCH-T05 remain unpromoted robustness tests and must not be treated as verified bugs.
- Primary legitimate normal-path candidate: PMATCH-T03.
- PMATCH-T01 requires controlled multi-core placement but otherwise normal search flow.
- PMATCH-T02 requires a deliberately overlapping Exchange + Party Match state and debug/sanitizer observation.
- Party core ownership remains separate from Party Match.
- Overall first live runtime gate remains DUNGEON-T10.
