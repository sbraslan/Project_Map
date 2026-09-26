# shop — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-SHOP-001 — Premium shop sale commits before DB stash acknowledgement
- Statik durum: **cross-process atomicity gap doğrulandı**
- Etki koşulu: DB/cache bağlantı veya packet işleme başarısızlığı

Premium `CShop::Buy` sırası:
1. buyer gold/cheque yeterlilik kontrolü
2. buyer destination hesaplama
3. buyer currency debit
4. seller-shop item `RemoveFromCharacter`
5. buyer `AddToCharacter`
6. item `FlushDelayedSave`
7. local shop slot clear/broadcast
8. GAME -> DB `HEADER_GD_SHOP / SHOP_SUBHEADER_GD_BUY(pid,pos)`

Seller proceeds doğrudan GAME'de verilmez. DB `CClientManager::ShopSaleResult` ilgili DB-side shop itemını bulur, stored price/cheque değerini stash'e ekler, itemı DB shop tablosundan kaldırır ve cache'e yazar.

`CShop::Buy` commit öncesinde `db_clientdesc->GetSocket()` kontrolü yapmıyor ve DB tarafında sale result için synchronous acknowledgement beklemiyor. GAME tarafında debit/item transferini geri alan rollback yolu da yok.

Sonuç: DB sale notification işlenmezse item + buyer debit kalıcılaşabilirken seller stash credit'i gerçekleşmeyebilir. Bu durum normal gameplay exploitinden çok availability/persistence bütünlüğü bugıdır.

### BUG-SHOP-002 — Empty Private Shop Search result uses vector[0]
- Statik durum: **doğrulandı**
- Build: `ENABLE_PRIVATESHOP_SEARCH_SYSTEM`

Search response sonunda:
`ch->GetDesc()->Packet(&vecPrivateShopSearchItem[0], sizeof(TPrivateShopSearchItem) * vecPrivateShopSearchItem.size());`

çağrısı vektör boşken de çalışıyor.

`std::vector::operator[](0)` empty vector için undefined behavior'dır; packet length 0 olsa bile pointer ifadesi güvenli değildir.

Beklenen güvenli şekil:
- empty ise yalnız header gönder,
- veya C++11+ `vec.data()` kullan ve transport'ın zero-length semantics'ini açık tut.

Runtime/ASan testi boş search result ile yapılmalı.

### BUG-SHOP-001 — Premium Private Shop stash cap silently clips sale proceeds
- Statik durum: **doğrulandı**
- Build: ENABLE_PREMIUM_PRIVATE_SHOP
- Sınıf: currency cap / asymmetric sale commit

DB ShopSaleResult credits sold.price/sold.cheque through AlterGoldStash/AlterChequeStash. Those functions add then clamp to GOLD_MAX/CHEQUE_MAX.

There is no sale precheck ensuring stash + sale <= cap.

Buyer has already been charged and item ownership already transferred before DB stash credit. Therefore seller can receive only part of the proceeds while sale still completes.

### BUG-SHOP-002 — personal_shop tax not applied to premium private shop stash
- Statik durum: **doğrulandı**
- Build: ENABLE_PREMIUM_PRIVATE_SHOP
- Sınıf: cross-layer accounting mismatch

CShop::Buy computes tax and reduces local dwPrice after buyer was charged. In premium branch seller is not credited from this local dwPrice.

Game sends DB only seller pid + display pos. DB ShopSaleResult reloads cached sold entry and executes AlterGoldStash(sold.price,true), so full listed price enters stash. Net price/tax is absent from packet.

Effect: game-side personal_shop tax calculation does not reduce premium seller stash proceeds.

### OBS-SHOP-001 — item/currency/shop-cache persistence is non-atomic
Premium sale ordering spans three save domains: item FlushDelayedSave, DB shop sale/cache mutation, buyer character delayed Save. A process/connection failure between stages can produce divergent persisted state. Runtime fault-injection is required before classifying a concrete crash-recovery outcome.


