# Aura System — Deferred Runtime Tests

**Execution:** LOCKED / NOT RUN

## AURA-T01 — Move out of Aura opener range after opening
Covers `BUG-AURA-001`.

Future controlled test:
1. open the Aura window normally while within `AURA_REFINE_MAX_DISTANCE`;
2. move beyond the allowed range without closing the window;
3. try a non-destructive check-in/check-out first;
4. only in an isolated disposable setup, test final accept with harmless disposable materials if needed;
5. record whether the server rejects for distance or continues.

Static prediction:
- `IsAuraRefineWindowCanRefine()` returns false early because `CanHandleItem()` sees Aura itself as open;
- caller fallback accepts the open-window/non-null-opener state;
- distance is not enforced.

Safety class: Stage A observation for check-in/check-out; destructive accept only isolated Stage B.

Global first future live gate remains `DUNGEON-T10`.
