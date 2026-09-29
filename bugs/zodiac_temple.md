# Zodiac Temple / 12ZI — Bug Registry

**Status:** STATIC MAPPING OPEN / 7 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

No runtime reproduction has been executed. Every promoted item below has a reachable static source chain or a deterministic C++ lifetime/state proof.

## BUG-ZOD-001 — reward/check-box helpers leak registered temporary item objects

**Class:** server resource leak / player-triggerable lifetime bug

### Proof
- `/cz_check_box <color> <index>` is registered as `GM_PLAYER` and calls `CHARACTER::ZTT_CHECK_BOX`.
- For every valid color/index, `ZTT_CHECK_BOX` calls `ITEM_MANAGER::CreateItem` twice for the row/column material prototypes before it checks whether the player owns the required materials.
- Both returned raw `LPITEM` values are never used and never passed to `DestroyItem` / `RemoveItem`.
- `CreateItem` assigns a VID/ID and inserts the object into `m_VIDMap` and `m_map_pkItemByID` when save skipping is false.
- Therefore even a failed material check leaves two ownerless registered item objects behind.
- `ZTT_REWARD` repeats the same pattern with one temporary reward object used only for inventory-space/type inspection and never destroys it.

### Consequence
A normal player command path can monotonically accumulate server-side item objects. Anti-command-flood limiting may bound rate, but it does not repair ownership or reclamation.

### Deferred validation
`ZOD-T01`.

## BUG-ZOD-002 — unpaired gold reward can create a free box through count-zero coercion

**Class:** reward authorization / item-count coercion

### Proof
- `/cz_reward 3` is a `GM_PLAYER` command.
- Type 3 is rejected only when **both** `zt_yellowreward_count` and `zt_greenreward_count` are zero; it does not require both to be positive and does not enforce `zt_can_get_goldreward`.
- If the counters are, for example, yellow=1 and green=0, `gold_box_count_give` becomes 0 and neither counter is reduced.
- The code still calls `AutoGiveItem(33028, 0)`.
- If no matching stack is already found, `AutoGiveItem` reaches `ITEM_MANAGER::CreateItem(..., 0)`.
- `CreateItem` clamps a stackable count with `MINMAX(1, count, ...)`; for a non-stackable item it also sets count to 1. Therefore a zero-count creation becomes one physical item.

### Consequence
A character with progress on only one color can obtain a gold reward box even though no yellow/green pair exists. Because subtracting zero leaves the asymmetric counters intact, the state can be reused; whether an immediately repeated call creates another item depends on whether a matching stack is still present in the inventories scanned by `AutoGiveItem`.

### Deferred validation
`ZOD-T02`.

## BUG-ZOD-003 — check-box replay uses arithmetic addition instead of idempotent bit setting

**Class:** reward-state integrity / replay

### Proof
- Client state is explicitly treated as a bitmask: `yellowmark & (1 << index)` / `greenmark & (1 << index)`.
- The client disables an already-marked cell, but `/cz_check_box` itself is player-accessible and the server does not reject an already-set bit.
- Server mutation is `old_value + (1 << index)`, not bitwise OR.
- Replaying the same index can therefore carry into a different bit (for example 0 + 1 + 1 = 2), while consuming the same row/column material pair twice.
- Completion is compared against decimal `1073741823` (all 30 low bits set), so arithmetic carries directly alter reward eligibility.

### Consequence
A modified/direct-command client can corrupt the progress mask and can substitute repeated lower-cell material consumption for higher bits instead of satisfying the intended per-cell material matrix.

### Deferred validation
`ZOD-T03`.

## BUG-ZOD-004 — revive authorization crosses independent Zodiac instances

**Class:** multiplayer instance-boundary authorization

### Proof
- `/revive <VID>` resolves the target through `CHARACTER_MANAGER::Find(VID)`, which uses the core-wide `m_map_pkChrByVID`.
- The only map authorization requires that both map indices satisfy `IsZiStageMapIndex`.
- `IsZiStageMapIndex` accepts the entire private-map range for base `MAP_12ZI_STAGE`; it does not require `ch->GetMapIndex() == target->GetMapIndex()`, the same `CZodiac*`, the same party, or proximity.
- On success the caller pays prisms and the remote target is revived in place.
- `/revivedialog <VID>` is even broader: it resolves any core-local dead PC and has no Zodiac-map check before returning the target's required prism count to the caller.

### Consequence
Given a valid target VID on the same game core, a player in one Zodiac instance can revive a dead player in another Zodiac instance. The dialog command also exposes/opens Zodiac revive UI for unrelated dead PCs outside the intended instance boundary.

### Deferred validation
`ZOD-T04`.

## BUG-ZOD-005 — Zodiac skill event handles are never initialized in CHARACTER::Initialize

**Class:** uninitialized pointer / undefined behavior

### Proof
- `char.h` declares raw `LPEVENT m_pkZodiacSkill1` through `m_pkZodiacSkill11` without in-class initializers.
- `CHARACTER` constructor calls `Initialize()`.
- The tracked `char.cpp` has no assignment to any `m_pkZodiacSkill1..11`; the Zodiac initialization block initializes `m_pkZodiac`, timestamps and dead count only.
- `ZodiacDamage` tests these members and may call `event_cancel(&m_pkZodiacSkillN)` before assigning a newly created event.

### Consequence
The first Zodiac skill scheduling on a fresh character object can branch on an indeterminate event pointer and attempt to cancel it, which is C++ undefined behavior and can manifest as a crash or memory corruption.

### Deferred validation
`ZOD-T05`.

## BUG-ZOD-006 — pending Zodiac skill events survive character destruction with raw CHARACTER pointers

**Class:** event lifetime / use-after-free

### Proof
- Zodiac skill event info stores raw `LPCHARACTER` pointers. Skills 1..10 retain the mob pointer; skill 11 retains both mob and victim player.
- Event callbacks dereference these pointers after a delay.
- `CHARACTER::Destroy()` explicitly cancels many character-owned events and clears generic `m_mapMobSkillEvent`, but it never cancels `m_pkZodiacSkill1..11`.
- The Zodiac skill events are separate fields, not entries in the generic cancellation map.
- After a mob is destroyed/private map is torn down, or after a skill-11 victim disconnects before the delayed callback, the event still owns only the stale raw address.

### Consequence
Delayed Zodiac combat callbacks can dereference freed character objects, creating a server crash/UAF surface during mob/map teardown and player disconnect timing.

### Deferred validation
`ZOD-T06`.

## BUG-ZOD-007 — CZodiacManager::Initialize falls off a bool function on non-channel-99 cores

**Class:** C++ undefined behavior / initialization

### Proof
- With `ENABLE_SERVERTIME_PORTAL_SPAWN`, `CZodiacManager::Initialize()` returns `true` only inside `if (g_bChannel == 99)`.
- There is no return statement for other channels.
- The manager is initialized from the game startup path on enabled builds; the caller currently discards the returned value.

### Consequence
Execution on a non-99 core reaches the end of a value-returning function without returning a value. The current caller does not consume the result, so practical impact is likely low, but the function still contains undefined C++ behavior and compiler/optimization sensitivity.

### Deferred validation
`ZOD-T07`.

## Active candidates under verification
- `DecMember` has an erase-while-iterating branch gated by `zodiac_disconnect_member_2`; deployment/default flag state is still being mapped.
- Zodiac deployment/entry ownership is still being mapped because `quest_list` has no visible Zodiac quest source while server-time portal spawn is enabled.
- Floor/event timer and private-map teardown ordering remains under review.
- 12ZI shop-limit and bead regeneration/persistence paths remain under review.