## Canonical Shop bug index — 2026-09-26 second pass

> Bu bölüm Shop/Private Shop için canonical numaralandırmadır ve yukarıdaki provisional/çakışan BUG-SHOP numaralarını supersede eder.

### BUG-SHOP-001 — Premium sale cross-process atomicity gap
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP`
- Sınıf: GAME item/currency commit -> DB stash commit arasında transaction/ack eksikliği

Premium `CShop::Buy` buyer debit + item ownership transfer + `FlushDelayedSave` yaptıktan sonra DB'ye yalnız `pid + display_pos` sale bildirimi yollar.
DB `ShopSaleResult` seller stash credit + shop table removal yapar.
GAME tarafı synchronous acknowledgement/rollback beklemez.

DB packet kaybı/crash/peer failure aralığında buyer debit ve item transferi persist olurken seller stash credit'i eksik kalabilir.

### BUG-SHOP-002 — Empty Private Shop Search result vector[0] UB
- Statik durum: **doğrulandı**
- Build: `ENABLE_PRIVATESHOP_SEARCH_SYSTEM`

Search sonucu boşken `&vecPrivateShopSearchItem[0]` ifadesi oluşturuluyor.
Packet size 0 olsa bile empty vector üzerinde `operator[](0)` undefined behavior'dır.
ASan/runtime empty-result testi gerekir.

### BUG-SHOP-003 — Premium personal_shop tax accounting mismatch
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP`

GAME `CShop::Buy` buyer'dan full listed price düşer, ardından `personal_shop` tax hesaplayıp local `dwPrice` değerini net'e indirir.
Premium seller credit bu local net değeri kullanmaz.
DB sale packet yalnız `pid + display_pos` taşır; DB cached `sold.price` üzerinden **full listed price** seller stash'e ekler.

Sonuç: premium private shop satışında GAME'de hesaplanan personal_shop tax seller proceeds'ten düşülmez.

### BUG-SHOP-004 — TransferItemAway off-by-one -> vector OOB
- Statik durum: **doğrulandı**
- Trigger: crafted owner remove-item packet
- Sınıf: bounds / memory safety

`CShop::TransferItemAway(ch, pos, ...)` kontrolü:
`if (pos > m_itemVector.size()) return false;`

Doğru sınır `pos >= size` olmalı.
`pos == m_itemVector.size()` geçer ve hemen ardından:
`SHOP_ITEM& r_item = m_itemVector[pos];`
ile out-of-bounds erişim oluşur.

`TPacketMyShopRemoveItem.slot` int'tir ve caller bunu `uint8_t` olarak geçirir; aktif shop vector size 90 iken slot=90 erişilebilir crafted input'tur.

