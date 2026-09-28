# Refine / Cube / Crafting — Verified Bugs

## BUG-REFCUBE-001 — Cube Renewal accepts non-positive/out-of-domain multiplier

**Status:** VERIFIED STATIC / MODIFIED-CLIENT ECONOMY/CRAFTING BUG

`TSubPacketCGCubeRenwalMake.multiplier` is a signed `int` controlled by the client.

Normal UI constrains its own value, but `CInputMain::CubeRenewalSend()` forwards the received integer directly to `CCubeManager::RefineCube()`.

The server has no `multiplier >= 1` or maximum-domain check.

For **multiplier = 0**, all multiplied material/Yang/Gem requirements become zero. The availability gates therefore pass even with no required resources. Actual material removal later uses the unmultiplied base count and its result is not checked; reward creation can still continue. Currency cost is also zero.

For **negative multiplier**, the multiplied requirements become negative and ordinary non-negative balances satisfy the checks.

Preconditions use:
- `CountSpecifyItem(...) < material.count * multiplier`;
- `GetGold() < gold * multiplier`;
- `GetGemPoint() < gem_point * multiplier`.

Currency mutation later uses:
- `PointChange(POINT_GOLD, -(gold * multiplier), false)`;
- `PointChange(POINT_GEM, -(gem_point * multiplier), false)`.

Therefore a negative multiplier reverses the sign and credits recipe currency costs. A zero multiplier can bypass multiplied availability/currency requirements entirely while still reaching the single-reward path.

Extremely large signed values are likewise outside any server-enforced domain and can enter signed multiplication overflow territory; the canonical defect is the missing authoritative multiplier range validation.

**Impact:** crafted Cube Renewal requests can bypass recipe resource/currency requirements with zero multiplier or turn positive Yang/Gem costs into credits with a negative multiplier.

**Runtime:** Stage B isolated modified-client/economy test only; do not execute in production. See `REFCUBE-T01`.

## BUG-REFCUBE-002 — Cube Renewal batch multiplier is omitted from material consumption and reward quantity

**Status:** VERIFIED STATIC / NORMAL-CLIENT REACHABLE

For stackable result recipes, the normal Cube Renewal UI lets the player choose a multiplier greater than 1 (up to 200) and displays:
- required material count × multiplier;
- reward count × multiplier;
- Yang/Gem cost × multiplier.

The server also validates:
`CountSpecifyItem(material) >= material.count * multiplier`
and charges Yang/Gem using the multiplier.

However removable recipe materials are consumed with:
`RemoveSpecifyItem(Material.vnum, Material.count, ...)`
without multiplying the count.

The normal reward path creates:
`CreateItem(itemVnum, itemCount)`
also without multiplying reward count.

Therefore a legitimate multi-craft request does not execute the quantity contract represented by the UI and pre-validation.

**Impact:** for multiplier >1, material consumption, currency cost and produced quantity diverge. Removable items are under-consumed relative to the requested batch, while the output is also under-produced and the full multiplied Yang/Gem cost is still charged.

**Runtime:** Stage A/B controlled disposable recipe test. See `REFCUBE-T02`.


## BUG-REFCUBE-003 — Cube Renewal MAKE remains authorized after window close / at arbitrary distance

**Status:** VERIFIED STATIC / MODIFIED-CLIENT REACHABLE

The Renewal open command caches the current quest NPC race in:
`SetTempCubeNPC(ch->GetQuestNPC()->GetRaceNum())`
and stores the live NPC pointer through `SetCubeNpc()`.

`Cube_close()` clears the live Cube NPC pointer and `W_CUBE`, but it does **not** clear `tempCubeNPC`.

The MAKE packet path does not require:
- `IsCubeOpen()`;
- `GetOpenedWindow(W_CUBE)`;
- a live Cube NPC pointer;
- a current distance check to that NPC.

`RefineCube()` instead selects recipes only from the stale cached `GetTempCubeNPC()`.

Therefore, once a player has legitimately opened a Cube Renewal NPC at least once, closing the window or moving away does not revoke the server-side recipe identity. A crafted MAKE packet can still submit recipes belonging to that cached NPC, subject only to the normal recipe/material/currency checks.

**Impact:** Cube crafting authorization survives window close and NPC range; crafting can be performed remotely using the last cached Cube NPC identity.

**Runtime:** Stage B isolated modified-client authorization test only. See `REFCUBE-T03`.

