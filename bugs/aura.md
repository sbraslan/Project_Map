# Aura System — Static Bug Registry

## BUG-AURA-001 — Aura distance check is structurally bypassed after the window opens

**Status:** VERIFIED STATIC

Server:
- `CHARACTER::IsAuraRefineWindowCanRefine()`
- `AuraRefineWindowCheckIn()`
- `AuraRefineWindowCheckOut()`
- `AuraRefineWindowAccept()`

`IsAuraRefineWindowCanRefine()` starts by calling `CanHandleItem()`.

The generic item gate returns false whenever:
```
IsAuraRefineWindowOpen() || nullptr != GetAuraRefineWindowOpener()
```

Consequently the Aura-specific permission function normally returns false immediately while its own window is legitimately open and never reaches its opener-distance comparison.

Each Aura transaction handler then catches that false result but explicitly continues if:
```
IsAuraRefineWindowOpen() && GetAuraRefineWindowOpener() != nullptr
```

So the fallback condition that is true for a normal open Aura window suppresses the failed permission result.

**Impact:** after opening Aura within the initial allowed distance, the player can move away while retaining the window and the mapped check-in/check-out/final accept paths no longer enforce `AURA_REFINE_MAX_DISTANCE`. Final destructive Aura operations can therefore remain remotely callable as long as the Aura state/opener survives.

**Runtime:** controlled normal-flow distance test only after phase unlock; see `AURA-T01`.
