# Growth Pet System — Static Bug Registry

## BUG-GPET-001 — Current 55713 growth pet indexes past the 12-row hatch-info table

**Status:** VERIFIED STATIC / CURRENT-DATA REACHABLE

`PET_HATCH_INFO_RANGE` is declared:
`const float PET_HATCH_INFO_RANGE[12][6]`.

The hatching handler derives:
`const uint8_t bPetInfo = dwPetVnum - 55701;`
and then accesses:
- `PET_HATCH_INFO_RANGE[bPetInfo][1..2]` for lifetime;
- `PET_HATCH_INFO_RANGE[bPetInfo][3..4]` for skill count.

Current tracked proto contains:
- PET_EGG `55413` with Value0 `55713`;
- PET_UPBRINGING `55713`.

For 55713:
`55713 - 55701 = 12`,
but valid indices for a 12-row array are `0..11`.

No server range check occurs before those reads.

The enabled `ENABLE_PET_ATTR_DETERMINE` path has the same defect: after checking only ITEM_PET/PET_UPBRINGING, `CHARACTER::PetAttrChange()` indexes `PET_HATCH_INFO_RANGE[itemPet->GetVnum() - 55701]` for type/lifetime/skill-count generation. Upbringing VNUM 55713 therefore again indexes row 12 out of bounds.

The client hatching UI reads the egg's Value0 and sends the normal hatching request; it does not remove the 55413/55713 path.

**Impact:** the current 13th pet family can read beyond the hatch-info table during ordinary hatching and during attribute determination. Outcomes are undefined and can include invalid lifetime/skill parameters, memory-safety faults, or process instability.

**Runtime:** Stage C debug/ASan with disposable 55413/55713 data only. See `GPET-T01`.


## BUG-GPET-002 — Feed packet count can index past the fixed 9-slot packet array

**Status:** VERIFIED STATIC / MODIFIED-CLIENT MEMORY-SAFETY

`TPacketCGGrowthPetFeedRequest` contains exactly:
`uint16_t iFeedItemsCubeSlot[9]`
plus the client-controlled `uint16_t iFeedItemsCount`.

`CInputMain::GrowthPetFeedRequest()` forwards both values directly to:
`CGrowthPetSystemActor::ItemCubeFeed(..., p->iFeedItemsCubeSlot, p->iFeedItemsCount)`
without enforcing `iFeedItemsCount <= 9`.

`ItemCubeFeed()` then executes:
`for (uint32_t i = 0; i < wFeedItemsCount; ++i)`
and reads `iFeedItemsCubeSlot[i]`.

A crafted count above 9 therefore reads beyond the packet's fixed slot array.

**Impact:** server-side out-of-bounds read from the receive-buffer/adjacent packet memory, with invalid inventory-cell interpretation and crash/undefined-behavior potential.

**Runtime:** Stage C crafted-packet/ASan only. See `GPET-T02`.

## BUG-GPET-003 — Pet skill packet slot indices are used against fixed 3-element arrays without range checks

**Status:** VERIFIED STATIC / MODIFIED-CLIENT MEMORY-SAFETY

`TGrowthPetInfo` stores all per-skill state in fixed arrays of length 3:
- `skill_vnum[3]`;
- `skill_level[3]`;
- `skill_spec[3]`;
- cooldown/formula arrays `[3]`.

Client packets carry unrestricted `uint8_t` slot indices.

Mapped server paths use those indices before any `slot < 3` validation:
- `LearnPetSkill()` reads `skill_vnum[bSkillBookSlotIndex]`;
- `PetSkillUpgradeRequest()` reads `skill_level[bSkillSlot]`;
- `IncreasePetSkill()` reads and can increment `skill_level[bSkillSlot]`;
- `DeleteSkill()` reads/writes the skill arrays at `bSkillBookDelSlotIndex`.

A crafted slot 3..255 therefore escapes the three real skill entries; some paths are read-only while upgrade/delete paths can write outside the arrays.

