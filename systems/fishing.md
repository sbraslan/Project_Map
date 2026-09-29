# Fishing Renewal — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source/game repositories; only Project_Map is writable.  
**Feature gate:** `ENABLE_FISHING_RENEWAL`.

## Entry roots
Server:
- `game/src/char_fishing.cpp` — renewed fishing start/stop/catch/event/reward lifecycle.
- `game/src/fishing.cpp/.h` — fish tables and `GetFishCatchedVnum()`.
- `game/src/input_main.cpp::FishingNew` — `HEADER_CG_FISHING_NEW` dispatch.
- `game/src/packet.h::TPacketFishingNew`.
- `game/src/char_item.cpp::USE_BAIT` — bait -> rod socket2.
- `game/src/item_manager.cpp::CreateItem/DestroyItem` — temporary-item lifetime boundary.

Client:
- `Project_Binary/root/uifishing.py` — renewed fishing minigame UI and catch/fail actions.

Data:
- `Project_DumpProto/tr/item_names.txt` — current rod family names/VNUM presence.

## Normal renewed flow
Client starts renewed fishing
-> `HEADER_CG_FISHING_NEW / FISHING_SUBHEADER_NEW_START`
-> `CInputMain::FishingNew`
-> `CHARACTER::fishing_new_start()`
-> validates map/rod/bait/inventory
-> chooses fish through `fishing::GetFishCatchedVnum(..., second)`
-> starts `m_pkFishingNewEvent`
-> client fishing UI sends catch/fail packets
-> three valid catches call `fishing_catch_decision(itemVnum)`
-> chance roll -> AutoGiveItem on success.

Current rod-name deployment includes:
- 27400..27490 = Olta +1..+10;
- 27500..27590 = Olta +11..+20;
- 27591 = Karbon olta.

The code defines `second = false` only for VNUM 27400..27490, so 27500..27591 use the second fish tables.

## BUG-FISH-001 — second normal fish table can be indexed out of bounds
`aFishSecondTableNormal` contains 5 entries, but the normal second-table path returns:
`aFishSecondTableNormal[number(0, 6)]`.

For second-family rods, indices 5 or 6 are therefore outside the declared array.

Reachability is normal gameplay:
`FishingNew START -> fishing_new_start -> GetFishCatchedVnum(..., second=true)`.

Consequence: undefined read / incorrect selected fish VNUM is possible during ordinary renewed fishing with current +11..+20 / Carbon rod family. A crash is not claimed statically.

Promoted as `BUG-FISH-001`.

## BUG-FISH-002 — fishing start leaks a temporary item object
`fishing_new_start()` calls:
`ITEM_MANAGER::CreateItem(50187)`
only to pass the created object to `GetEmptyInventory(...)`.

The temporary item is never added to the character and no `RemoveItem/DestroyItem/M2_DESTROY_ITEM` call releases it on either the success path or the inventory-full return path.

`ITEM_MANAGER::CreateItem` allocates a `CItem`, assigns ID/VID, and registers it in the manager maps when `bSkipSave == false`; explicit `DestroyItem` removes those registrations and deletes the object.

Current item names contain VNUM 50187 (Çırak Sandığı I), so the probe item exists in tracked deployment.

Consequence: each renewed fishing start can leave an ownerless registered item allocated, causing cumulative item-manager/memory growth.

Promoted as `BUG-FISH-002`.

## Additional candidate
In `fishing_catch_decision`:
`if (dwVnum == 27591 && dwVnum >= 27400 && dwVnum <= 27490)`
is impossible because 27591 cannot also be <=27490. The exact intended Carbon-rod chance rule still needs semantic/data closure before promotion.

## Exact next work
1. map packet registration/size checks on client and server;
2. audit catch/fail packet trust, timing and replay boundaries;
3. close bait/POINT_FISHING_RARE arithmetic and uint8 probability underflow/overflow;
4. trace success reward, logging and Battle Pass/Achievement ownership;
5. audit stop/death/warp/logout/equipment-change cleanup;
6. audit rod refine lifecycle and current proto values;
7. promote only verified reachable findings.

No Fishing runtime test is authorized. Global first future live gate remains `DUNGEON-T09`.


## BUG-FISH-003 — server trusts client-declared successful minigame catches
The renewed client UI decides whether the moving fish is inside the target area. On a visual hit it sends `FISHING_SUBHEADER_NEW_CATCH`; on a miss it sends `...CATCH_FAILED`.

Server `CInputMain::FishingNew` forwards a CATCH directly to `CHARACTER::fishing_new_catch()`.

Server-side acceptance checks are only:
- a renewed fishing event exists;
- `GetLastCatchTime() <= get_global_time()`.

