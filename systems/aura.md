# Aura System — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.

## Entry points mapped so far

Server implementation:
- `Project_ServerSRC/game/src/char_aura.cpp`
- `CHARACTER::OpenAuraRefineWindow()`
- `CHARACTER::AuraRefineWindowCheckIn()`
- `CHARACTER::AuraRefineWindowCheckOut()`
- `CHARACTER::AuraRefineWindowAccept()`
- `CHARACTER::IsAuraRefineWindowCanRefine()`

State:
- `m_pointsInstant.m_pAuraRefineWindowOpener`
- `m_bAuraRefineWindowType`
- `m_bAuraRefineWindowOpen`
- `m_pAuraRefineWindowItemSlot[]`
- `W_AURA` under ENABLE_CHECK_WINDOW_RENEWAL

Window types currently observed:
- ABSORB
- GROWTH
- EVOLVE

The server stores Aura checked-in items as `TItemPos` and locks the real inventory item while it is present in the window.

## First verified static finding

### Aura distance authorization is bypassed after opening
`IsAuraRefineWindowCanRefine()` intends to enforce:
1. global item-handling eligibility;
2. Aura window open;
3. non-null opener;
4. distance < `AURA_REFINE_MAX_DISTANCE`.

But it begins with:
```
if (!CanHandleItem())
    return false;
```

The generic `CanHandleItem()` itself returns false while Aura is open or has an opener.

Therefore, during the exact state in which Aura operations are supposed to execute, `IsAuraRefineWindowCanRefine()` returns false before reaching the distance check.

Check-in, check-out and final accept all handle that false result like this:
- if Aura is open and opener is non-null, continue anyway;
- otherwise return.

That fallback converts the failed permission check into an allow path and bypasses the intended distance check.

See `BUG-AURA-001`.

## Cross-system observation
Aura and Dragon Soul do not share one universal opener mutex. Dragon Soul closure found no duplicate DS-specific destructive alias because Aura slots accept Aura-costume / armor / Aura-resource inputs rather than ITEM_DS. Aura-side overlap still requires its own audit.

## Exact next work
1. map client -> packet -> server Aura open/check-in/check-out/accept contract;
2. trace ABSORB item copy and material destruction lifetime;
3. trace GROWTH refine table, material counts, EXP/socket arithmetic and output persistence;
4. trace EVOLVE success/failure item lifecycle;
5. audit booster/eraser and absorption-rate arithmetic;
6. audit warp/disconnect/close cleanup and locked-item recovery;
7. audit opener lifetime and cross-window coexistence;
8. map Aura visual/proto/client persistence surfaces;
9. create additional bug/test records only from verified reachable paths.

No Aura runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.
