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
