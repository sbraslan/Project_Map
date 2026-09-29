# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Classic Pet System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 1 PROMOTED CLASSIC-PET BUG / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Classic Pet System  
**System:** `systems/pet.md`  
**Bugs:** `bugs/pet.md`  
**Tests:** `tests/pet.md`  
**Last completed subsystem:** Horse / Mount / Riding  
**Effective completed/readiness-covered subsystems:** 28  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Horse closure
Horse / Mount / Riding is STATIC COMPLETE:
- BUG-HORSE-001..003
- HORSE-T01..T03 deferred / not run.

## Classic Pet progress
Active feature state:
- `ENABLE_PET_SYSTEM` enabled;
- `PET_AUTO_PICKUP` enabled;
- `__PET_SYSTEM__` enabled;
- PetSystem and questlua_pet are built.

Promoted:
- `BUG-PET-001` — Bruce auto-pickup range calculation ignores Y distance.

Deferred test:
- `PET-T01`.

Open:
- Bruce raw pickup-item lifetime;
- PET_PAY expiry/death/login cleanup;
- CPetSystem event/actor lifetime;
- Lua pet.summon legacy-signature mismatch reachability;
- current PET_PAY family/race/client coverage;
- Achievement summon-time abnormal cleanup.

Do not execute `PET-T01`.
GitHub state is canonical.