## BUG-REFCUBE-004 — Player-accessible /cube command dereferences a null quest NPC

**Status:** VERIFIED STATIC / CRASH CANDIDATE

The command table registers:
`"cube" -> do_cube`
at `GM_PLAYER` and `POS_DEAD`, so ordinary players can invoke the command.

Under `ENABLE_CUBE_RENEWAL`, `do_cube()` immediately executes:
`ch->SetTempCubeNPC(ch->GetQuestNPC()->GetRaceNum());`
and then again uses `ch->GetQuestNPC()`.

There is no null check.

A character initializes `m_dwQuestNPCVID = 0`, and `GetQuestNPC()` resolves it through `CHARACTER_MANAGER::Find(m_dwQuestNPCVID)`. Without a valid current quest NPC this can return null.

Thus a direct player `/cube` command outside an NPC quest context can dereference a null pointer.

**Impact:** player-reachable game-process crash candidate from an ordinary command path.

**Runtime:** Stage C crash/sanitizer test only; do not execute in production. See `REFCUBE-T04`.

## BUG-REFCUBE-005 — Cube improve items are consumed before reward-space validation

**Status:** VERIFIED STATIC / NORMAL-CLIENT REACHABLE DATA LOSS

Cube Renewal supports the chance-improvement item VNUM `79605`.

When `indexImprove != -1`, the server:
1. resolves the inventory item;
2. verifies VNUM 79605 and count <= 40;
3. computes the added success chance;
4. immediately consumes the applicable improve-item count with `SetCount(... - substract)`.

Only **after that consumption** does the code create a temporary reward item and call `GetEmptyInventory()` / `GetEmptyDragonSoulInventory()` to verify reward space.

If no destination slot is available, the temporary reward is destroyed and the craft returns without consuming normal recipe materials or currency — but the already-consumed improve items are not restored.

The normal UI sends the improve-item slot directly and has no equivalent free-inventory guard before `SendRefine()`.

**Impact:** a normal player can lose Cube chance-improvement items when attempting a craft with insufficient reward inventory space.

**Runtime:** Stage A controlled disposable-item test. See `REFCUBE-T05`.


## BUG-REFCUBE-006 — Cube recipe control fields are left uninitialized when directives are absent

**Status:** VERIFIED STATIC / CURRENT DEPLOYMENT REACHABLE

`CUBE_DATA` has a user-defined constructor:

`CUBE_DATA() : set_value(0), gem_point(0) {}`

It does **not** initialize at least:
- `allow_copy`;
- `not_remove`;
- `percent`;
- `gold` (later explicitly set to 0 at section start).

During parsing, a field is assigned only when its matching directive exists. Craft execution then reads `allow_copy` and `not_remove` as transaction-control values.

Current tracked `cube.txt` contains **3327 complete recipe sections**:
- `allow_copy`: present in **0 / 3327** sections;
- `not_remove`: present in **2815 / 3327** sections, absent in **512**;
- `percent`: present in all 3327 sections.

Therefore `allow_copy` is indeterminate for every current recipe, and `not_remove` is indeterminate for 512 current recipes.

These values affect live behavior:
- `allow_copy || SetVal` can skip normal removal of the first recipe material;
- the same condition can invoke `CopyAllAttrTo()` from that first material into the reward;
- later source-count handling changes based on `allow_copy`;
- `NotRem` can suppress material removal or subsequent cleanup when treated as nonzero / matching a VNUM.

Because reading an uninitialized scalar is undefined/indeterminate behavior, observed results can depend on allocator/stack/build state and are not a reliable implicit zero default.

**Impact:** current Cube recipes can nondeterministically enter copy/not-remove semantics, causing incorrect material consumption and/or metadata copying even though the deployment never enables `allow_copy`.

**Runtime:** Stage C debug/MemorySanitizer or deterministic initialization-comparison test only. See `REFCUBE-T06`.


## BUG-REFCUBE-007 — Classic refine dereferences the source item after RemoveItem destroys it

**Status:** VERIFIED STATIC / NORMAL REFINE SUCCESS REACHABLE

`ITEM_MANAGER::RemoveItem(item)` removes the item from its owner and ends with:
`M2_DESTROY_ITEM(item)` -> `ITEM_MANAGER::DestroyItem()` -> `M2_DELETE(item)`.

Classic refine paths keep using the raw `item` pointer after that destruction.

