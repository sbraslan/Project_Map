# Costume / Appearance / ChangeLook — Deferred Tests

**Phase:** Detection / Mapping Only  
**Execution:** LOCKED / NOT RUN

## LOOK-T01 — Right-slot-before-left server safety
Covers `BUG-LOOK-001`.

Future isolated/modified-client test:
1. open ChangeLook window normally;
2. send a valid ChangeLook ITEM_CHECK_IN targeting RIGHT slot before LEFT is populated;
3. observe server/core behavior.

Expected from static code: null dereference in `CheckOtherItem` before the null guard.

Safety class: **Stage B isolated / modified client**.
Do not run on production.

## LOOK-T02 — Quest mount accepts non-costume material
Covers `BUG-LOOK-002`.

Future ordinary/UI-first validation:
1. open mount ChangeLook mode;
2. place one of 50051..50053 in the left slot;
3. attempt a clearly non-costume inventory item in the right slot;
4. verify whether UI accepts it;
5. do not press Accept in a valuable/live inventory until isolated test data is used.

Static client/server predicate predicts right-slot acceptance.

Second isolated step, only after safe disposable test data:
- Accept and verify stored ChangeLook VNUM / consumed material.

Safety class:
- eligibility observation: ordinary/current UI;
- commit/consumption: isolated disposable data.

No test executed.


## LOOK-T03 — Warp with ChangeLook window active
Covers `BUG-LOOK-003`.

Future controlled test:
1. open ChangeLook normally;
2. check in at least the left target;
3. invoke a normal server-authorized same-core warp path;
4. observe whether warp succeeds;
5. after arrival inspect ChangeLook/UI state and attempt ordinary inventory movement;
6. capture whether CANCEL/reopen/reconnect is needed to restore item handling.

Static prediction:
- `CanWarp()` does not reject `W_CHANGELOOK`;
- `WarpSet()` does not clear `m_pkTransmutation`.

Safety class: **Stage A/B controlled normal-flow**, depending on available same-core warp path.

## LOOK-T04 — Sealed material server revalidation
Covers `BUG-LOOK-004`.

Future isolated modified-client test:
1. use a disposable sealed/bound item that is otherwise type/subtype/anti-flag compatible as the right material;
2. bypass the official client UI seal check and send normal ChangeLook check-in;
3. verify server right-slot acceptance;
4. do not commit against valuable data;
5. in an isolated disposable setup, optionally Accept and verify consumption.

Static prediction: server accepts because no `IsSealed()` validation exists in the transmutation path.

Safety class: **Stage B isolated / modified client**.

No test executed.


## LOOK-T05 — LEFT replacement compatibility bypass
Covers `BUG-LOOK-005`.

Future isolated modified-client sequence:
1. open item ChangeLook;
2. check in target A;
3. check in compatible material B;
4. send LEFT checkout while RIGHT remains;
5. check in target C of another allowed ChangeLook class/subtype;
6. verify server accepts C without revalidating B;
7. only with disposable data, Accept and inspect stored transmutation VNUM.

Static prediction: final incompatible pair is accepted.

Safety class: **Stage B isolated / modified client**.

## LOOK-T06 — Checked-in real-time item expires
Covers `BUG-LOOK-006`.

Future sanitizer/debug-only test:
1. prepare a disposable ChangeLook-eligible item with a very short real-time expiry;
2. check it into LEFT or RIGHT;
3. keep the window open through expiry;
4. confirm item destruction;
5. trigger checkout/accept only under ASan/debug instrumentation.

Static prediction: CTransmutation retains a stale raw pointer after item deletion.

Safety class: **Stage B/C crash / lifetime / sanitizer**.
Do not run on production.

No test executed.


## LOOK-T07 — ChangeLook + Acce shared-item lifetime
Covers `BUG-LOOK-007`.

Future isolated debug sequence:
1. open ChangeLook and check in a disposable eligible weapon/body armor;
2. open Acce absorption without closing ChangeLook;
3. use the same item as Acce material with a disposable sash;
4. allow Acce to consume the material;
5. under ASan/debug only, trigger ChangeLook checkout/accept.

Static prediction:
- Acce and ChangeLook can coexist;
- Acce deletes the material;
- ChangeLook retains the stale raw pointer.

Safety class: **Stage B/C cross-window lifetime / sanitizer**.
Do not run on production.

No test executed.


## Runtime-readiness classification — 2026-09-28

**Documentation:** READY  
**Execution:** LOCKED / NOT RUN

Coverage:
- LOOK-T01 -> BUG-LOOK-001 — Stage B modified-client/null-deref safety.
- LOOK-T02 -> BUG-LOOK-002 — Stage A ordinary UI eligibility observation; Stage B disposable commit.
- LOOK-T03 -> BUG-LOOK-003 — Stage A/B controlled normal-flow warp/state observation.
- LOOK-T04 -> BUG-LOOK-004 — Stage B modified-client sealed-material validation.
- LOOK-T05 -> BUG-LOOK-005 — Stage B modified-client state-invariant bypass.
- LOOK-T06 -> BUG-LOOK-006 — Stage C crash/lifetime/sanitizer.
- LOOK-T07 -> BUG-LOOK-007 — Stage C cross-window lifetime/sanitizer.

Primary non-destructive future observations are LOOK-T02 eligibility-only and LOOK-T03.
They do **not** supersede the global first live gate: DUNGEON-T10 remains first.

No test executed.
