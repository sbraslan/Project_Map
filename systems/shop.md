# shop

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Shop / Premium Private Shop audit başladı

**Tarih:** 2026-09-26

Exchange static completion sonrasında Shop / Premium Private Shop transaction flow açıldı.

### Haritalanan
- NPC Shop `CShop::Buy`
- ShopEx alternate currency purchase
- Premium Private Shop item ownership transfer
- Premium shop stash sale notification: GAME -> DB -> seller sync
- Private Shop Search buy path
- NPC sell path
- Premium shop open/add-item source-window validation

### İlk sonuçlar
- Buyer destination slot, debit öncesinde gerçek `GetEmptyInventory(item)` / DS helper ile seçiliyor; Exchange'deki CheckSpace/Done divergence burada yok.
- Premium shop açılışı stash + tüm listing fiyatlarını `long long` ile kontrol ediyor.
- Premium add-item source allowlist açıkça INVENTORY / DRAGON_SOUL_INVENTORY ile sınırlı.
- `ENABLE_SHOP_BLACKLIST` bu build'de kapalı.

### Yeni adaylar
- **BUG-SHOP-001:** Premium PC shop sale, buyer debit + item transfer/flush işlemlerini DB shop-sale notification'dan önce kalıcılaştırıyor; Exchange'deki gibi DB-cache socket/ack guard yok. DB notification kaybolursa buyer itemı alıp ödeme yapmışken seller stash güncellenmeyebilir.
- **BUG-SHOP-002:** Aktif Private Shop Search sonuç vektörü boşken `Packet(&vecPrivateShopSearchItem[0], 0)` çağrısı yapılıyor. Empty vector üzerinde `operator[]` UB.

### Sıradaki
1. BUG-SHOP-001 DB disconnect/failure sınırını ve rollback/ack olup olmadığını tamamla.
2. Premium shop stash withdraw rollback akışını haritala.
3. ShopEx ve NPC Sell için source/currency invariantlarını kapat.
4. Shop subsystem completion checkpoint oluştur.

## Checkpoint — Shop / Premium Private Shop first transaction pass

**Tarih:** 2026-09-26

Exchange static completion sonrasında Shop / Private Shop subsystem başlatıldı.

Active build:
- ENABLE_PREMIUM_PRIVATE_SHOP
- ENABLE_PREMIUM_PRIVATE_SHOP_TIME
- ENABLE_OPEN_SHOP_WITHOUT_BAG
- ENABLE_OPEN_SHOP_ONLY_IN_MARKET
- ENABLE_OPEN_SHOP_WITH_PASSWORD
- ENABLE_PREMIUM_PRIVATE_SHOP_TEXTTAIL
- ENABLE_SHOP_NO_SPEND_MIN_IF_ONLINE
- ENABLE_PRIVATESHOP_SEARCH_SYSTEM.

İlk buy transaction zinciri çıkarıldı:
CShopManager::Buy -> CShop::Buy -> buyer balance/space checks -> buyer currency debit -> item owner transfer -> per-item FlushDelayedSave -> game-to-DB SHOP_SUBHEADER_GD_BUY -> DB CClientManager::ShopSaleResult -> shop stash credit + item removal + shop cache update.

### Yeni doğrulanmış buglar
- **BUG-SHOP-001:** Premium shop DB stash için sale-before-cap precheck yok. AlterGoldStash / AlterChequeStash önce tutarı ekleyip sonra GOLD_MAX/CHEQUE_MAX'e clamp ediyor. Stash limite yakınsa buyer tam fiyatı öder ve itemı alır; seller limit üstü geliri sessizce kaybeder.
- **BUG-SHOP-002:** personal_shop tax premium private shop seller stash'ine uygulanmıyor. Game CShop::Buy tax sonrası local dwPrice hesaplıyor fakat DB'ye tax/net price gönderilmiyor; yalnız pid+displayPos gider. DB ShopSaleResult kendi cached sold.price değerini tam olarak stash'e ekler.

