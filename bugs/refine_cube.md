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