### Normal blacksmith / guild / money-only refine
On successful non-Metin refine, `DoRefine()`:
1. creates the refined item;
2. copies metadata;
3. calls `RemoveItem(item, "REMOVE (REFINE SUCCESS)")`;
4. then can call:
   - announcement predicates using `item->CheckItemUseLevel()`, `item->GetType()`, `item->GetSubType()`;
   - `IsConquerorItem(item)` and `CRandomHelper::RefineRandomAttr(item, ...)`;
   - Battle Pass progress with `item->GetVnum()`.

This build enables `ENABLE_ANNOUNCEMENT_REFINE_SUCCES`, `ENABLE_YOHARA_SYSTEM` and `ENABLE_BATTLE_PASS_SYSTEM`. The Battle Pass call alone makes a post-destruction dereference part of ordinary successful non-Metin refinement.

### Scroll refine
`DoRefineWithScroll()` has the same ordering on success, and the grade-down failure branch also removes the old item before Yohara `IsConquerorItem(item)` / random-refine handling.

### Serpent refine
`DoRefineSerpent()` likewise removes the source before the Yohara random-refine call.

Stackable Metin handling can avoid immediate object destruction when only count is decremented, but the ordinary weapon/armor/accessory paths call `RemoveItem()` and destroy the object.

**Impact:** use-after-free during normal successful classic refinement, with crash/corruption/stale-data potential; additional UAF exists on relevant scroll downgrade/Serpent paths.

**Runtime:** Stage C debug/ASan only. See `REFCUBE-T07`.


## BUG-REFCUBE-008 — Classic REFINE_TYPE_NORMAL is not bound to an opened refine session or blacksmith

**Status:** VERIFIED STATIC / MODIFIED-CLIENT REACHABLE

`CInputMain::Refine()` accepts `TPacketCGRefine.type` directly.

For `REFINE_TYPE_NORMAL` it resolves the inventory item and immediately calls:
`ch->DoRefine(item)`.

There is no authoritative check that:
- `RefineInformation()` was previously called;
- `m_bUnderRefine` / a refine session is active;
- the packet type matches the refine type that the server offered;
- a stored refine NPC is still valid / in range.

`DoRefine()` reinforces the gap by calling:
`CanHandleItem(true)`,
where `true` explicitly skips the under-refine check.

It does scan nearby entities through `FindBlacksmith`, but when no valid blacksmith is found the code only emits:
`HackLog("REFINE_FAR_BLACKSMITH", ...)`
and then deliberately continues execution.

Therefore a crafted `HEADER_CG_REFINE` packet with `REFINE_TYPE_NORMAL` can invoke ordinary refinement from arbitrary location without first opening a blacksmith refine dialog, as long as the item/refine recipe/material/currency checks themselves pass.

The same trust boundary also means a client can choose NORMAL independently of the type originally displayed by a refine-information flow.

**Impact:** remote / sessionless classic refinement; blacksmith proximity is logged rather than enforced.

**Runtime:** Stage B isolated modified-client authorization test only. See `REFCUBE-T08`.


## BUG-REFCUBE-009 — Soul Awake scroll is assigned the wrong refine request type

**Status:** VERIFIED STATIC / CURRENT FEATURE-DATA REACHABLE

`RefineItem()` handles both Soul scroll values in one branch.

The intended mapping is:
- `SOUL_EVOLVE_SCROLL` -> `REFINE_TYPE_SOUL_EVOLVE`;
- `SOUL_AWAKE_SCROLL` -> `REFINE_TYPE_SOUL_AWAKE`.

But the code is:
`if (pkItem->GetValue(0) == SOUL_EVOLVE_SCROLL) ...`
`else if (pkItem->GetValue(0) == SOUL_EVOLVE_SCROLL) ...`

The second comparison repeats EVOLVE instead of testing `SOUL_AWAKE_SCROLL`.

Thus when the actual Awake scroll is used, `refType` remains its initial `REFINE_TYPE_SCROLL`.

`RefineInformation()` special-cases ITEM_SOUL only for `REFINE_TYPE_SOUL_EVOLVE` / `REFINE_TYPE_SOUL_AWAKE`; with generic SCROLL it sends the wrong refine type.

On confirmation, `CInputMain::Refine()` dispatches only the two Soul refine types to `DoRefineSoul()`; generic SCROLL goes to `DoRefineWithScroll()` instead.

Current build enables `ENABLE_SOUL_SYSTEM`, and tracked item data contains:
- 70602 — Soul parchment / evolve;
- 70603 — Soul parchment / awake;
- 70500..70509 Soul item family names.

