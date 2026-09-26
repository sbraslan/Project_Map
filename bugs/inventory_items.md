# inventory items — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-ITEM-001 — Destroy sonrası use-after-free
- Statik durum: **çok yüksek güven / doğrudan kod yolu doğrulandı**
- Build: `ENABLE_DESTROY_SYSTEM`

`CHARACTER::RemoveItem`:
1. item pointer alınır.
2. `ITEM_MANAGER::RemoveItem(item, "DESTROY")` veya doğrudan `DestroyItem(item)`.
3. `RemoveItem` sonunda `M2_DESTROY_ITEM(item)`.
4. `DestroyItem` sonunda `M2_DELETE(item)`.
5. Caller daha sonra:
   `ChatPacket(..., item->GetName())`
   çağırır.

Sonuç:
Silinmiş C++ nesnesine erişim. Allocator/build/timing'e göre crash, bozuk item adı veya görünürde sorunsuz davranış oluşabilir.

### BUG-ITEM-002 — Destroy packetindeki count yok sayılıyor
- Statik durum: **doğrulandı**

Client:
`SendItemDestroyPacket(Cell, ..., count)`

Packet:
`TPacketCGItemDestroy.count`

Server:
`CInputMain::ItemDestroy`
→ `RemoveItem(Cell, count)`

Ancak `CHARACTER::RemoveItem(..., uint8_t bCount)` içinde `bCount` stack azaltımı için kullanılmıyor.
Item count yalnız `>0` kontrol ediliyor ve ardından item objesi tamamen destroy ediliyor.

Sonuç:
UI/API partial destroy count gönderse bile tüm stack silinir.

### BUG-ITEM-003 — AddToGround başarısızlığında DropItem rollback yok
- Statik durum: **yüksek güven / hata yolu doğrulandı**

Full drop:
- item önce `RemoveFromCharacter()` ile inventory'den çıkarılıyor.

Partial drop:
- source count önce azaltılıyor
- yeni split item yaratılıyor.

Sonra:
`pkItemToDrop->AddToGround(...)`

Eğer bu false dönerse:
- source değişimi geri alınmıyor
- full-drop item envantere geri eklenmiyor
- partial source count geri artırılmıyor
- yaratılan detached item cleanup/rollback yapılmıyor
- fonksiyon sonunda yine `true` dönüyor.

Trigger düşük frekanslı olabilir; `AddToGround` başarısızlığı map index 0, zaten sectree'de olma, owner pointer kalması veya geçersiz sectree/koordinat ile oluşabilir.

### OBS-ITEM-001 — Destroy sender SendSequence çağırmıyor
`SendItemDestroyPacket`, packet `Send` başarılı olduktan sonra diğer item action sender'larının aksine `SendSequence()` çağırmadan true dönüyor.

Şimdilik yalnız gözlem:
Network transport'ın mevcut davranışında bunun gerçek paket kaybı/flush problemi oluşturup oluşturmadığı runtime/transport incelemesi gerektiriyor.

### BUG-ITEM-004 — AddToCharacter yanlış değişkenle target-cell bounds check yapıyor
- Statik durum: **doğrulandı**
- Etki: internal/DB-corruption kaynaklı crash veya inconsistent item state

`CItem::AddToCharacter(ch, Cell)`:
`pos = Cell.cell` alıyor ancak tüm overflow kontrollerinde `pos` yerine **`m_wCell`** kontrol ediyor.

Fresh/detached itemlarda `m_wCell=0` olduğu için invalid target kolayca ilk doğrulamayı geçebilir.

İkinci katman `CHARACTER::SetItem`:
- BELT: `pBeltItems[wCell]` bounds check öncesi erişiliyor.
- Dragon Soul: `pDSItems[wCell]` bounds check öncesi erişiliyor.

Bu iki window için invalid persisted target OOB erişim/core crash üretebilir.

INVENTORY gibi erken-return yapan windowlarda ise başka problem oluşur:
`SetItem` başarısızlığı AddToCharacter tarafından görülemez (void), fakat fonksiyon sonrasında `m_pOwner=ch`, `Save()`, `return true` yapar.

Güvenlik sınırı:
Normal client `CHARACTER::MoveItem` destination'ı önceden doğrular; doğrudan ITEM_MOVE exploit'i olarak işaretlenmemeli. En güçlü mevcut trigger bozuk DB item position veya yanlış internal caller'dır.

### OBS-ITEM-002 — Additional Equipment SwapItem variable shadowing
- Statik durum: **kod kusuru doğrulandı; mevcut çağrı zincirinde doğrudan runtime etkisi bulunamadı**
- Build: `ENABLE_ADDITIONAL_EQUIPMENT_PAGE`

`SwapItem` başında outer:
`TItemPos srcCell(INVENTORY,...), destCell(EQUIPMENT,...)`

oluşturuluyor; if/else içindeki aynı isimli tanımlar inner-scope olduğu için outer değerleri değiştirmiyor.

