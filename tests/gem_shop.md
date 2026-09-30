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
