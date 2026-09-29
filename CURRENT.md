# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static Mapping In Progress  
**Status:** MESSENGER / FRIEND / BLOCK / 6 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Messenger / Friend / Block  
**Latest completed subsystem:** Mining / Pickaxe  
**System:** `systems/messenger.md`  
**Bugs:** `bugs/messenger.md`  
**Tests:** `tests/messenger.md`  
**Effective completed/readiness-covered subsystems:** 31  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Previous closure
Mining / Pickaxe is **STATIC COMPLETE** with `BUG-MIN-001..007` and deferred `MIN-T01..MIN-T07`.

## Active Messenger findings
- `BUG-MSG-001` — block-add friend validation duplicates `IsBlocked`; friend+block coexistence is allowed and the intended already-blocked branch is shadowed.
- `BUG-MSG-002` — P2P/remote whispers bypass messenger block checks because enforcement requires local `pkChr`.
- `BUG-MSG-003` — asynchronous messenger DB callbacks can repopulate relation state and emit presence after logout.
- `BUG-MSG-004` — client `OnBlockLogin` invokes `OnLogout`, rendering online blocked users offline.
- `BUG-MSG-005` — client `Destroy()` clears friend/guild state but leaves block and GM caches across session teardown.
- `BUG-MSG-006` — pending friend authorization tokens have no server timeout/logout cleanup and remain consumable later.

## Exact continuation cursor
1. Audit friend/block remove-all and inverse relation symmetry.
2. Audit P2P presence consistency and non-whisper block enforcement surfaces.
3. Audit messenger client packet-size/state handling and GM cache lifecycle.
4. Close reconnect/channel-change boundaries and remaining candidates.
5. Keep all runtime tests deferred; global first future live gate remains `DUNGEON-T09`.