**Impact:** server memory corruption / crash candidate through Growth Pet skill packets.

**Runtime:** Stage C crafted-packet/ASan only. See `GPET-T03`.

## BUG-GPET-004 — Learn-skill accepts arbitrary inventory items and can persist an out-of-range skill VNUM

**Status:** VERIFIED STATIC / CURRENT-DATA + MODIFIED-CLIENT REACHABLE

`LearnPetSkill(slot, inventoryCell)` fetches the supplied inventory item but never verifies:
- `ITEM_PET`;
- subtype `PET_SKILL`;
- or that `GetValue(0) < PET_SKILL_MAX`.

It treats:
`const uint8_t skill_value = pItem->GetValue(0)`
as the new skill VNUM, stores it in `m_PetInfo.skill_vnum[slot]`, consumes the supplied item, and calls `GiveBuff()`.

Current tracked proto has many non-pet items whose Value0 falls outside the valid Growth Pet skill range 0..23. Example:
- VNUM 25101, ITEM_USE/USE_SPECIAL, Value0 = 100.

After such a crafted learn request, `GiveBuff()` and later `Summon()` use:
`pet_skill_table[skill_vnum][0]`
with no skill-VNUM range guard.

Thus Value0=100 produces an out-of-bounds skill-table access immediately after learning and leaves an invalid value in persistent pet state if execution continues.

Even in-range Value0 values from unrelated items can be used as unauthorized substitute skill books.

**Impact:** skill-book type bypass plus persistent invalid skill metadata and server out-of-bounds read/crash potential.

**Runtime:** Stage C crafted-packet/ASan with disposable pet/item data only. See `GPET-T04`.

## BUG-GPET-005 — Premium revive trusts client material count and can turn an undersized arbitrary item stack into the item-count limit

**Status:** VERIFIED STATIC / MODIFIED-CLIENT ECONOMIC + DATA-INTEGRITY

Premium revive is enabled in the tracked build.

The material predicate in `CHARACTER::RevivePet()` is:
`if (material->GetType() == ITEM_PET && material->GetSubType() != PET_PREMIUM_FEEDSTUFF) return;`

This rejects only the wrong **PET** subtypes. Any non-PET inventory item passes.

The server then validates the required revive quantity against:
`revivePacket->count[i]`
which is supplied by the client, rather than against `material->GetCount()`.

Once the claimed count is sufficient, it consumes with:
`material->SetCount(material->GetCount() - requiredMaterialCount)`.

Both operands are unsigned counts. If the actual stack is smaller than the required count, subtraction underflows to a huge `uint32_t`. `CItem::SetCount()` then clamps that huge value to `g_bItemCountLimit` instead of rejecting it.

Therefore a crafted revive request can:
1. select a non-PET item;
2. claim enough count in the packet;
3. provide an actual stack smaller than the revive requirement;
4. pass validation;
5. underflow the subtraction and inflate the selected stack to the configured item-count limit;
6. continue into pet revival.

Tracked proto also contains the legitimate premium feed VNUM 55100, confirming this feature is data-backed.

**Impact:** arbitrary-item stack inflation/duplication plus revival-cost/type bypass.

**Runtime:** Stage C destructive crafted-packet test with disposable data only. See `GPET-T05`.

## BUG-GPET-006 — Evolution requirement can be satisfied by repeating one inventory slot for every required material entry

**Status:** VERIFIED STATIC / MODIFIED-CLIENT ECONOMIC

Growth Pet evolution obtains a map of seven distinct required VNUM/count pairs for each evolution stage.

Validation does not track which requirement has already been satisfied. For each of the expected input positions it:
1. resolves the client-supplied inventory cell;
2. checks whether that item's VNUM exists anywhere in the requirement map;
3. checks global `CountSpecifyItem(vnum) >= requiredCount`;
4. increments `iItemFound`.