### BUG-SHOP-005 — Initial MyShop bCount > host max -> transfer-before-validation orphan state
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP + ENABLE_MYSHOP_DECO`
- Aktif limitler: shop grid = 10x9 = 90 hücre; `SHOP_HOST_ITEM_MAX = 80`

`TPacketCGMyShop.bCount` uint8_t ve `CInputMain::MyShop` / `CHARACTER::OpenMyShop` tarafında `bCount <= SHOP_HOST_ITEM_MAX` explicit guard yok.

`CShopManager::CreatePCShop` sırası:
1. `TransferItems(owner,pTable,bItemCount)`
2. `SetShopItems(pTable,bItemCount)`

`SetShopItems` ancak **transferden sonra** `bItemCount > SHOP_HOST_ITEM_MAX` deyip return eder.

Crafted `bCount=81` ile source itemlar PREMIUM_PRIVATE_SHOP window'una taşınıp item save'i flush edilebilir; ardından shop listing vector/table kurulmaz.
Bu, runtime item ile persisted shop metadata arasında orphan/inaccessible item state oluşturabilir.

### BUG-SHOP-006 — Duplicate display_pos during initial shop creation can overwrite/orphan runtime item
- Statik durum: **doğrulandı**
- Trigger: crafted initial MyShop item table

`CHARACTER::OpenMyShop` duplicate **source TItemPos** kontrol eder fakat duplicate `display_pos` kontrol etmez.

Initial `TransferItems` sırasında `m_pGrid->IsEmpty(display_pos,...)` çağrılır fakat bu fonksiyon içinde grid'e `Put` yapılmaz.
Aynı display_pos'a iki source item bu aşamadan geçebilir.

İkinci item `AddToCharacter(PREMIUM_PRIVATE_SHOP, same_cell)` yaptığında `CHARACTER::SetItem` mevcut `pShopItems[cell]` pointer'ını reddetmeden yeni pointer ile overwrite eder.
İlk item owner/window/cell state'ini koruyup runtime lookup'tan kopabilir.

Ardından `SetShopItems` ilk listing sırasında aynı hücrede son overwrite edilen itemı okuyabilir ve ikinci listing grid collision nedeniyle reddedilebilir.
Sonuç: item ownership/listing/shopItems metadata tutarsızlığı ve orphan riski.

### BUG-SHOP-007 — Withdraw stash TOCTOU -> DB stash debit without player credit
- Statik durum: **doğrulandı**
- Build: premium private shop
- Sınıf: async request/response + unchecked void currency mutation

`CInputMain::WithdrawShopStash` request anında:
- requested <= local shop stash
- player gold + requested < GOLD_MAX
- cheque + requested < CHEQUE_MAX
kontrollerini yapar.

DB `WithdrawShopGold` stash'i **önce azaltır** ve success result yollar.

GAME `CInputDB::WithdrawGoldResult` success geldiğinde cap'i tekrar doğrulamaz:
1. local shop stash azaltılır
2. `PointChange(POINT_GOLD,+amount)`
3. cheque için aynı

Request ile response arasında player gold/cheque başka bir yoldan yükselirse `PointChange` overflow'da void early-return eder.
DB stash debit zaten uygulanmış olduğundan ve bu branch rollback yollamadığından gelir kalıcı kaybolabilir.

### OBS-SHOP-001 — stash clamp tek başına normal-flow exploit değil
DB `AlterGoldStash/AlterChequeStash` add sonrası clamp uygular.
Ancak normal open/add akışları `current stash + tüm listed values < cap` invariantını korur.
Dolayısıyla önceki “her satışta cap clipping doğrudan reachable” provisional sınıflandırması **downgrade** edilmiştir.

Clipping hâlâ:
- state desync,
- crafted count/display corruption,
- arithmetic/config edge,
- crash/recovery divergence
sonrası ikincil etki olabilir.

### OBS-SHOP-002 — cross-empire 3x uint32 overflow, default configte dormant
Normal shop buy path'inde başka imparatorluk için `uint32_t dwPrice *= 3` overflow guard olmadan uygulanır.

Server default:
`g_bEmpireShopPriceTripleDisable = true`
yani 3x fiyat varsayılan olarak kapalıdır.

Runtime config ile `SHOP_PRICE_3X_TAX` açılırsa yüksek listed price ×3 32-bit wrap yapabilir.
Premium DB seller credit'i cached original listed price üzerinden yaptığı için bu config kombinasyonu ayrıca ekonomik test gerektirir.

### Shop static second-pass güvenli kapatılanlar
- Initial source window: OpenMyShop yalnız INVENTORY / DRAGON_SOUL_INVENTORY kabul ediyor.
- AddMyShopItem source window: explicit INVENTORY / DRAGON_SOUL_INVENTORY allowlist var.
- PrivateShopSearchBuy: shop existence + editing/closed + visibility + same-map + VIEW_RANGE checks mevcut.
- NPC Sell: active viewed shop must be NPC shop, CanHandleItem/distance/item-lock/seal/anti-sell/gold-cap kontrolleri mevcut.
- Last-item manual remove: TransferItemAway -> CloseMyShop -> Save() ile full closed shop table DB'ye flush ediliyor.


## Shop canonical third-pass update — boot/reload + size-domain audit

### BUG-SHOP-004 reachability refinement
`TransferItemAway` içindeki `if (pos > m_itemVector.size())` off-by-one statik olarak gerçektir; ancak temiz/resmî client state'inde shop display slotları 0..79 olduğu için `pos == 80` bağımsız normal-flow trigger değildir.

Aktif build'de `m_itemVector.size()==80`, fakat PREMIUM_PRIVATE_SHOP item window/grid 90 hücre kabul ettiği için bu bug özellikle BUG-SHOP-008 ile üretilmiş/corrupt 80..89 slot state'inde reachable olur. Bu nedenle BUG-SHOP-004 **dependent memory-safety hardening bug** olarak tutulur.

### BUG-SHOP-008 — 90-slot server grid vs 80-entry shop vector → crafted display_pos OOB
- Statik durum: **doğrulandı**
- Trigger: modified/crafted client
- Sınıf: server-side bounds / memory safety

Aktif sabitler:
- `SHOP_GRID_WIDTH=10`
- `SHOP_GRID_HEIGHT=9`
- `SHOP_INVENTORY_MAX_NUM=90`
- `SHOP_HOST_ITEM_MAX=80`

`CShop::SetShopItems` premium MYSHOP_DECO build'inde:
`m_itemVector.resize(SHOP_HOST_ITEM_MAX)` → 80 entry.

Fakat `CShop::SetShopItem`:
1. `iPos = pTable->display_pos`
2. `m_pGrid->IsEmpty(iPos,...)`
3. `m_pGrid->Put(iPos,...)`
4. `SHOP_ITEM& item = m_itemVector[iPos]`

şeklinde ilerliyor ve `iPos < m_itemVector.size()` kontrolü yok.

CGrid 10x9 olduğu için display_pos 80..89 grid kontrolünden geçebilir; ardından 80-entry vector OOB erişimi oluşur.

**Reachability:**
- `CInputMain::AddMyShopItem` source window için allowlist uygular, fakat `p->targetPos` için 0..79 server bound uygulamaz.
- Packet target `int`, TShopItemTable `display_pos` uint8.
- `TransferItems` 80..89'u 90-slot PREMIUM_PRIVATE_SHOP window'una taşıyabilir.
- hemen sonraki `SetShopItem` vector OOB'a gider.
- Initial MyShop table için de aynı display_pos sınıfı geçerlidir.

Resmî client UI 5x8 = 40 slot/page ve 2 page kullanır; yani normal UI 0..79 üretir. Python/C++ send binding ise arbitrary int target alır ve server limiti enforce etmez.

### BUG-SHOP-009 — ClosePlayerShop Special Inventory preflight/commit mismatch → partial close
- Statik durum: **doğrulandı**
- Build: `ENABLE_SPECIAL_INVENTORY`
- Sınıf: preflight/commit mismatch + non-atomic item recovery

`CInputMain::ClosePlayerShop` tüm non-DragonSoul shop itemlarını önce yalnız dört regular inventory `CGrid` üzerinde simüle eder.

Gerçek transfer loop'unda ise:
`ch->GetEmptyInventory(item)`
kullanılır; Special Inventory itemları Skillbook/Stone/Material domainlerine yönlenebilir.

Sonuçlar:
- regular inventory'de yer var ama ilgili special inventory dolu/locked → preflight true, gerçek transfer daha sonra fail.
- regular inventory dolu ama special inventory boş → false-negative close rejection.

Daha kritik ilk durumda transferler sırayla `TransferItemAway` ile uygulanır ve her item save edilir / DB remove packetleri gönderilir. Sonraki item fail olursa önce taşınan itemlar rollback edilmez; shop kısmen kapanmış/boşalmış state'te kalabilir.

### BUG-SHOP-010 — Shop cache persistence DELETE + INSERT non-transactional crash window
- Statik durum: **doğrulandı**
- Sınıf: DB persistence atomicity / crash recovery

`CShopCache::OnFlush` shop item metadata için ayrı AsyncQuery'ler çalıştırır:
1. `DELETE FROM private_shop_items WHERE pid=...`
2. ayrı `INSERT INTO private_shop_items (...)`

SQL transaction yoktur.

DB process/core crash veya connection failure DELETE uygulandıktan sonra INSERT uygulanmadan gerçekleşirse:
- actual item rows `item.window=PREMIUM_PRIVATE_SHOP` olarak kalabilir,
- fakat fiyat/display metadata `private_shop_items` kaybolabilir.

Boot loader `private_shop_items INNER JOIN item` kullandığı için bu itemlar shop cache reconstruction'a girmeyebilir. Character item load tarafında PREMIUM_PRIVATE_SHOP item row'ları mevcut kalabildiğinden inaccessible/orphan shop-item state oluşabilir.

### OBS-SHOP-003 — MyShopInfoLoad position-index robustness
DB `SendMyShopInfo` active build'de 80-entry price-info domain kullanır ve display position check'i `item.display_pos > SHOP_HOST_ITEM_MAX` şeklindedir; equality 80 reddedilmez.

GAME `MyShopInfoLoad`:
`std::array<TMyShopPriceInfo, SHOP_HOST_ITEM_MAX> info;`
oluşturup bounds check olmadan:
`info[p->pos] = *p`
ve daha sonra
`info[item.pos]`
okur.

Normal official state 0..79 ile çalışır. BUG-SHOP-008 / DB corruption / stale persisted slot 80+ sonrası OOB read/write mümkündür.

Ayrıca `info` value-initialize edilmediği için metadata'sı eksik bir position okunursa POD alanları uninitialized olabilir. Bu gözlem özellikle BUG-SHOP-010 crash-recovery state'i ile birlikte runtime/ASan test edilmelidir.

### OBS-SHOP-004 — Official client Won withdraw width mismatch
Client packet `TPacketCGShopWithdraw.chequeAmount` uint32_t olmasına rağmen:
`CPythonNetworkStream::WithdrawMyShopMoney(uint32_t goldAmount, uint8_t chequeAmount)`
olarak tanımlı.

Python binding `int chequeAmount` okur, sonra bu uint8_t parametreye daralır ve packet'e uint32 olarak yazılır.

Sonuç: resmî client üzerinden 255 üstü Won/cheque withdraw miktarı modulo-256/truncate davranışı gösterebilir. Server crafted packet'te uint32 kabul eder; bu server integrity exploiti değil, client functional bugıdır.

### Shop boot/expiry/static lifecycle closure
- DB boot: `private_shop_items` ile `item(window=PREMIUM_PRIVATE_SHOP)` pid+pos üzerinden JOIN edilir.
- GAME boot: fake CHARACTER oluşturulur, persisted itemlar fake shop window'una bağlanır, sonra `SpawnShop -> CreatePCShop` üzerinden detached shop char'a transfer edilir.
- Expiry/delete: `ITEM_MANAGER::RemoveItem`, PREMIUM_PRIVATE_SHOP itemında `shop->RemoveItemByID` çağırır; runtime vector + owner shopItems + DB remove/closed save zinciri mevcut.
- Last item expiry/removal `CloseMyShop -> Save()` ile full closed table flush yapar.
- Client add/remove bindings target/slot değerlerini server limitine göre sanitize etmez; server validation bu nedenle security boundary olmalıdır.

### Shop static completion note
Premium Private Shop / NPC Shop ana statik haritası bu turla **static completion** seviyesine alınmıştır.

Canonical high-confidence set:
- BUG-SHOP-001..003
- BUG-SHOP-005..010
- BUG-SHOP-004 dependent hardening/reachability
- OBS-SHOP-001..004

Bundan sonraki Shop işi öncelikle runtime/ASan/fault-injection test matrisidir.


## Safebox / Mall — canonical first pass (2026-09-26)