An accepted packet increments `m_bFishCatch`. Once the count reaches `FISHING_NEED_CATCH` (3), the periodic event calls `fishing_catch_decision(info->vnum)`.

The server does not reproduce/validate:
- fish position;
- target-circle position;
- client hit-test;
- mouse position;
- a server-issued challenge token/state proving that a visual hit occurred.

Therefore a modified client can submit one CATCH per accepted time interval and satisfy the three-hit minigame without performing the UI hit test.

Promoted as `BUG-FISH-003`.

## Packet boundary closure
`HEADER_CG_FISHING_NEW` is registered in `packet_info.cpp` as fixed-size `sizeof(TPacketFishingNew)`, so no separate variable-length packet-size defect is promoted here.


## BUG-FISH-004 — renewed fishing does not server-lock movement or revalidate fishing position
`fishing_new_start()` starts the renewed fishing event but does not set the character to `POS_FISHING` or otherwise install a movement lock.

`CHARACTER::CanMove()` does not check `m_pkFishingNewEvent`, and `CInputMain::Move` therefore accepts normal movement while the renewed event is active.

The periodic renewed fishing event checks only:
- character still exists;
- catch count;
- rod still equipped;
- timeout/fail counters.

It does not revalidate:
- current map/sectree fishing attribute;
- distance from the original fishing position;
- whether the player has moved away from water.

A modified client can therefore move during an active renewed fishing session while preserving the event and continue submitting catches.

Promoted as `BUG-FISH-004`.

## BUG-FISH-005 — Carbon rod special catch bonus branch is unreachable
Current deployment includes item VNUM `27591` = Carbon rod.

In `fishing_catch_decision()`, the intended special handling is:
`if (dwVnum == 27591 && dwVnum >= 27400 && dwVnum <= 27490)`

No value can satisfy both `dwVnum == 27591` and `dwVnum <= 27490`.

Therefore Carbon rod always falls into the normal `else` branch and receives only `rod->GetValue(0) / 10` rather than the explicit doubled special-case bonus.

Promoted as `BUG-FISH-005`.

## Probability arithmetic status
`GetFishCatchedVnum` takes `uint8_t normal_chance, uint8_t rare_chance`, while callers construct the rare value from `15 + POINT_FISHING_RARE + rod socket2`.

The cast/wrap boundary is real, but no current tracked producer/value range proving a >255 or otherwise invalid reachable value has been established yet. Keep this as a candidate, not a promoted bug.


## BUG-FISH-006 — renewed fishing event is not cancelled on death or warp
The renewed event is explicitly cancelled by:
- `fishing_new_stop()`;
- character destruction/logout final teardown;
- the event itself when rod/timeout conditions fail.

But neither `CHARACTER::Dead()` nor `CHARACTER::WarpSet()` cancels `m_pkFishingNewEvent`.

### Death path
`Dead()` sets `POS_DEAD`, changes the descriptor to `PHASE_DEAD`, and cancels the stun event, but does not cancel renewed fishing.

The fishing event itself does not check `IsDead()`.

If the server has already accepted the third CATCH before death and `m_bFishCatch >= FISHING_NEED_CATCH`, the next fishing event tick calls `fishing_catch_decision(info->vnum)` even though the character is now dead. That decision path also has no death check and can execute the normal final reward roll.

### Warp path
`CanWarp()` does not consider `m_pkFishingNewEvent`, and `WarpSet()` does not cancel it.

On a same-process/same-character warp, the renewed event therefore remains attached to the character. The event does not revalidate the original map/water location after warp, so the same fishing session can survive a map relocation until another stop condition fires.

Promoted as `BUG-FISH-006`.

## Logout/bait persistence candidate
`Disconnect()` flushes equipped items before final character destruction; final `Destroy()` cancels `m_pkFishingNewEvent` directly rather than calling `fishing_new_stop()`.

Because bait is stored in rod socket2 and `SetSocket` is persisted, a disconnect during active renewed fishing can save a non-zero bait socket before the event is destroyed. This is a plausible bait-persistence/abort semantic issue, but intended persistence semantics are not yet established, so it remains a candidate rather than a promoted bug.

## Rod refine closure
`ENABLE_FISHINGROD_RENEWAL` is active.

`RefinableRod()` requires:
- ITEM_ROD;
- not equipped;
- socket0 exactly equals Value2 mastery target.

`RealRefineRod()`:
- success creates RefinedVnum and replaces the old rod;
- failure under the active renewal flag keeps the rod grade and subtracts 10% of current mastery socket0.

No additional verified Fishing bug is promoted from the refine routine in this pass.