There is no duplicate-cell or duplicate-required-VNUM rejection.

A crafted request can therefore repeat the **same inventory cell** in all seven material positions. Before consumption, the same stack satisfies the same global count check seven times, so `iItemFound == itemlistcnt`.

Consumption then re-resolves the listed cells. If the chosen stack contains exactly its own required count, the first `RemoveSpecifyItem()` deletes it; later repetitions resolve empty and are skipped. `EvolvePet()` is still called unconditionally afterward.

**Impact:** a pet evolution that normally requires seven distinct material families can proceed while paying only one required material stack.

**Runtime:** Stage B modified-client / disposable-material test. See `GPET-T06`.

## BUG-GPET-007 — Premium revive computes new pet birthday/duration in a local copy and never stores it back

**Status:** VERIFIED STATIC / NORMAL-FLOW DATA-INTEGRITY

`CHARACTER::Revive()` obtains:
`TGrowthPetInfo info = itemPet->GetGrowthPetItemInfo();`

It then computes:
- a renewed maximum-duration value in `info.pet_max_time`;
- a premium age reduction in `info.pet_birthday` (80% of prior computed age), or a birthday reset in the non-premium branch.

However the function ends without:
- `itemPet->SetGrowthPetItemInfo(info)`;
- updating the character Growth Pet info;
- or otherwise persisting that modified local struct.

Only socket 0 is changed and its real-time event restarted.

The database persists `pet_duration` and `pet_birthday` from the item's `aPetInfo`, so these local changes are not merely delayed—they are absent from the persistent source object.

**Impact:** revive renews the expiration socket but discards its intended structured age/birthday changes. Age-dependent evolution/stat behavior and later revive calculations continue from stale pet metadata.

**Runtime:** Stage A normal disposable-pet state comparison/relog test. See `GPET-T07`.

## BUG-GPET-008 — Specialist skill table is declared two-dimensional but lookup examines only every fourth entry

**Status:** VERIFIED STATIC / NORMAL-FLOW GAMEPLAY

`pet_skill_specialist_table` is declared as:
`const TPetSkillSpecialistTable pet_skill_specialist_table[][4]`.

Its initializer is a flat sequence of specialist records.

`GetGrowthPetSkillSpecialistValue()` uses:
`for (auto table : pet_skill_specialist_table)`
and then dereferences `table->...`.

Each loop element is therefore one four-record array and `table->` addresses only its first record. The remaining three records in every group are never compared.

The current initializer has 54 specialist records; the lookup can examine only the first record of each four-record aggregate (14 effective candidates, with the last partial group zero-filled).

For example, the listed Monkey HEAL specialist record is not a group-first entry and therefore cannot be matched by this lookup, while some unrelated later entries happen to land on group boundaries and remain reachable.

**Impact:** most configured pet/skill specialist ranges silently never apply; skill values depend on accidental initializer position rather than the declared specialist data.

**Runtime:** Stage A deterministic normal-flow specialist-value observation. See `GPET-T08`.

## BUG-GPET-009 — HEAL auto-skill passes an absolute target HP as a delta and over-heals

**Status:** VERIFIED STATIC / NORMAL-FLOW GAMEPLAY

The HEAL auto-skill computes:
`restore_hp = MIN(ownerHP + skill_spec, maxHP)`.

That value is an **absolute target HP**.

It then calls:
`PointChange(POINT_HP, restore_hp)`.

`CHARACTER::PointChange(POINT_HP, amount)` is delta-based:
it executes `SetHP(GetHP() + amount)` after clamping the delta to remaining max HP.

Therefore HEAL adds the computed target HP on top of the current HP instead of adding only the intended heal amount.

Examples:
- with specialist value 0, it still passes current HP as the heal delta and can approximately double current HP;
- with a ~7000 specialist heal, the delta becomes `currentHP + 7000`, frequently filling HP to maximum far earlier than intended.

