# Gem Shop — Static System Map

**Status:** STATIC MAPPING OPEN  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source snapshot:** pinned by `MAP_STATE.json`

## Canonical scope
Gem Shop owns an independent persistent economy lifecycle:
- dedicated `HEADER_CG_GEM_SHOP` and GC open/refresh/buy/add packet families;
- `POINT_GEM` currency spend path;
- per-character persistent `gemItems[]` slot state;
- persistent `gem_next_refresh`;
- random row-based offer generation;
- refresh-item consumption;
- slot unlock item consumption;
- purchase -> item creation -> inventory placement -> Gem debit -> slot consume;
- DB player load/save of Gem Shop state.

## Deployment proof
- `ENABLE_GEM_SYSTEM` and `ENABLE_GEM_SHOP` are enabled on the pinned snapshot.
- `CInputMain::GemShop()` handles BUY / ADD / REFRESH.
- `CHARACTER` owns `m_gemItems`, refresh time and temporary all-slots-unlocked state.
- DB player persistence includes `gem_items` and `gem_next_refresh`.
- `CShopManager` owns the Gem Shop table and row-based random item selection.
- Client/Binary provide dedicated Gem Shop protocol/UI surfaces.

## Boundary
Owned here:
- Gem Shop offers, refresh, slot unlock, Gem-cost purchase and persistent shop state.

Not owned here:
- Premium Private Shop and Private Shop Search: already CLOSED under canonical `shop`.
- player Exchange: already CLOSED under canonical `exchange`.
- Won/Cheque currency transfer inside Shop/Exchange and generic player-point persistence.
- generic inventory item placement itself.

## Audit cursor
1. Table initialization / random-row generation / persisted item-id validity.
2. BUY transaction ordering, item creation, inventory routing and Gem debit.
3. REFRESH / ADD item consumption and state persistence.
4. login/load/save/reset and time-refresh lifecycle.
5. client/server packet parity.
6. close subsystem and return to discovery queue.
