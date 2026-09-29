# 6th/7th Attribute — Bug Registry

**Status:** STATIC COMPLETE / 9 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

## BUG-ATTR67-001 — action packets bypass OPEN/window authorization

**Class:** packet authorization / cross-window state  
**Reachability:** VERIFIED from direct client packet path.

### Proof
1. OPEN sets `IsOpenSkillBookComb` and `W_ATTR_6TH_7TH`.
2. Other server systems use `W_ATTR_6TH_7TH` in mutual-exclusion guards.
3. `CInputMain::Attr67Send` calls SKILLBOOK_COMB, GET_FRAGMENT and ADD handlers directly.
4. Those action branches do not require `IsOpenSkillBookComb()` or `GetOpenedWindow(W_ATTR_6TH_7TH)`.

### Consequence
A modified client can execute Attr67 item mutations without the server-side window state that is supposed to exclude Exchange/Shop/Safebox/Cube-style concurrent activity.

### Deferred validation
`ATTR67-T01`.

## BUG-ATTR67-002 — one skillbook can satisfy the ten-slot combination

**Class:** uniqueness validation / resource accounting  
**Reachability:** VERIFIED through SKILLBOOK_COMB packet.

### Proof
1. Packet contains ten byte-sized cells.
2. `CheckCombStart` fetches each cell independently and increments the counter for every ITEM_SKILLBOOK occurrence.
3. It does not reject duplicate cells.
4. Ten copies of the same valid cell therefore satisfy the ten-entry count.
5. `DeleteCombItems` re-fetches the same cell ten times.
6. First decrement can destroy the one-count item; subsequent iterations see an empty cell.
7. Gold is still deducted and a combination reward is granted.

### Consequence
The intended ten-skillbook cost can be bypassed with one skillbook plus the server gold charge.

### Deferred validation
`ATTR67-T02`.

## BUG-ATTR67-003 — Attr67 can destroy Exchange-held items and leave a dangling exchange pointer

**Class:** cross-system item lifetime / UAF  
**Reachability:** VERIFIED from packet + Exchange state.

### Proof
1. `CExchange::AddItem` stores raw `LPITEM` in `m_apItems[]` and sets `IsExchanging=true`.
2. Attr67 combination does not reject `IsExchanging()`.
3. Attr67 fragment removal also lacks an Exchange exclusion.
4. A one-count referenced item can reach `CItem::SetCount(0)` and be destroyed.
5. Exchange still retains the stale raw pointer.
6. `CExchange::Cancel` later dereferences every non-null `m_apItems[i]` to call `SetExchanging(false)`.

### Consequence
A cross-window crafted sequence can produce use-after-free/server-crash behavior and can affect another player participating in the exchange.

### Deferred validation
`ATTR67-T03`.

## BUG-ATTR67-004 — m_AttrItemAdded can outlive the selected item

**Class:** raw-pointer lifetime / UAF  
**Reachability:** VERIFIED at packet/lifetime level.

### Proof
1. GET_FRAGMENT calls `CheckFragment`.
2. `CheckFragment` stores `LPITEM` in `m_AttrItemAdded`.
3. CLOSE clears only window state, not `m_AttrItemAdded`.
4. `ITEM_MANAGER::DestroyItem` has no mapped cleanup for this raw pointer.
5. Direct GET_FRAGMENT can be sent without OPEN, so normal item operations are not blocked by W_ATTR.
6. Later ADD reads `get_item->GetID()` after only checking that the pointer value is non-null.

### Consequence
Destroying the selected target before a later ADD can leave a dangling pointer that the server dereferences.

### Deferred validation
`ATTR67-T04`.

## BUG-ATTR67-005 — fragment cost commits before additive validation

**Class:** transaction ordering / irreversible partial mutation  
**Reachability:** VERIFIED through ADD packet.

### Proof
1. ADD validates fragment availability.
2. It immediately calls `RemoveSpecifyItem(fragmentVnum, bFragmentCount)`.
3. Only afterwards does it resolve `wCellAdditive`, validate the item, VNUM and count.
4. Any failure in that later block returns without restoring fragments or starting the Attr67 timer/storage operation.

### Consequence
A stale or crafted additive slot can consume valid fragments for no operation result.

### Deferred validation
`ATTR67-T05`.

## BUG-ATTR67-006 — upper endpoint of each skillbook reward range is impossible

**Class:** RNG boundary / reward distribution  
**Reachability:** VERIFIED in combination reward path.