**Impact:** the normal Soul Awake scroll flow is routed through the generic scroll refine path instead of the dedicated Soul-awakening transaction/probability path.

**Runtime:** Stage A controlled disposable Soul-item test after global phase unlock. See `REFCUBE-T09`.


## BUG-REFCUBE-010 — Refine skill bonus is applied to the random roll, reducing real success while the UI reports an increase

**Status:** VERIFIED STATIC / NORMAL REFINE REACHABLE

Current build enables `ENABLE_REFINE_ABILITY_SKILL`.

The configured bonus tables are positive:
- normal blacksmith: `aiRefinePowerByLevel` = 0..6;
- guild blacksmith: `aiGuildRefinePowerByLevel` = 0..3.

`RefineInformation()` presents these values as success bonuses:
- normal: `prt->prob + refine_skill`;
- guild: `prt->prob + 10 + guild_refine_skill`.

But `DoRefine()` does not increase the success threshold. Instead it increases the random roll:
`int prob = number(1, 100);`
then
- normal: `prob += refine_skill`;
- guild / money-only: `prob += 10 + guild_refine_skill`;
and success remains:
`if (prob <= prt->prob)`.

For positive bonus `k`, the real success probability becomes approximately `max(prt->prob - k, 0)%`, while the UI advertises `min(prt->prob + k, 100)%`.

Example:
base 50%, normal refine skill +6:
- UI: 56%;
- execution: `roll + 6 <= 50` -> 44%.

Guild/money-only additionally subtracts roughly 10..13 percentage points from the base while the UI reports that same amount as an increase.

**Impact:** leveling the refine ability skill makes actual normal refinement worse, opposite to the displayed probability; guild/money-only probability is likewise inverted.

**Runtime:** Stage A statistical/debug RNG-seed validation only after phase unlock. See `REFCUBE-T10`.

## BUG-REFCUBE-011 — Scroll refine preview probability does not match the execution formula

**Status:** VERIFIED STATIC / NORMAL UI FLOW REACHABLE

For non-guild `RefineInformation()`, the displayed probability is always built as:
`prt->prob + refine_skill + scroll_buff`.

However `DoRefineWithScroll()` does not use the refine skill bonus at all and several scroll values are **absolute success probabilities**, not additive buffs.

Examples:
- Magic Stone: execution = `prt->prob + 10`; preview = `prt->prob + refine_skill + 10`.
- Dragon Scroll: execution = table `{100,75,65,55,45,40,35,25,20}`; preview adds that absolute table value to `prt->prob + refine_skill`, often clamping to 100.
- War/Musin Scroll: execution = 100%; preview adds 100 to base and clamps to 100.
- Smith Handbook: execution = absolute table `{100,100,90,80,70,60,50,30,20}`; preview adds it to base + skill.
- Memo: execution = 100%; preview also clamps 100, coincidentally matching.
- BDragon: execution = 80%; preview = base + skill + 80, generally clamped/higher.
- Ritual / Seal of God: execution = base + 15 / +20; preview additionally includes refine skill, which execution ignores.

Thus the probability sent in `TPacketGCRefineInformation` is not an authoritative representation of the probability later used by the server transaction.

**Impact:** the refine dialog can materially overstate scroll success chance, including showing 100% for attempts that execute below 100%.

**Runtime:** Stage A deterministic formula comparison with disposable items only. See `REFCUBE-T11`.


## BUG-REFCUBE-012 — Refine preview reads set_value with the wrong Python API signature

**Status:** VERIFIED STATIC / NORMAL CLIENT

Current build enables both `ENABLE_YOHARA_SYSTEM` and `ENABLE_SET_ITEM`.

In `root/uirefine.py::RefineDialogNew.Open()`, after the Yohara loops, the set id is read as:
`playerm2g2.GetItemSetValue(targetItemPos, i)`.

But the C++ Python binding defines:
- one argument: inventory slot index;
- two arguments: `(window_type, cell)`.

Therefore the normal refine UI interprets the target inventory cell as a **window type**, while stale loop variable `i` becomes the item cell.

For most target inventory positions this creates an invalid `TItemPos` and `CPythonPlayer::GetItemSetValue()` returns 0. If the target happens to be in a numeric cell equal to a valid window enum (notably slot 0 -> INVENTORY), the code instead reads a different cell selected by the stale loop index.

The resulting wrong `set_value` is passed into `toolTip.AddRefineItemData(...)`.

