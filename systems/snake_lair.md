# Snake Lair / Queen Nethis — Static System Map

**Status:** STATIC MAPPING OPEN / 1 VERIFIED BUG
**Mode:** detection / mapping only
**Execution:** LOCKED / NOT RUN

## Canonical scope
This node owns the enabled Queen Nethis / Snake Lair system:
- `ENABLE_QUEEN_NETHIS`;
- `SnakeLair::CSnk` and `CSnkMap`;
- portal entry, private-map ownership and party binding;
- floor progression, pillar/blacksmith/statue/item mechanics;
- Sungma dungeon attributes;
- kill, party-leave, input and item interaction hooks.

White Dragon / Alastor and DawnMist / Temple of Ochao are separate canonical families.

## Live deployment proof
- The feature macro is enabled under `ENABLE_YOHARA_SYSTEM`.
- Character NPC-click handling directly calls `SnakeLair::CSnk::Start()` for the Snake portal.
- Character combat, input and party code call SnakeLair lifecycle methods.
- SnakeLair Lua functions are registered.
- Relevant server and map/data assets are tracked.

## Audit cursor
1. Map portal access, party requirements, private-map creation and registration.
2. Audit floor/state progression, timer/event ownership and item-based transitions.
3. Audit disconnect, party-leave, kick/exit and destruction ordering.
4. Audit Sungma calculations, boss/debuff logic and client/server parity.
5. Close deployment/data parity and promote only source-proven defects.


## Cursor 1 finding — CSnkMap reads uninitialized event pointers during construction
`CSnk::Access()` creates each private instance with `M2_NEW CSnkMap(lMapIndex)`.

The `CSnkMap` class declares `e_SpawnEvent`, `e_pEndEvent` and `e_pSkillEvent` as raw pointer members without in-class initializers. The constructor immediately does:
- `if (e_SpawnEvent != nullptr) event_cancel(&e_SpawnEvent)`;
- the same for `e_pEndEvent` and `e_pSkillEvent`;
- then calls `SetDungeonStep(1)`, which reads `e_SpawnEvent` again;
- later `Start()` reads `e_pEndEvent` again.

No constructor initializer establishes these members before the reads. Under the normal C++ object-allocation semantics used by this construction path, these pointer values are indeterminate. A non-null garbage value can reach `event_cancel()`.

Promoted as `BUG-SNK-001`.

## Cursor 1 checkpoint — entry / registration ownership
- `SnakeLair.Access()` is the private-map creation API. It requires a party and party leader, creates a private copy of `MAP_SNAKE_TEMPLE_02`, registers `CSnkMap*` by private map index, and warps only same-map party members.
- The private instance constructor spawns portal VNUM 4022; clicking that portal is wired directly in `CHARACTER::OnClick` to `CSnk::Start()`, which binds the current party and starts floor 2.
- The tracked Game map spawns NPC 20807 on Snake Temple 01, but the pinned quest source/object tree does not expose a tracked `SnakeLair.Access()` caller or a 20807 quest-object binding. Full outer-entry requirements (level/item/cooldown) are therefore a deployment/source-completeness gap and are not invented from C++.
- `Access()` has a recall-position failure branch after private-map allocation/registration that returns success without immediate cleanup. The pinned Snake Temple 02 has a valid Town recall entry, so this branch is not promoted as an active snapshot defect.
- Same-map party collection is consistent with the actual warp set; off-map party members are neither collected nor warped.

### Next cursor
Floor/state progression, timer/event ownership and item-based transitions.