**Impact:** Growth Pet HEAL magnitude is materially larger than its configured specialist amount and can collapse low-HP recovery into a near/full heal.

**Runtime:** Stage A normal-flow HP before/after observation. See `GPET-T09`.

## BUG-GPET-010 — Hatching and pet name-change use unbounded strlen() on packet character arrays

**Status:** VERIFIED STATIC / MODIFIED-CLIENT MEMORY-SAFETY

Both packet structures carry fixed `char[PET_NAME_MAX_SIZE + 1]` name arrays.

The server validates them first with unbounded C-string scans:
- hatching: `strlen(p->sGrowthPetName)`;
- name change: `strlen(p->sPetName)`.

A network packet is not intrinsically guaranteed to contain a NUL byte inside that fixed array. A crafted packet can fill the entire field with nonzero bytes.

The later code does use bounded `strnlen(..., sizeof(field))` for SQL escaping, but that happens only **after** the unsafe `strlen()` calls.

**Impact:** out-of-bounds read past the fixed packet name field, potentially continuing beyond the packet structure into adjacent receive-buffer memory.

**Runtime:** Stage C crafted-packet/ASan only. See `GPET-T10`.

## BUG-GPET-011 — Newly hatched pets store duration in socket1 but the evolution age gate interprets socket1 as a birth timestamp

**Status:** VERIFIED STATIC / CURRENT NORMAL-FLOW REACHABLE

During hatching:
- `petInfo.pet_birthday = time(0)`;
- `petInfo.pet_max_time` is a duration in seconds;
- socket 0 receives the absolute expiry `birthday + max_time`;
- socket 1 receives **`petInfo.pet_max_time`**.

The actor helper later defines:
`GetPetBirthday() = (get_global_time() - m_pkPetSeal->GetSocket(1)) / 86400`.

So socket 1 is interpreted as an absolute historical timestamp even though hatching stored only a small 1..45 day duration.

`CanIncreaseEvolvePet()` uses this helper for the evolution-3 -> evolution-4 age gate:
`GetEvolution() == 3 && GetPetBirthday() >= 30`.

With current epoch time and a socket1 value of only a few days in seconds, the computed result is thousands of days. A newly hatched pet therefore satisfies the nominal 30-day age condition as soon as its level/evolution prerequisites are met.

The structured `m_PetInfo.pet_birthday` used by `GetPetAgeDays()` is initialized correctly; the defect is specifically the socket1-based evolution-age helper.

**Impact:** the intended 30-day final evolution age requirement is effectively bypassed for normally hatched current pets.

**Runtime:** Stage A normal-flow socket/age observation. See `GPET-T11`.


## BUG-GPET-012 — Unsummoned pet name-change returns success without changing the name or consuming the scroll

**Status:** VERIFIED STATIC / NORMAL CLIENT REACHABLE

Normal client `PetNameChangeWindow.NameChangeWindowOpen()` accepts an inventory growth-pet seal with a valid pet id; it does not require that pet to be currently summoned.

The client sends:
- name-change material slot;
- target pet seal slot;
- requested name.

Server `CInputMain::GrowthPetNameChangeRequest()` validates both items and the name, then resolves the target pet actor.

If that pet is summoned:
- `pGrowthPetSystem->ChangeName(__pet_name)` executes;
- the PET_NAME_CHANGE material is removed;
- success is sent.

If that pet is **not** summoned, the server instead:
1. constructs `NAME_CHANGE_USE_SUCCESS`;
2. sends the success packet and chat message;
3. immediately `return`s.

That branch never:
- writes `pet_nick` into the target seal's `TGrowthPetInfo`;
- saves the target item;
- consumes the PET_NAME_CHANGE material.

The client treats success as authoritative and closes the dialog.

**Impact:** the normal unsummoned rename flow falsely reports success while performing no rename and charging no material. The user sees a completed operation but the pet name remains unchanged.

