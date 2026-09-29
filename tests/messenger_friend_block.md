# Messenger / Friend / Block — Future Test Plan

**Execution:** LOCKED

## MSG-T01 — friend then block by VID
Precondition: A and B are friends, neither blocks the other.
Action: A attempts block-by-VID on B.
Expected after fix: request rejected as friend relation exists.
Current static prediction: block is accepted and friend+block coexist.

## MSG-T02 — friend then block by name
Same as MSG-T01 using name-based path.

Do not execute until the global runtime phase is explicitly opened.


## MSG-T03 — blocked target relog persistence
Precondition: B blocks A; B remains online.
Action: A logs out, then logs back in while B stays online.
Expected: `IsBlocked(B,A)` remains true and block enforcement remains active.
Current static prediction: A logout removes A from `m_BlockRelation[B]`; A relog does not restore B->A, so enforcement becomes false while the DB row remains.

Do not execute until runtime phase is explicitly opened.
