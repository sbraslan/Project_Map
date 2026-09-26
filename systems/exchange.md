# exchange

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Player Exchange / Trade static audit başladı

**Tarih:** 2026-09-26

Inventory/Item statik completion sonrasında yeni yüksek öncelikli subsystem olarak player-to-player Exchange seçildi.

### Uçtan uca zincir
Client Python
→ `SendExchangeStartPacket / SendExchangeItemAddPacket / SendExchangeElkAddPacket / SendExchangeAcceptPacket`
→ `HEADER_CG_EXCHANGE`
→ `CInputMain::Exchange`
→ `CExchange::{AddItem,AddGold,Check,CheckSpace,Done,Accept,Cancel}`
→ item RemoveFromCharacter/AddToCharacter + FlushDelayedSave
→ gold/cheque PointChange + character Save.

### Yeni doğrulanmış problemler
- **BUG-EXCHANGE-001:** `CheckSpace()` ile `Done()` special-inventory placement modeli farklı. Ön kontrol regular inventory gridini simüle ediyor; commit `GetEmptyInventory(item)` ile special itemı special tab'a yönlendiriyor. Commit incremental ve rollback yok; bu nedenle ön kontrol true iken gerçek commit ortada fail ederek kısmi trade oluşturabilir.
- **BUG-EXCHANGE-002:** `CheckSpace()` page-4 branch'inde size-boundary `if` sonrasında beklenen `return false` yok. `s_grid4.Put(...)` koşullu statement haline geliyor; çoğu valid item için grid reservation yapılmayabiliyor. Birden çok incoming item aynı boş alanı paylaşmış gibi hesaplanabilir ve Done ortada fail edebilir.
- **BUG-EXCHANGE-003:** Exchange ITEM_ADD source `TItemPos` için semantic window allowlist yok. Client Python binding explicit window_type gönderiyor; server `IsValidItemPosition()` kabul ettiği için SWITCHBOT ve ADDITIONAL_EQUIPMENT_1 itemları crafted client ile trade offer'a sokulabilir.

### Atomicity sonucu
`Accept()` iki taraf için Check/CheckSpace yapıyor, sonra ilk taraf `Done()`, ardından ikinci taraf `Done()` çalışıyor. `Done()` itemları tek tek kalıcı olarak taşır. Herhangi bir sonraki item/currency adımında false dönerse önceki mutationları geri alan transaction/rollback yoktur.

### Sıradaki
1. BUG-EXCHANGE-003'ün Switchbot active-event ve Additional Equipment unequip etkisini sınıflandır.
2. gold/cheque late-overflow ve iki taraflı Done sırasındaki atomicity riskini kapat.
3. disconnect/cancel lifecycle ve item `SetExchanging` cleanup davranışını tara.

## Checkpoint — Exchange/Trade preflight + atomicity audit

**Tarih:** 2026-09-26

Inventory/Item statik completion sonrasında sıradaki subsystem olarak player-to-player Exchange/Trade haritalamasına geçildi.

### Uçtan uca zincir
`root/uiexchange.py`
→ `PythonNetworkStreamModule.cpp`
→ `CPythonNetworkStream::SendExchange*`
→ `HEADER_CG_EXCHANGE / TPacketCGExchange`
→ `CInputMain::Exchange`
→ `CExchange::{AddItem,AddGold,Check,CheckSpace,Accept,Done,Cancel}`.

Official UI item eklerken yalnız `INVENTORY` ve `DRAGON_SOUL_INVENTORY` source üretir. Python binding ise `window_type` değerini doğrudan `TItemPos` içine alır; server `AddItem` tarafı yalnız `IsValidItemPosition()` + `!IsEquipPosition()` kullanır.

### Yeni doğrulanmış buglar
- **BUG-EXCHANGE-001:** `CheckSpace()` special-inventory itemlarını normal inventory gridlerinde simüle ediyor; `Done()` ise `GetEmptyInventory(item)` ile gerçek special inventory type/range'e yönlendiriyor. Preflight true iken commit sırasında space failure oluşabilir. `Done()` itemları tek tek taşıdığı ve rollback yapmadığı için önceki itemlar transfer edilmiş halde kalabilir.
- **BUG-EXCHANGE-002:** extended inventory page-4 branch'inde `s_grid4.Put()` yanlış `if (item->GetSize() > 1 && ...)` gövdesine bağlı. Size=1 itemlar simülasyon gridine hiç rezerve edilmiyor; birden fazla incoming item aynı tek boş slotu varmış gibi kullanabilir. Bu da `CheckSpace()==true` sonrası `Done()` partial transfer üretebilir.
- **BUG-EXCHANGE-003:** `CExchange::AddItem` source window allowlist kullanmıyor. Modified Python/client `SWITCHBOT` ve `ADDITIONAL_EQUIPMENT_1` gibi valid fakat exchange için semantik olarak beklenmeyen source windowları gönderebilir; normal `MoveItem` guardları (active Switchbot / CanUnequipNow vb.) bypass edilir.

### Ek gözlemler
- `ENABLE_CHEQUE_SYSTEM` altında `AddGold` insufficient-funds kontrolü `&&` kullanıyor; tek currency yetersizliği offer aşamasında geçebilir, fakat final `Check()` her currency'yi ayrı doğruladığından statik olarak transaction exploitine dönüşmedi.
- Client `SendExchange*` fonksiyonları `TPacketCGExchange packet;` nesnesini zero-init etmiyor. Subheader'a ait olmayan alanlar wire'a uninitialized gidebilir; server ise switch öncesi `pinfo->arg1` ile character lookup yapıyor. Bu şimdilik client nondeterminism / information-leak observation olarak tutuluyor.

