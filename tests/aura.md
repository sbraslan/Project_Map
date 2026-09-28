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


## AURA-T04 — Aura SET_ITEM packet initialization
Covers `BUG-AURA-004`.

Future isolated debug test:
1. instrument or capture one Aura SET_ITEM packet in a controlled debug environment;
2. compare every serialized `TItemData` field against the source item and explicit zero/default expectations;
3. repeat several times to detect nondeterministic bytes in seal/transmutation/basic/element/set metadata;
4. inspect the client-side stored Aura `TItemData` for fields the receive path does not initialize.

Static prediction:
members not explicitly assigned by the Aura server/client packet paths contain indeterminate data.

Safety class: **Stage C debug / sanitizer / packet-observation only**.

## AURA-T05 — Yohara random apply persistence across successful EVOLVE
Covers `BUG-AURA-005`.

Future isolated disposable-item test:
1. prepare an Aura that has a known absorbed Yohara random apply;
2. record classic attributes and Yohara random-apply array;
3. perform one successful grade evolution in an isolated test environment;
4. compare the new Aura's sockets, classic attributes and Yohara random applies.

Static prediction:
sockets and classic attributes survive, but the Yohara random-apply array is not copied to the new item.

Safety class: **Stage B/C destructive data-integrity / disposable items only**.


## AURA-T06 — Terminal Radiant EVOLVE check-in arithmetic
Covers `BUG-AURA-006`.

Future isolated modified-client/debug test:
1. prepare a legitimate grade-6 Radiant Aura at level 250 / exp 0;
2. open the EVOLVE Aura window normally;
3. bypass the client-side `curLevel >= AURA_MAX_LEVEL` rejection and submit the Radiant Aura to MAIN;
4. instrument `__GetAuraRefineInfo()` and the outgoing current-info preview byte;
5. stop before any destructive follow-up.

Static prediction:
the server accepts MAIN check-in, resolves the terminal table row with `NEED_EXP=0`, and evaluates the current EXP percentage with a zero denominator before the later Accept-time Radiant rejection.

Safety class: **Stage B/C modified-client + arithmetic instrumentation**.
No production execution.
