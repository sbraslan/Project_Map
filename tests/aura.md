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


## AURA-T02 — EVOLVE undersized checked-in material stack
Covers `BUG-AURA-002`.

Future isolated modified-client test:
1. prepare an Aura costume at a valid evolution boundary;
2. prepare the exact required material total split across at least two stacks;
3. make the Aura SUB stack smaller than the required count while total inventory count still meets the requirement;
4. submit the check-in/final accept sequence;
5. record the checked-in stack count, remaining other stacks, Yang change and evolution result.

Static prediction:
the global `CountSpecifyItem()` gate passes but consumption removes only the undersized checked-in SUB stack.

Safety class: **Stage B modified-client / disposable data**.

## AURA-T03 — Aura Eraser targeting a non-Aura socketed item
Covers `BUG-AURA-003`.

Future isolated modified-client test:
1. use a disposable unequipped non-Aura item with a known nonzero socket 2 value;
2. retain a before-state record of all sockets;
3. use an Aura Eraser with that item as the crafted destination;
4. record eraser consumption and target socket state.

Static prediction:
the server accepts the non-Aura destination because no COSTUME_AURA check exists and executes `SetSocket(2, 0)`.

Safety class: **Stage B destructive modified-client / disposable data only**.
Never use production or valuable items.