**Runtime:** Stage A/B disposable unsummoned-pet rename check; see `GPET-T12`.


## BUG-GPET-013 — Invincibility pet skill uses a hard-coded 123-second effect duration

**Status:** VERIFIED STATIC / CURRENT SKILL DATA REACHABLE

The tracked client skill table exposes Growth Pet skill 17 as:
`Immortal / AUTO / Chance of temporary invincibility / cooldown 600`.

The matching server table row is:
`PET_SKILL_AFFECT_INVINCIBILITY, AUTO, APPLY_NONE, 52, 2.7, 600`.

`GetPetSkillInformation()` derives the skill's scaled formula values and stores them into the active pet state.

But the execution branch in `CGrowthPetSystemActor::Update()` ignores the computed effect-duration/value path:
```
 //const long duration = m_dwSkillDuration[slot];
 const long duration = 123; //review
```

It then applies:
`AFFECT_IMPOSSIBLE_ATTACK`
for exactly 123 seconds.

The explicit `//review` placeholder plus the configured/scaled skill formula path makes this a current implementation defect rather than an intended table-driven value.

**Impact:** a successful Immortal proc grants a fixed 123-second invulnerability-like affect instead of the configured/scaled skill duration.

**Runtime:** Stage B controlled low-HP disposable test; see `GPET-T13`.

## BUG-GPET-014 — Feather skill is shipped but its server execution immediately aborts pet Update and follow AI

**Status:** VERIFIED STATIC / CURRENT SKILL DATA REACHABLE

The tracked client skill table exposes skill 23:
`Light as a Feather / AUTO / Activates Feather Walk / cooldown 180`.

The matching server skill row and client UI asset `feather.sub` are present.

However the active-skill execution branch is:
```
case PET_SKILL_FEATHER:
{
    return false;
    // under construction
}
```

This returns from `CGrowthPetSystemActor::Update()` before the normal:
`_UpdateFollowAI()`
call at the bottom of the function.

The Growth Pet system event ignores the actor Update return as a stop signal and continues ticking, so the same early return repeats while Feather remains in a skill slot.

**Impact:** Feather provides no advertised skill effect/cooldown handling and can continuously suppress that pet actor's normal follow-AI update.

**Runtime:** Stage A normal-flow follow observation with disposable pet/skill; see `GPET-T14`.

## BUG-GPET-015 — Invincibility and Panacea use the next skill level's proc probability

**Status:** VERIFIED STATIC / NORMAL FLOW

`GetPetSkillInformation()` computes both:
- `skill_formula1` for the current skill level;
- `next_skill_formula1` for `skill_level + 1`.

HEAL correctly tests its proc with:
`m_PetInfo.skill_formula1[slot]`.

By contrast both:
- `PET_SKILL_AFFECT_INVINCIBILITY`;
- `PET_SKILL_AFFECT_REMOVAL` (Panacea)

test:
`m_PetInfo.next_skill_formula1[slot]`.

Therefore these two AUTO skills use the probability intended for the next upgrade level before that level has actually been reached.

The current server table uses base probability values 52 (Invincibility) and 41 (Panacea), so the difference exists throughout non-max skill progression.

**Impact:** proc chance is systematically higher than the current learned skill level is configured to provide.

**Runtime:** Stage A statistical/debug-state validation; see `GPET-T15`.


## BUG-GPET-016 — Crafted attribute-change on the summoned seal is later overwritten by stale actor state

**Status:** VERIFIED STATIC / MODIFIED-CLIENT REACHABLE

The normal client refuses to place the currently active pet seal into the attribute-change target slot:
`metinSlot[2] == GetActivePetItemId() -> return false`.

The server does not repeat that invariant.

`GrowthPetAttrChangeRequest()` directly calls:
`CHARACTER::PetAttrChange(petSlot, materialSlot)`.

