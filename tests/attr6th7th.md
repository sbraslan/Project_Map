# 6th/7th Attribute — Deferred Runtime Tests

**Status:** DOCUMENTED / EXECUTION LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## ATTR67-T01 — direct action without OPEN
In an isolated future runtime:
1. ensure Attr67 window state is closed;
2. send GET_FRAGMENT/SKILLBOOK_COMB/ADD directly;
3. inspect `W_ATTR_6TH_7TH` and mutation results.

Expected signature: action executes without OPEN/window state.

Covers `BUG-ATTR67-001`.

## ATTR67-T02 — duplicate combination cells
1. place one disposable one-count skillbook in a known cell;
2. construct all ten bCell entries with that same cell;
3. send SKILLBOOK_COMB;
4. inspect consumed items, Yang and reward.

Expected signature: only one book is consumed while reward is granted.

Covers `BUG-ATTR67-002`.

## ATTR67-T03 — Exchange/Attr67 lifetime collision
1. offer a disposable Attr67-relevant item in Exchange;
2. keep Exchange open;
3. trigger crafted Attr67 consumption against that item;
4. cancel/complete Exchange under instrumentation.

Expected signature: Exchange retains a pointer after item destruction; cancellation reaches a stale pointer.

Covers `BUG-ATTR67-003`.

## ATTR67-T04 — stale selected target
1. send GET_FRAGMENT for an eligible disposable target;
2. destroy/move the target through an allowed ordinary path while W_ATTR was never opened;
3. place another item in the later ADD packet cell as needed;
4. issue ADD under ASan/debug instrumentation.

Expected signature: `m_AttrItemAdded` remains stale and ADD dereferences it.

Covers `BUG-ATTR67-004`.

## ATTR67-T05 — fragment loss on invalid additive
1. choose valid target and fragment count;
2. submit ADD with valid fragments but an invalid/stale additive slot;
3. compare fragment count and NPC_STORAGE/timer state.

Expected signature: fragments are removed although the operation aborts.

Covers `BUG-ATTR67-005`.

## ATTR67-T06 — skillbook RNG endpoint
Use deterministic/instrumented RNG or a high-volume isolated sample for each configured class/group range.

Expected signature: each configured upper endpoint is never generated.

Covers `BUG-ATTR67-006`.

## ATTR67-T07 — legitimate deployment entry/retrieval
Boot the exact tracked quest deployment without adding new quests and inspect available NPC/item quest entry paths and registered quest state.

Expected signature: no normal quest flow calls Attr67 open/retrieval bindings.

Covers `BUG-ATTR67-007`.

## ATTR67-T08 — high special-inventory slot encoding
1. place skillbooks in special-inventory global slots 255, 256 and a high slot near 359;
2. capture the client packet and server-decoded bCell values;
3. compare original/global positions.

Expected signature: positions above 255 wrap/truncate.

Covers `BUG-ATTR67-008`.

Do not run any ATTR67 test while the global execution lock is active.
