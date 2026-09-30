# Snake Lair / Queen Nethis — Bug Registry

**Status:** STATIC MAPPING OPEN / 1 VERIFIED BUG
**Execution:** LOCKED / NOT RUN

## BUG-SNK-001 — CSnkMap constructor reads uninitialized event pointers and may cancel garbage addresses

**Class:** object lifetime / undefined behavior

### Proof
- `CSnkMap` owns raw event pointers `e_SpawnEvent`, `e_pEndEvent` and `e_pSkillEvent` with no in-class initialization.
- `CSnk::Access()` constructs a new instance via `M2_NEW CSnkMap(lMapIndex)`.
- At the top of `CSnkMap::CSnkMap`, each event member is compared against `nullptr` before any assignment and may be passed to `event_cancel`.
- The constructor then calls `SetDungeonStep(1)`, which reads `e_SpawnEvent` again, and later `Start()` reads `e_pEndEvent`.
- The members are assigned `nullptr` only after those initial reads.

### Consequence
Private Snake instance creation has undefined behavior at construction time. Depending on the indeterminate pointer contents, it can appear to work or can attempt to cancel an invalid event pointer, causing memory corruption/crash during dungeon creation.

### Deferred validation
`SNK-T01`.

