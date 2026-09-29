# Switchbot — Deferred Runtime Tests

> Lifecycle authority: `../MAP_STATE.json`. This file contains canonical `SWB-*` test evidence only.
> Runtime remains locked until the project phase is explicitly changed.

### SWB-T01 — cross-core object ownership
Repeatedly transfer a Switchbot-using disposable character between two GAME cores/channels while observing source-core object/memory ownership with debug/LSan/RSS instrumentation.

Covers BUG-SWITCHBOT-001.

### SWB-T02 — manager reinitialize ownership
In an isolated debug lifecycle test, create multiple Switchbot manager entries, exercise a controlled manager Initialize/destruction lifecycle, and verify owned Switchbot objects/events are released rather than only removed from the map.

Covers BUG-SWITCHBOT-002.

### SWB-T03 — logout manager/event cleanup
Use an active Switchbot slot, perform a normal logout, and inspect whether the PID manager entry and periodic event are removed.

Covers BUG-SWITCHBOT-003.

### SWB-T04 — empty/stale START validation
With a controlled test client, request START for an empty or stale registered slot and verify the server rejects it without leaving an active periodic event.

Covers BUG-SWITCHBOT-004.

### SWB-T05 — last-active-item unregister cleanup
Remove/unregister the last active Switchbot item through a legitimate lifecycle path that reaches UnregisterItem, then verify the switching event stops when no active slots remain.

Covers BUG-SWITCHBOT-005.

### SWB-T06 — UPDATE_ITEM VNUM width observation
Use an item VNUM above 255 in an isolated debug session and capture the Switchbot update packet versus the normal item state.

Validates OBS-SWITCHBOT-001 only; observation status is unchanged.

### SWB-T07 — client slot boundary observation
Call the client Start/Stop binding at exactly SWITCHBOT_SLOT_COUNT in an isolated debug client and verify the client-local guard behavior while confirming the server rejects the out-of-range slot.

Validates OBS-SWITCHBOT-002 only; observation status is unchanged.

### Readiness consolidation — 2026-09-28
- No Switchbot runtime test was executed.
- SWB-T01..SWB-T05 cover BUG-SWITCHBOT-001..005 one-to-one.
- SWB-T06/SWB-T07 cover OBS-SWITCHBOT-001/002 without promoting them to verified bugs.
- The duplicated legacy SWITCHBOT-T01..T06 identifiers above are non-canonical historical notes.
- Primary legitimate runtime candidates: SWB-T01 and SWB-T03; both need monitoring/instrumentation rather than crafted packet input.
- Overall first live runtime gate remains DUNGEON-T09.
