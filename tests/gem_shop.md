# Gem Shop — Test Plan

## GEM-001 — Missing-row configuration
1. Prepare Gem Shop data with one required row absent.
2. Start/load the server and open Gem Shop on a fresh character.
3. **Expected after fix:** startup/config load rejects the invalid table cleanly, or Gem Shop open returns a safe error; no empty-vector random selection occurs.
4. Repeat with a row whose only item references an invalid item vnum.
5. Verify the same safe behavior.
6. Restore a complete valid table and verify first open and refresh populate every offer slot normally.


## GEM-002 — Full-inventory BUY cleanup
1. Fill the target inventory so the offered Gem Shop item has no valid destination.
2. Attempt to BUY the item repeatedly.
3. **Expected after fix:** no Gem is deducted, no item is placed, and no ownerless item object/ID remains registered in ITEM_MANAGER.
4. Free one valid slot and buy again.
5. Verify exactly one item is created, placed, saved and the offer slot becomes consumed.


## GEM-003 — BUY crash-order persistence
1. Start with a known Gem balance and an available offer.
2. Buy the offer.
3. Inject a process stop after the purchased item has been flushed but before player CHARACTER data is saved.
4. Restart/login.
5. **Expected after fix:** either the item and debit/consumed-slot all persist, or none of them persist; never retain the item with restored Gem balance/offer availability.

## GEM-004 — REFRESH / ADD crash-order persistence
1. Give exactly one refresh consumable and record current Gem Shop offers/timer.
2. Trigger REFRESH and stop the process after item consumption but before player state persistence.
3. Restart/login.
4. **Expected after fix:** consumable removal and refreshed state commit together, or both roll back/reconcile.
5. Repeat for slot ADD/unlock with the exact required unlock-item count.
6. **Expected after fix:** the unlock item cannot be permanently consumed while the slot returns to locked.
