# Refine / Cube / Crafting — Verified Bugs

## BUG-REFCUBE-001 — Cube Renewal accepts negative multiplier and can credit Yang/Gem

**Status:** VERIFIED STATIC / MODIFIED-CLIENT ECONOMY BUG

`TSubPacketCGCubeRenwalMake.multiplier` is a signed `int` controlled by the client.

Normal UI constrains its own value, but `CInputMain::CubeRenewalSend()` forwards the received integer directly to `CCubeManager::RefineCube()`.

The server has no `multiplier >= 1` or maximum-domain check.

Preconditions use:
- `CountSpecifyItem(...) < material.count * multiplier`;
- `GetGold() < gold * multiplier`;
- `GetGemPoint() < gem_point * multiplier`.

For a negative multiplier these required values are negative, so ordinary non-negative player balances satisfy the checks.

Currency mutation later uses:
- `PointChange(POINT_GOLD, -(gold * multiplier), false)`;
- `PointChange(POINT_GEM, -(gem_point * multiplier), false)`.

Therefore a negative multiplier reverses the sign and credits recipe currency costs.

The code still removes base recipe materials and uses the normal single reward path, but that does not neutralize the currency-credit defect.

**Impact:** crafted Cube Renewal request can turn positive recipe Yang/Gem costs into player currency gains.

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
