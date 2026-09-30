# Snake Lair / Queen Nethis — Static System Map

**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS
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