### Proof
1. Each class/group is configured as inclusive-looking integer VNUM endpoints.
2. Code constructs `std::uniform_real_distribution<>(low, high)`.
3. Standard semantics produce reals in `[low, high)`.
4. Result is truncated to `uint32_t`.
5. Therefore `high` can never be generated.
6. DumpProto confirms each high endpoint is a valid distinct skillbook.

### Consequence
Nine configured skillbooks are permanently excluded from random combination rewards.

### Deferred validation
`ATTR67-T06`.

## BUG-ATTR67-007 — no deployed quest caller opens or retrieves the feature

**Class:** deployment integration / missing entry state machine  
**Reachability:** VERIFIED against tracked quest deployment.

### Proof
1. Feature flags compile Attr67 on server/client.
2. Server registers `game.open_skillbook_comb`, `game.open_skillbook_comb_books`, and `game.open_attr67_add`.
3. Attr67 retrieval Lua bindings are compiled.
4. Every one of the 97 source quests named by the deployed `quest_list` was scanned.
5. None calls the Attr67 open or retrieval bindings.
6. Compiled quest-object inventory exposes no Attr67-named quest package.

### Consequence
The stock tracked gameplay deployment has no legitimate quest path to use and complete the system, despite its compiled UI/network/server implementation.

### Deferred validation
`ATTR67-T07`.

## BUG-ATTR67-008 — special skillbook slots 256..359 cannot survive the packet encoding

**Class:** protocol width / inventory integration  
**Reachability:** VERIFIED statically; stock UI route currently shadowed by BUG-ATTR67-007.

### Proof
1. Extended inventory is enabled and page count is 4.
2. Normal inventory size is 180.
3. Skillbook special inventory spans global cells 180..359.
4. Skillbook combination packet stores each cell in `uint8_t bCell[10]`.
5. Client assigns the global Python slot integer directly into that byte.
6. Cells 256..359 truncate modulo 256 before the server receives them.

### Consequence
The latter portion of the skillbook special inventory cannot be represented correctly by this protocol.

### Deferred validation
`ATTR67-T08`.

## BUG-ATTR67-009 — server eligibility ignores existing rare attributes and retrieval can report a false success

**Class:** attribute-state validation / ignored mutation result  
**Reachability:** VERIFIED statically; retrieval consequence is deployment-shadowed by BUG-ATTR67-007.

### Proof
1. Attr67 target validation calls `CItem::GetAttributeCount()` and accepts counts `>= 5 && < 7`.
2. `CItem::GetAttributeCount()` iterates only `MAX_NORM_ATTR_NUM` normal-attribute slots; it does not count the rare 6th/7th slots.
3. An item with five normal attributes and both rare attributes already present therefore still reports normal count 5 and passes server-side Attr67 validation.
4. On a successful retrieval roll, `GetItemAttr` calls `item->AddRareAttribute()`.
5. `AddRareAttribute()` returns `false` when `GetRareAttrCount() >= ITEM_ATTRIBUTE_RARE_NUM`.
6. `GetItemAttr` ignores that boolean return, resets Attr67 time/percent state, sets `*bResult = 1`, and returns the item as though a rare attribute was added.

### Consequence
A fully 7-attribute item can enter the server Attr67 pipeline. If the success roll is reached, the operation can be reported as successful even though no new rare attribute was written. Materials/wait state can therefore be consumed for a no-op mutation.

### Deferred validation
`ATTR67-T09`.

## Deployment-shadowed findings
Not promoted independently while the legitimate quest entry is absent:
- UI displays 1,000,000 Yang for skillbook combination, server uses 100,000.
- UI permits support-only Attr67 submission; server rejects support when fragment count is zero.
- native client range predicates for subheaders are logically impossible conjunctions.
- fixed ten-cell packet builder lacks a Python-list upper-bound check.

## Dormant retrieval candidates
- null dereference in `attr67_get_vnum` when NPC_STORAGE is empty;
- `GetItemAttr` -1 empty-position conversion into `uint16_t`;
- wait-time enforcement depends on caller rather than `GetItemAttr`.
No deployed caller is present in the tracked corpus.

Runtime execution remains locked.

## Static closure
Attr67 static mapping is closed with `BUG-ATTR67-001..009`. NPC_STORAGE persistence/reconstruction and quest-flag persistence were mapped without an additional loss bug. Remaining retrieval hazards stay explicitly dormant because the tracked deployment contains no retrieval caller. Runtime execution remains locked.
