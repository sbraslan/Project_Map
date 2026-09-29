# Fishing Renewal — Deferred Runtime Tests

**Status:** READY FOR DOCUMENTATION / EXECUTION LOCKED  
**Global first live gate:** `DUNGEON-T09`

## FISH-T01 — second-table bounds/reward validation
Goal: validate `BUG-FISH-001` under a current second-family rod (+11..+20 or Carbon).

Preconditions:
- runtime phase explicitly unlocked later;
- controlled test character, rod and bait;
- preferably ASan/UBSan or equivalent bounds instrumentation.

Observe repeated renewed fishing starts and selected fish VNUMs. Confirm whether indices 5/6 trigger sanitizer evidence and/or unintended reward selection.

Do not run while execution lock is active.

## FISH-T02 — temporary probe-item leak
Goal: validate `BUG-FISH-002`.

Preconditions:
- runtime phase explicitly unlocked later;
- item-manager object/VID-map count or allocator instrumentation.

Repeatedly start/stop renewed fishing without receiving item 50187. Compare item-manager/allocated-item counts before and after.

Do not run while execution lock is active.


## FISH-T03 — client-declared catch bypass
Goal: validate `BUG-FISH-003`.

With runtime explicitly unlocked, use a controlled test client/harness during an active renewed fishing event. Send valid CATCH packets at the accepted timing interval without performing the UI target hit. Verify server catch count reaches `FISHING_NEED_CATCH` and enters the normal final reward roll.

Do not run while execution lock is active.


## FISH-T04 — movement during renewed fishing
Goal: validate `BUG-FISH-004`.

With runtime explicitly unlocked, begin renewed fishing normally, then send/perform movement while leaving the fishing event active. Verify the server accepts the move and the same event continues processing catches without revalidating the original fishing position.

Do not run while execution lock is active.

## FISH-T05 — Carbon rod special bonus
Goal: validate `BUG-FISH-005`.

With runtime explicitly unlocked, compare Carbon rod 27591 final catch chance behavior against the intended doubled branch and a normal rod with equivalent Value0. Instrumentation/logging is preferred to avoid statistical ambiguity.

Do not run while execution lock is active.


## FISH-T06 — death/warp lifecycle cleanup
Goal: validate `BUG-FISH-006`.

Death branch:
- with runtime explicitly unlocked, reach the required renewed catch count;
- trigger death before the next fishing-event decision tick;
- verify whether the event still resolves the final reward path while the character is dead.

Warp branch:
- start renewed fishing normally;
- trigger a same-character/same-process warp before event completion;
- verify `m_pkFishingNewEvent` survives and whether catch/event processing continues on the destination map without water-position revalidation.

Do not run while execution lock is active.