Buna rağmen mevcut swap akışında gerçek Additional Equipment hedefi daha sonra bağımsız olarak belirleniyor:
- `CheckAdditionalEquipment(wDestCell)`
- `GetAdditionalEquipmentItem(wDestCell)`
- `CItem::EquipTo`
- `GetWear/SetWear`
- `CheckAdditionalEquipment(bWearCell)`

Repo içinde görülen aktif çağrı da inventory itemı mevcut wear slotundaki itemla değiştiren `EquipItem` yoludur.

Bu nedenle shadowing şu an için item duplication/loss/misplacement bugı olarak doğrulanmadı. Refactor sırasında yanlış güvenlik varsayımı yaratabileceği için cleanup/maintainability gözlemi olarak tutulur.

### BUG-ITEM-006 — Storage checkin/checkout window allowlist eksikliği
- Statik durum: **doğrulandı**
- Etki: client-controlled TItemPos ile normal MoveItem semantic guard'larının bypass edilmesi
- Handler: personal Safebox + Mall + Guild Storage ortak yolu

`SafeboxCheckout` destination için `IsEmptyItemGrid` kullanıyor ancak allowed destination window listesi tanımlamıyor.

`IsEmptyItemGrid` şu windowları da kabul ediyor:
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1.

#### SWITCHBOT
Normal MoveItem destination'da:
`SwitchbotHelper::IsValidItem(item)`
zorunlu.

Checkout'ta bu kontrol yok.
Normal/uygunsuz item, slot boşsa doğrudan SWITCHBOT window'una yerleştirilebilir ve manager'a register edilir.

#### Additional Equipment
Checkout'ta:
- CanEquipNow
- FindEquipCell
- EquipTo
- page eligibility
kontrolleri yok.

Doğrudan `AddToCharacter(ADDITIONAL_EQUIPMENT_1,...)` mümkün.

#### Checkin yönü
`SafeboxCheckin` source window için de allowlist uygulamıyor.
SWITCHBOT source item normal MoveItem active guard'ını bypass ederek storage'a alınabilir.

Client binding'in 3-arg formu explicit window_type kabul ettiği için bu yalnız wire-format teorisi değildir; değiştirilmiş Python/client tarafından üretilebilir.

Normal resmi UI davranışı ayrıca test edilmeli; server bug sınıflandırması client'ın dürüst olmasına bağlı olmamalı.

### BUG-ITEM-007 — Special Inventory extend bWindow out-of-bounds
- Statik durum: **doğrulandı**
- Sınıf: client-controlled index / bounds validation eksikliği
- Build: `ENABLE_EXTEND_INVEN_ITEM_UPGRADE_SPECIAL_INV`

Client packet: `TPacketCGSendExtendInvenRequest/Upgrade.bWindow` = `uint8_t`.

Server `CInputMain::ExtendInvenRequest/Upgrade` -> `ExtendSpecialInvenRequest/Upgrade(packet->bWindow)` öncesinde `bWindow < 3` kontrolü yapmıyor.

Character helperları:
- `GetExtendSpecialInvenStage(bPage) -> bSpecialInventoryStage[bPage]`
- `SetExtendSpecialInvenStage(..., bPage) -> bSpecialInventoryStage[bPage] = ...`
- `GetExtendSpecialInvenMax(bPage)` ayrıca local 3-element base array kullanıyor.

Bu nedenle 3..255 special window:
- request/upgrade yolunda OOB read,
- sonraki index calculations'da undefined behavior,
- upgrade başarılı path'e ulaşırsa OOB write
riski oluşturuyor.

Official UI'nin 0..2 üretmesi server trust boundary için yeterli koruma değildir; client Python binding de kendi 0..2 allowlist'ini uygulamıyor.

### BUG-ITEM-008 — Locked Special Inventory slot DB restore
- Statik durum: **doğrulandı**
- Sınıf: persistence / unlock-state invariant bypass
- Doğrudan normal-client trigger: bulunmadı

Normal `MoveItem` locked special target'ı `IsEmptySpecialItemGrid -> GetExtendSpecialInvenMax` ile reddeder.

Fakat DB load: `ItemLoad -> persisted window=INVENTORY,pos -> AddToCharacter -> SetItem` yolunda target special position için character'ın unlocked max'ı zorunlu placement guard değildir.

`IsValidItemPosition(INVENTORY)` tüm static `INVENTORY_SLOT_COUNT` aralığını kabul eder. `SetItem` special branch'i target static special range içindeyse locked max üzerindeki hücreyi kesin olarak reject etmez ve item/grid pointerlarını kurabilir.

Sonuç: malformed, legacy veya başka bir bug tarafından üretilmiş DB row açılmamış special slotta item restore edebilir. Bu durum görünmeyen/erişilemeyen item ve persistence tutarsızlığına dönüşebilir.