**Impact:** refine preview can hide or display the wrong Set Item identity/effect for a set-marked target. This is presentation-only; server refinement remains authoritative.

**Runtime:** Stage A normal-client preview validation; see `REFCUBE-T12`.

## BUG-REFCUBE-013 — Successful classic refine drops persistent ChangeLook / transmutation metadata

**Status:** VERIFIED STATIC / NORMAL FLOW REACHABLE

`CTransmutation::CanAddItem()` explicitly allows ordinary:
- weapons except arrows;
- body armor.

On successful transmutation, the target item receives:
`left->SetChangeLookVnum(right->GetVnum())`,
and this field is persistent item-instance metadata.

Classic refine success paths create a new result item and call:
`ITEM_MANAGER::CopyAllAttrTo(old, new)`.

`CopyAllAttrTo()` transfers sockets, element state, normal attributes and Yohara random applies, but it does **not** transfer `dwTransmutationVnum / GetChangeLookVnum()`.

Neither `RefineInformation()` nor the mapped `DoRefine()/DoRefineWithScroll()/DoRefineSerpent()` entry validation rejects a target merely because it carries ChangeLook metadata.

Thus a normal refinable weapon/body item can first receive a ChangeLook and then, on a successful refine that creates the next VNUM, lose that appearance metadata with the destroyed source item.

**Impact:** paid/persistent appearance state can be silently lost on successful refinement.

**Ownership note:** the producer belongs to Costume/Appearance, but the destructive metadata-loss boundary is the classic refine transform and is owned here.

**Runtime:** Stage B disposable transformed-item test only; see `REFCUBE-T13`.

## BUG-REFCUBE-014 — Successful classic refine drops persistent Set Item identity

**Status:** VERIFIED STATIC / CURRENT DATA REACHABLE

`set_value` is persistent item-instance metadata:
- stored on `CItem`;
- serialized to DB/player item data;
- used by equipped set-bonus counting;
- exposed to client item packets.

Cube Renewal Set Smith recipes actively assign set ids 1..5 through NPCs 20475..20479.

The tracked `cube.txt` contains **2790 set_value recipes** and includes normal refine families such as:
- Poison Sword 187/188/189 = +7/+8/+9;
- Zodiac Dagger 1187/1188/1189 = +7/+8/+9;
- Zodiac Bow 2207/2208/2209 = +7/+8/+9.

Classic refine creates the next item and calls `ITEM_MANAGER::CopyAllAttrTo()`, but that function never copies `GetItemSetValue()` / `set_value`.

No mapped classic refine gate rejects a set-marked item.

Therefore a set-marked refinable item that succeeds into the next VNUM loses its Set Item identity on the new item.

**Impact:** set membership and consequently equipped set-bonus eligibility can disappear after an otherwise successful refine.

**Runtime:** Stage B disposable set-item refinement; see `REFCUBE-T14`.

## BUG-REFCUBE-015 — Serpent refinement drops random-default base values

**Status:** VERIFIED STATIC / NORMAL SERPENT FLOW

`CRandomHelper::GenerateRandomAttr()` recognizes the current Serpent families, including:
- weapons 360..375, 380..395, 1210..1225, 2230..2245, 3250..3265, 5200..5215, 6150..6165, 7330..7345;
- gloves 23050..23089;
- armor 21310..21406.

For Serpent items it creates persistent `alRandomValues[]` entries from proto socket min/max ranges using `SetRandomDefaultAttr()`.

Those values are not cosmetic. For armor, combat calculation checks `ItemHasRandomDefaultAttr()` and uses `GetRandomDefaultAttr(0)` as the armor value.

`DoRefineSerpent()` creates the next item, calls `ITEM_MANAGER::CopyAllAttrTo()`, then calls `CRandomHelper::RefineRandomAttr()`.

However:
- `CopyAllAttrTo()` does not copy `alRandomValues[]`;
- `RefineRandomAttr()` recalculates only `aApplyRandom[]` and never restores/copies random-default values.

The tracked names show continuous current Serpent refine families such as Snake Sword 360..375 (+0..+15) and Snake Coat 21310..21325 (+0..+15).

**Impact:** a successful Serpent refinement can replace an item carrying generated random-default base values with a new item whose random-default array is zero, changing base combat/stat behavior and permanently losing the rolled values.

**Runtime:** Stage B disposable Serpent item test with before/after random-default capture; see `REFCUBE-T15`.