## BUG-FISH-007 — fish_new_log records a fresh reroll instead of the actual selected fish
`fishing_new_start()` selects the actual session fish once and stores it in `info->vnum`.

After the minigame completes, `fishing_catch_decision(itemVnum)` receives that actual selected VNUM.

However both success and failure logging call `GetFishCatchedVnum(...)` again instead of logging `itemVnum`.

Worse, `rod->SetSocket(2, 0)` is executed before the log call, so the reroll is computed without the bait/socket2 contribution used when the real session fish was selected.

Therefore `fish_new_log.vnum` is not a reliable record of the fish actually selected/rewarded for the session.

Promoted as `BUG-FISH-007`.

## BUG-FISH-008 — renewed successful catches do not progress TYPE_FISH achievements
Legacy successful fishing calls:
`CAchievementSystem::OnFishItem(ch, TYPE_FISH, item_vnum, 1)`.

Renewed `fishing_catch_decision()` rewards `itemVnum` but contains no `OnFishItem(...TYPE_FISH...)` call.

Current achievements.xml actively contains TYPE_FISH (numeric type 8) tasks.

Therefore successful catches through the enabled renewed system do not progress configured fishing achievements.

Promoted as `BUG-FISH-008`.

## BUG-FISH-009 — renewed catch updates the wrong Battle Pass fishing mission family
Current Battle Pass configs define three distinct fishing mission types:
- `FISH_FISHING`;
- `FISH_GRILL`;
- `FISH_CATCH`.

Legacy successful fishing advances `FISH_FISHING`.
Legacy `UseFish()` advances `FISH_CATCH`.

Renewed successful fishing instead calls:
`UpdateExtBattlePassMissionProgress(FISH_CATCH, 1, itemVnum)`.

Thus a renewed catch can progress the configured `FISH_CATCH` family while the configured `FISH_FISHING` mission does not progress from the actual fishing action.

Promoted as `BUG-FISH-009`.

## BUG-FISH-010 — disconnect preserves loaded bait across relog
Using a bait item:
- writes bait power into equipped rod socket2;
- consumes one bait item immediately.

Normal renewed stop/decision clears socket2.

But `Disconnect()` flushes equipped items before final character destruction. Final `Destroy()` cancels `m_pkFishingNewEvent` directly rather than calling `fishing_new_stop()`, so socket2 is not cleared during an active-session logout.

`CItem::SetSocket` is persisted and item saves serialize all sockets.

Therefore logging out during an active renewed fishing session can persist the non-zero bait socket; relog restores the already-consumed bait on the rod, allowing it to be reused in a later fishing start.

Promoted as `BUG-FISH-010`.


## BUG-FISH-011 — CATCH_FAILED is unthrottled and immediately rebroadcast to nearby players
`fishing_new_catch_failed()` checks only that `m_pkFishingNewEvent` exists.

For every accepted client packet it:
1. increments the uint32 failed counter;
2. immediately builds `FISHING_SUBHEADER_NEW_CATCH_FAILED`;
3. calls `PacketAround(&p, sizeof(p))`.

Unlike successful CATCH, there is no `GetLastCatchTime()` rate limit.

The accumulated failed count is consumed only by the periodic fishing event on the next tick, where it advances `info->sec` and may stop the session.

Therefore, during the interval before that tick, a modified client can burst repeated CATCH_FAILED packets and force the server to rebroadcast each one to nearby clients.

This is a multiplayer/server-amplification defect rather than a reward bypass.

Promoted as `BUG-FISH-011`.

## Final candidate closure
### POINT_FISHING_RARE / uint8 narrowing
`GetFishCatchedVnum` still accepts uint8 inputs while the caller expression is wider:
`15 + GetPoint(POINT_FISHING_RARE) + rod->GetSocket(2)`.

However, this source snapshot contains the point enum/use sites but no mapped active producer/apply source that proves a reachable value outside the uint8 domain. No current deployment row proving >255 was found.

Result: retain as a code-quality/domain-risk note only; do not promote as a current verified bug.

### CATCH_FAILED counter overflow
The counter is uint32 and unchecked, but practical overflow requires an extreme packet volume. The directly reachable and meaningful defect is the unthrottled immediate broadcast captured by `BUG-FISH-011`; no separate overflow bug is promoted.

## Static closure
Fishing Renewal coverage now includes:
- feature gates and packet registration;
- client minigame and trust boundary;
- start/stop/catch/fail/event lifecycle;
- movement/death/warp/logout/equipment cleanup;
- bait persistence;
- fish selection and reward tables;
- rod family and Carbon special handling;
- rod refine lifecycle;
- Battle Pass and Achievement integration;
- logging/audit integrity;
- current data/config cross-checks.

**Fishing Renewal: STATIC COMPLETE.**