`PetAttrChange()` validates item type/subtype and lifetime, but does not reject:
- the currently summoned pet;
- a locked pet seal;
- a pet whose id is owned by the active Growth Pet actor.

It then writes newly rolled type/lifetime/HP/DEF/SP state into the seal with:
`itemPet->SetGrowthPetItemInfo(info)`
and updates the CHARACTER-side pet-info cache.

The active `CGrowthPetSystemActor`, however, still owns its older private `m_PetInfo` copy.

Later normal actor dismissal executes:
`m_pkPetSeal->SetGrowthPetItemInfo(m_PetInfo)`
and saves it, overwriting the attribute-changed seal with the stale pre-change actor state.

**Impact:** a crafted request against the active pet can consume the attribute-change material, temporarily report the new state, and then lose/revert that state when the pet is dismissed.

**Runtime:** Stage B modified-client / disposable pet only; see `GPET-T16`.

## BUG-GPET-017 — Multi-item Life/EXP feed processes only the final selected slot

**Status:** VERIFIED STATIC / NORMAL CLIENT REACHABLE

The normal Pet Feed UI supports up to `PET_FEED_SLOT_MAX` entries, builds:
`resultFeedItems = [all populated feed slots]`,
and sends the whole list with `SendPetFeedPacket()`.

Server `CGrowthPetSystemActor::ItemCubeFeed()` first loops across `wFeedItemsCount`, but only assigns each resolved item into one local variable:
`pFeedItem`.

After the loop, `pFeedItem` therefore refers only to the **last** submitted slot.

For:
- `PET_FEED_WINDOW` (life feeding);
- `PET_EXP_ITEM_WINDOW` (EXP feeding),

all type checks, EXP/lifetime calculation and item consumption operate only on that final `pFeedItem`.

The earlier selected slots are neither consumed nor applied. The client nevertheless receives a successful feed result and clears its feed-window selection.

The EVOLVE branch is structurally different and re-iterates the submitted slot array; this finding is limited to Life/EXP feed modes.

**Impact:** a normal multi-item feed action applies only one selected item, so displayed/selected feed batches do not match server-side effect and consumption.

**Runtime:** Stage A normal-client multi-slot feed observation with disposable items; see `GPET-T17`.


## BUG-GPET-018 — Destroying a Growth Pet seal leaves an orphan row in the separate pet table

**Status:** VERIFIED STATIC / NORMAL ITEM LIFECYCLE

Growth Pet persistence uses two linked records:

1. the normal `item` row for the seal;
2. a separate `pet` row keyed by a generated pet id stored in seal socket 2.

Hatching inserts the pet row first:
`INSERT INTO pet (...) VALUES(NULL, ...)`,
then stores the returned auto-increment id into:
- `petInfo.pet_id`;
- seal `socket2`.

Load later resolves the structured pet state with:
`SELECT ... FROM pet WHERE id = item.alSockets[2]`.

DB-side deletion support exists:
`HEADER_GD_PET_ITEM_DESTROY -> QUERY_PET_ITEM_DESTROY() -> DELETE FROM pet WHERE id=...`.

However the only game-side send found in the tracked source is inside `ITEM_MANAGER::DestroyItem()`, and that entire Growth Pet block is commented out:
```
/*
if (item is PET_UPBRINGING) {
    dwSocketID = item->GetSocket(2);
    send HEADER_GD_PET_ITEM_DESTROY(dwSocketID);
}
*/
```

The active destruction path sends only `HEADER_GD_ITEM_DESTROY`, which removes the normal item row.

Consequently deleting/destroying a Growth Pet seal removes its item record but does not remove the associated structured row from `pet`.

**Impact:** permanent orphan Growth Pet rows accumulate in the database. The tracked code does not establish normal pet-id reuse, so stale-row adoption is not claimed here; the confirmed defect is orphan persistence/storage leakage.

**Runtime:** Stage B disposable-pet + DB verification only after runtime phase unlock; see `GPET-T18`.
