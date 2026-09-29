# ITEM-T01 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-ITEM-002`  
**Canonical test:** `ITEM-T01`

> Handoff only. It does not authorize runtime execution.

## Static basis

Destroy packet explicitly carries a count:
```cpp
typedef struct command_item_destroy
{
    uint8_t header;
    TItemPos Cell;
    uint32_t gold;
    uint8_t count;
} TPacketCGItemDestroy;
```

Client Python binding accepts and forwards that count:
```cpp
netSendItemDestroyPacket(...)
-> SendItemDestroyPacket(Cell, 0, count);
```

Current normal Binary flow also forwards the user-selected stack amount:
```python
dropCount = self.itemDropQuestionDialog.dropCount
...
self.__SendDestroyItemPacket(dropNumber, dropCount, INVENTORY)
...
m2netm2g.SendItemDestroyPacket(itemInvenType, itemVNum, itemCount)
```

Server receives:
```cpp
CInputMain::ItemDestroy
-> ch->RemoveItem(pinfo->Cell, pinfo->count);
```

But `CHARACTER::RemoveItem(const TItemPos&, uint8_t bCount)` never uses `bCount` to reduce the stack.

After validation it destroys/removes the entire item object:
```cpp
ITEM_MANAGER::Instance().RemoveItem(item, "DESTROY");
```
(or the Growth Pet special branch).

Thus a partial destroy request against a stack still deletes the full stack.

## Normal-flow reachability

Fresh client audit confirms this is not only a modified-client test.

The ordinary item-drop/destroy UI stores the selected amount in `dropCount` and passes that exact value to `SendItemDestroyPacket`.

Therefore a normal player can reach the mismatch when:
- a stackable destroyable item has count > 1;
- the UI allows choosing a partial amount;
- the destroy action is confirmed.

## Preconditions

1. Use a disposable normal character.
2. Use a stackable, destroyable, non-sealed, non-basic-blocked item.
3. Initial stack count should be clearly > requested destroy count, e.g. 50.
4. No exchange/lock/quest-running state.
5. Record item VNUM, item ID if available, slot and initial count.
6. Prefer ordinary UI only; no packet crafting required.

## Future live action

When runtime is explicitly unlocked and earlier gates are resolved:

1. Place a stack of e.g. 50 identical destroyable items in inventory.
2. Open the normal destroy/drop confirmation flow.
3. Select a partial amount, e.g. 1.
4. Confirm destroy normally.
5. Record resulting inventory slot/count.
6. Repeat on a fresh stack with another partial amount, e.g. 10.
7. If safe DB observability exists, confirm whether the original item row is deleted rather than count-adjusted.
8. Stop after evidence capture.

## Expected result

Despite packet `count=1` or `count=10`, the complete item stack object is removed.

Example:
- before: count 50
- requested destroy: 1
- safe expected semantic: remaining 49, or UI/API explicitly documents full-stack-only behavior
- current mapped behavior: entire 50-stack disappears

## Important adjacent bug

The same normal destroy path also contains `BUG-ITEM-001`:
after the item is destroyed/freed, current code calls:
```cpp
item->GetName()
```
inside the success ChatPacket.

ITEM-T01 is focused on count semantics only.
If a crash/UAF symptom occurs first, stop and classify ITEM-T01 as inconclusive while preserving evidence for ITEM-T02 / BUG-ITEM-001.

## Result classification

### REPRODUCED
Use when a partial normal destroy request removes the complete stack.

### NOT REPRODUCED
Use if a mapped-equivalent deployment reduces only the requested count or the client/server explicitly forces full-stack count before send.

### INCONCLUSIVE
Use when:
- the chosen item cannot be destroyed;
- UI does not permit partial count for that item;
- BUG-ITEM-001 crashes first;
- deployment differs materially;
- inventory state cannot be reliably observed.

## Evidence template

- Test: `ITEM-T01`
- Bug: `BUG-ITEM-002`
- Result:
- Date/time:
- Client/server build:
- Item VNUM/name:
- Initial item ID:
- Initial count:
- Requested destroy count:
- Resulting slot/count:
- DB row/count before:
- DB row/count after:
- Client message:
- Crash/UAF symptom:
- Deployment drift:
- Evidence reference:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