### Persistence observation
Buyer debit, item ownership save ve offline-shop sale/cache update tek atomic transaction değildir. Item yeni owner'a FlushDelayedSave edilir; shop sale DB packet'i bunun ardından gider; buyer CHARACTER::Save ise fonksiyon sonunda delayed save'dir. Crash/failure ordering ayrıca test edilmelidir.

Sıradaki Shop adımları: listing/open validation, private-shop search buy path, withdraw/rollback, remove/close/edit ve sell-to-NPC akışları.

## Checkpoint — Shop / Premium Private Shop second pass

**Tarih:** 2026-09-26

### Bu tur kapatılan alanlar
- initial MyShop packet -> OpenMyShop -> SpawnShop -> CreatePCShop
- open-time source window / anti-flag / lock / sealed-item validation
- `TransferItems` ordering
- add-item path + stash projection checks
- owner remove-item -> `TransferItemAway`
- Private Shop Search buy distance/map/state checks
- stash withdraw GAME -> DB -> GAME rollback model
- close/save + last-item behavior
- NPC Sell basic source/currency path

### Yeni yüksek güvenli bulgular
- BUG-SHOP-004: `TransferItemAway` pos==vector.size off-by-one OOB.
- BUG-SHOP-005: initial `bCount > 80` item transferi listing-limit check'ten önce yapılıyor.
- BUG-SHOP-006: initial duplicate `display_pos` runtime shop slot overwrite/orphan state oluşturabiliyor.
- BUG-SHOP-007: async stash withdraw TOCTOU -> DB debit sonrası player credit overflow ile kaybolabilir.

### Düzeltme / yeniden sınıflandırma
Önceki stash-cap clipping provisional bugı normal-flow için downgrade edildi:
open/add akışları stash + remaining listed value invariantını koruyor.
Tax mismatch ve cross-process sale atomicity bugları geçerliliğini koruyor.

### Shop subsystem statik durum
Ana premium/private shop transaction ve lifecycle haritası **completion'a çok yakın**.
Kalan kısa tur:
1. shop boot/reload reconstruction ve fake shop char item bind
2. item expiry / RemoveItemByID lifecycle
3. DB SaveShop/LoadShop cache SQL persistence
4. client packet/binding tarafında add/remove/withdraw wire doğrulaması
5. ardından Shop static completion checkpoint.

## Checkpoint — Shop / Premium Private Shop static completion

**Tarih:** 2026-09-26

Shop final static pass tamamlandı:
- server grid/vector/display position domain
- official client builder/editor slot domain
- add/remove/build/withdraw wire bindings
- bulk close/recovery flow
- DB boot reconstruction
- item expiry -> RemoveItemByID
- cache SQL flush
- player MyShopInfo reload

### Final yeni bulgular
- BUG-SHOP-008: server 90-slot grid vs 80-entry vector, crafted display_pos 80..89 OOB.
- BUG-SHOP-009: bulk close Special Inventory preflight/commit mismatch ve partial transfer.
- BUG-SHOP-010: private_shop_items DELETE+INSERT transaction değil; crash aralığında metadata orphan.
- OBS-SHOP-003: MyShopInfoLoad price position indexing bounds/initialization.
- OBS-SHOP-004: client Won withdraw uint8 parameter vs uint32 packet.

### Client reachability sonucu
Official builder: 5x8 = 40 slot/page, 2 page → 80 slot.
Normal UI 0..79 üretir.
Network/Python bindings arbitrary integer target/slot kabul ettiği için 80..89 memory-safety yolu modified client ile reachable.

### Durum
**Shop / Premium Private Shop: STATIC COMPLETE**

Sonraki adım Project_Map açık alan/checkpoint sırasına göre bir sonraki subsystem'e geçmek; Shop için artık ana iş runtime testleridir.

## Related
- Bugs: `../bugs/shop.md`
- Runtime tests: `../tests/shop.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