### Sıradaki
1. Exchange accept/Done transaction ordering + rollback eksikliği ayrıntılandır.
2. Currency max/balance recheck'lerini ve PointChange davranışını tara.
3. Cancel / disconnect / death / distance lifecycle'ını kapat.
4. Exchange persistence/DB flush ordering'ini haritala.
5. Ardından runtime test matrisi oluştur.

## Checkpoint — Player Exchange static completion

**Tarih:** 2026-09-26

Player-to-player Exchange subsystem statik olarak completion seviyesine ulaştı.

Canonical bugs: BUG-EXCHANGE-001..004.
Runtime plan: EXCHANGE-T01..T05.

Source repolara değişiklik yapılmadı. Tüm ilerleme yalnız `Project_Map` içine kaydedildi.

Sonraki statik öncelik: Shop / Private Shop ownership-purchase transaction flow; Inventory ve Exchange ile ortak item/gold sınırları nedeniyle sıradaki mantıklı subsystem.

## Checkpoint — Exchange lifecycle / currency / persistence audit

**Tarih:** 2026-09-26

Exchange ikinci statik turu tamamlandı: movement/distance lifecycle, death/disconnect/warp, currency cap TOCTOU ve DB save ordering incelendi.

### Yeni doğrulanmış buglar
- **BUG-EXCHANGE-004:** server exchange mesafesini yalnız START aşamasında kontrol ediyor. Normal movement exchange'i server-side iptal etmiyor ve final ACCEPT'te mesafe tekrar ölçülmüyor. Official UI 1000 mesafeyi aşınca CANCEL gönderiyor; modified client bu client-side korumayı atlayarak uzak mesafede trade'i tamamlayabilir.
- **BUG-EXCHANGE-005:** gold recipient-cap kontrolü ELK_ADD/offer anında yapılıyor. Exchange açıkken ITEM_PICKUP engellenmediği için alıcının gold'u offer sonrası yükselebilir. Done() önce sender'dan gold düşüyor, sonra receiver PointChange(+gold) overflow nedeniyle return edebiliyor. Return değeri olmadığı için Done() bunu başarısızlık olarak görmüyor; sender gold kaybı + success-flow mümkündür.

### Lifecycle sonucu
- Character destruction/disconnect: m_pkExchange->Cancel().
- Death: Dead() içinde exchange cancel.
- Normal movement: cancel yok.
- Normal warp gate: active ENABLE_CHECK_WINDOW_RENEWAL altında SetExchange W_EXCHANGE flag'i set ediyor; CanWarp() W_EXCHANGE açıkken false.
- WarpSet() kendi içinde exchange cancel etmiyor; doğrudan WarpSet çağıran özel yollar ayrıca caller bazında değerlendirilebilir.

### Persistence sonucu
CExchange::Done() her moved item için RemoveFromCharacter -> AddToCharacter(victim) -> ITEM_MANAGER::FlushDelayedSave(item) -> SaveSingleItem -> HEADER_GD_ITEM_SAVE kullanıyor.

Bu nedenle BUG-EXCHANGE-001/002 ile oluşan partial item transfer, final transaction başarıya ulaşmadan DB cache katmanına item ownership/position save olarak gönderilebilir. Exchange için atomik DB transaction/rollback katmanı yok.

Currency CHARACTER::Save() ise delayed-save kuyruğuna girer; item ownership save'i ile iki karakterin currency save'i aynı atomic unit değildir.

### Exchange statik durum
Ana server/client/packet/transaction/lifecycle/persistence haritası tamamlanmaya yakın. Açık kalan başlıca alanlar runtime validation ve birkaç edge-path caller auditidir.

## Checkpoint — Exchange/Trade static completion

**Tarih:** 2026-09-26

Player-to-player Exchange subsystem için ana statik haritalama tamamlandı.

Kapatılan alanlar:
- official UI ve modified-client trust boundary
- CG/GC packet zinciri
- source TItemPos validation
- offer state / accept state
- Check / CheckSpace / Done transaction modeli
- normal + Special Inventory placement
- extended inventory page4 simulation
- Switchbot ve Additional Equipment source edge'leri
- gold / cheque ordering
- distance / movement / death / disconnect / standard warp lifecycle
- per-item DB flush ve character delayed-save persistence.

### BUG-EXCHANGE-003 etki doğrulaması
- Active SWITCHBOT item Exchange AddItem'a modified client ile sokulabilir.
- Item offer'dayken Switchbot event item ID üzerinden çalışmaya devam eder; IsExchanging kontrolü yoktur ve ChangeAttribute() çağırabilir. Böylece karşı tarafın gördüğü initial GC ITEM_ADD attribute snapshot'ı accept öncesinde stale olabilir.
- Transfer anında SWITCHBOT SetItem(nullptr) UnregisterItem çağırdığı için slot transfer sonrasında kapanır; kritik pencere offer→commit arasındadır.
- ADDITIONAL_EQUIPMENT_1 IsEquipPosition() sayılmaz. Exchange AddItem CanUnequipNow çağırmaz. Done içindeki RemoveFromCharacter/Unequip de ITEM_FLAG_IRREMOVABLE kontrolü yapmaz. Bu nedenle normal MoveItem yolunda çıkarılması reddedilecek equipped item Exchange yolu ile transfer edilebilir.

### Statik completion
Exchange için yeni source-code taraması ancak runtime test sonucu veya yeni çapraz subsystem bulgusu gerektirirse açılacak.

Sonraki subsystem: Shop / Private Shop item-transfer ve currency transaction haritası.

## Related
- Bugs: `../bugs/exchange.md`
- Runtime tests: `../tests/exchange.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
