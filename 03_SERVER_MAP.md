# 03 — Server Map

Server tarafındaki handler, karakter, item, guild ve servis akışları burada tutulur.

## Kayıt formatı

### Sistem
- Packet:
- Entry handler:
- Doğrulamalar:
- Business logic:
- Veri değişimi:
- DB çağrısı:
- Response:
- Edge cases:

## Öncelikli alan
- Guild Storage checkin
- Guild Storage checkout
- Slot / item validation
- Ownership / permission kontrolleri
- Persistence
- Logout / reconnect davranışı

## Guild Storage — doğrulanan server zinciri

### Entry
`CInputMain::Analyze` içinde:
- `HEADER_CG_GUILDSTORAGE_CHECKIN` → `SafeboxCheckin(ch, c_pData, 2)`
- `HEADER_CG_GUILDSTORAGE_CHECKOUT` → `SafeboxCheckout(ch, c_pData, 2)`

Bu sistem ayrı bir storage handler yerine safebox altyapısını `bMall == 2` modu ile yeniden kullanıyor.

### Checkin doğrulamaları
`CInputMain::SafeboxCheckin`:
- character null kontrolü
- GM block kontrolü
- aktif quest kontrolü
- `CanHandleItem()`
- `GetGuildstorage()` varlığı
- kaynak item varlığı
- irremovable / equipped kontrolleri
- hedef slot boşluğu: `pkSafebox->IsEmpty(...)`
- safebox-expand item engeli
- `ITEM_ANTIFLAG_SAFEBOX`
- item lock
- soulbind/seal
- basic item engeli
- belt inventory bağımlılığı

Başarılı işlem:
`RemoveFromCharacter()`
→ quickslot sync
→ `pkSafebox->Add(...)`
→ item log
→ guild log (Guild Renewal aktifse)

### Checkout doğrulamaları
`CInputMain::SafeboxCheckout`:
- character null
- GM block
- `CanHandleItem()`
- `GetGuildstorage()`
- storage slotunda item varlığı
- hedef inventory grid boşluğu
- Dragon Soul hedef alan doğrulaması
- belt inventory doğrulaması
- special inventory type doğrulaması

Başarılı işlem:
`pkSafebox->Remove(...)`
→ `pkItem->AddToCharacter(...)`
→ `ITEM_MANAGER::FlushDelayedSave(pkItem)`
→ `HEADER_GD_ITEM_FLUSH`
→ log / guild log

Guild Renewal aktifse ayrıca son checkout bilgisi güncellenip P2P refresh gönderiliyor.

## Guild Storage — open / lock / close

### Open command
`cmd.cpp`
- `click_guildstorage` → `do_click_guildstorage`

`cmd_general.cpp::do_click_guildstorage`
→ `SetGuildstorageOpenPosition()`
→ `ReqGuildstorageLoad()`

### `CHARACTER::ReqGuildstorageLoad()`
Doğrulamalar:
- guild pointer
- guild storage satın alınmış/seviye > 0
- exchange/shop/mailbox/change-look/safebox/cube çakışması
- `ENABLE_GUILDRENEWAL_SYSTEM` aktifse:
  - guild member alınır
  - `GUILD_AUTH_BANK` yetkisi kontrol edilir
- `pGuild->IsStorageOpen()`
- growth-pet window
- mevcut `GetGuildstorage()`
- 1 saniyelik pulse/rate limit
- açılış noktasına mesafe ≤ 1000
- `m_bOpeningGuildstorage` overlap kontrolü

Başarılı request:
`HEADER_GD_GUILDSTORAGE_LOAD`
→ DB
→ **hemen ardından**
`pGuild->SetStorageState(true, GetPlayerID())`

### Load response
`CInputDB::GuildstorageLoad`
- response guild ID, karakterin güncel guild ID'si ile eşleştiriliyor
- çakışan pencere kontrolü tekrar yapılıyor
- guild storage boyutu guild objesinden alınıyor
- `LoadGuildstorage(...)` çağrılıyor

`LoadGuildstorage`:
- `SetOpenGuildstorage(true)`
- `CSafebox` oluştur/değiştir
- window mode = `GUILDBANK`
- `HEADER_GC_GUILDSTORAGE_OPEN`
- DB'den gelen item'lar local storage'a ekleniyor

### Close
Client:
`/guildstorage_close`

Server:
`do_guildstorage_close`
→ `CloseGuildstorage()`
→ `ch->Save()`
→ tekrar `SetStorageState(false,0)`

`CloseGuildstorage()` zaten kendi içinde de:
- `SetOpenGuildstorage(false)`
- `GetGuild()->SetStorageState(false,0)`
- storage save (SAFEBOX_MONEY build'inde)
- object delete
- client `CloseGuildstorage` command
- load-time reset

Not: close komutunda state reset iki kez çağrılıyor.

## Guild Storage — CSafebox item lifecycle

### Checkin / Add
`CSafebox::Add(pos,item)` GUILDBANK modunda:
1. slot geçerliliği
2. `item->SetWindow(GUILDBANK)`
3. `item->SetCell(openingCharacter,pos)`
4. `item->Save()`
5. `ITEM_MANAGER::FlushDelayedSave(item)`
6. local grid/slot update
7. client'e `HEADER_GC_GUILDSTORAGE_SET`

### Checkout / Remove
`CSafebox::Remove(pos)`:
1. grid occupancy kaldırılır
2. `item->RemoveFromCharacter()`
3. local storage slot temizlenir
4. client'e `HEADER_GC_GUILDSTORAGE_DEL`

Checkout handler daha sonra:
`item->AddToCharacter(...)`
→ save
→ `FlushDelayedSave`
→ `HEADER_GD_ITEM_FLUSH`

### Container destroy
`CSafebox::__Destroy()` mevcut storage itemlarında `SetSkipSave(true)` kullanarak in-memory container kapanışının item silme/save dönüşümüne yol açmasını önlüyor.

## Guild Storage — cross-core state modeli

`CGuild::SetStorageState` yalnız:
- ilgili core'un local `m_data.guildstoragestate/guildstoragewho` alanını değiştiriyor
- SQL `UPDATE guild...` çalıştırıyor.

Guild P2P subheader'larında storage open/close state taşıyan alan bulunmadı.

Mevcut Guild Storage P2P:
- `GUILD_SUBHEADER_GG_REFRESH` → `RefreshP2P`
- `GUILD_SUBHEADER_GG_REFRESH1` → last-checkout bilgisi

Bunlar `guildstoragestate` değerini diğer core'un `CGuild::m_data` nesnesine yazmıyor.

### Startup reset
`main.cpp` içinde her non-auth game server startup:
`guild_manager.InitializeDonate()`

`CGuildManager::InitializeDonate()`:
`UPDATE guild SET guildstoragestate = 0`

Bu global reset tüm guild lock kayıtlarını temizliyor.

## Guild Storage — membership / permission lifecycle

### Yetki değişimi
`ChangeGradeAuth(grade, auth)`:
- DB guild_grade auth günceller
- local `grade_array[grade].auth_flag` günceller
- online guild üyelerine grade auth packet yollar
- **açık Guild Storage sessionlarını kontrol etmez/kapatmaz**

`ChangeMemberGrade(pid, grade)`:
- member grade'i değiştirir
- client/member data update yollar
- **hedef karakter storage açık mı kontrol etmez**

### Üyelikten çıkarma
`CGuild::RemoveMember(pid)`:
- member map'ten silinir
- guild manager unlink
- online character bulunursa:
  - memberOnline'dan çıkar
  - `ch->SetGuild(nullptr)`
- **`CloseGuildstorage()` çağrısı yok**

`CHARACTER::SetGuild(nullptr)` yalnız `m_pGuild` pointer'ını değiştirir ve UpdatePacket yapar.

Sonuç:
`m_pkGuildstorage != nullptr` + `m_pGuild == nullptr` durumu oluşabilir.

### Disband
`CGuild::Disband()` online üyelerde:
`ch->SetGuild(nullptr)`
yapar; açık Guild Storage cleanup yok.

## Inventory / Item Move — server dispatch

`HEADER_CG_ITEM_MOVE`
→ `CInputMain::ItemMove`
→ `CHARACTER::MoveItem(source,destination,count)`

Observer mode'da packet işlenmiyor.

### CHARACTER::MoveItem başlangıç doğrulamaları
- source `IsValidItemPosition`
- source item mevcut
- item exchange'de değil
- requested count source count'u aşmıyor
- extend inventory / IRREMOVABLE koşulu
- item locked değil
- destination `IsValidItemPosition`
- `CanHandleItem()`

Ek sistem kontrolleri:
- Belt Inventory: item tipi + belt varlığı + unlocked cell
- Special Inventory: item special type eşleşmesi
- Switchbot: aktif slot taşınamaz; target item tipi doğrulanır
- equipped source: `CanUnequipNow`
- destination equipment: occupied slot reddedilir ve `EquipItem` çağrılır
- Dragon Soul: özel pull-out/valid-cell kuralları

### Stack
Destination aynı vnum, stackable, anti-stack değil, exchange'de değil ve socketler eşitse:
- count 0 ise source count kullanılır
- count item-limit'e göre clamp edilir
- source `SetCount(old-count)`
- target `SetCount(old+count)`
- her `SetCount` client update + `Save()` yapar.

### Full move
`RemoveFromCharacter()`
→ source slot client-side temizlenir
→ item owner=null, cell=0, window=RESERVED
→ `Save()` ile delayed-save queue'ya eklenir
→ `SetItem(DestCell,item,true)`
→ `SetCell(this,dest)` owner'ı tekrar character yapar
→ destination window atanır
→ destination item packet'i client'e gönderilir.

Burada ikinci explicit `Save()` yoktur; persistence, daha önce queue'ya eklenmiş aynı item pointer'ının son state'i üzerinden gerçekleşir.

### Split
- source `SetCount(source-count)`
- yeni item `CreateItem(vnum,count)`
- sockets kopyalanır
- `AddToCharacter(destination)`
- split log

### Equip / Unequip
Equip:
`MoveItem` → `EquipItem` → `CItem::EquipTo`
→ source remove
→ `SetWear`
→ owner/equipped/cell
→ stat/event refresh
→ `Save()`

Unequip:
`UnequipItem`
→ uygun boş inventory / DS slotu bul
→ `RemoveFromCharacter`
→ `AddToCharacter`
→ `Save()`.

## Inventory — login ItemLoad reconstruction

`CInputDB::ItemLoad`:
1. character/descriptor doğrulaması
2. item daha önce load edilmişse return
3. item count + `TPlayerItem[]` decode
4. `ITEM_MANAGER::CreateItem(vnum,count,id)`
5. `SetSkipSave(true)`
6. sockets/attrs/random/seal/change-look/basic/element/set/pet metadata restore
7. `SetLastOwnerPID(owner)`
8. slot collision kontrolü
9. window'a göre restore:
   - INVENTORY → `AddToCharacter`
   - DRAGON_SOUL_INVENTORY → `AddToCharacter`
   - BELT_INVENTORY → `AddToCharacter`
   - SWITCHBOT → `AddToCharacter`
   - NPC_STORAGE → `AddToCharacter`
   - EQUIPMENT → level check → `EquipTo`
   - ADDITIONAL_EQUIPMENT_1 → `AddToCharacter` + refresh
10. `OnAfterCreatedItem()`
11. `SetSkipSave(false)`

### Collision recovery
DB item hedef slotta başka item varsa veya equipment restore başarısızsa `v` restore listesine alınır.

Sonra:
- uygun boş inventory slotuna `AddToCharacter`
- yoksa `AddToGround`
- ground item için 180 saniye ownership + destroy event.

Son:
- points refresh
- `SetItemLoaded()`.

## Inventory — Pickup

`CInputMain::ItemPickup`
→ `CHARACTER::PickupItem(vid)`.

Temel kontroller:
- PC dead değil
- item VID bulunuyor
- observer mode değil
- item sectree içinde
- `DistanceValid(this)`
- ownership
- bazı quest/pet itemlarında aktif quest kontrolü

`CItem::DistanceValid`:
- character + sectree gerekli
- approximate distance `RANGE_PICK` üstüyse false.

`CItem::IsOwnership`:
- ownership event yoksa herkes alabilir
- event varsa yalnız recorded PID.

Normal pickup:
- stackable ise belt/inventory/special-inventory mevcut stacklere birleştirme denenir
- remainder için boş DS/inventory slotu aranır
- `RemoveFromGround()`
- `AddToCharacter()`
- GET log
- quest pickup callback.

Party dağıtımında ownership sahibi on-map party member bulunup onun inventory'sine verme deneniyor; yer yoksa mevcut picker fallback olabiliyor.

## Inventory — Drop

`CInputMain::ItemDrop2`
→ gold > 0: `DropGold`
→ aksi: `DropItem(Cell,count)`.

`DropItem` kontrolleri:
- `CanHandleItem`
- rate/drop cooldown
- alive
- valid cell/item
- not exchanging
- not locked
- not sealed
- basic-item restriction
- no running quest
- no ANTI_DROP / ANTI_GIVE
- optional GM restriction

Full stack:
`RemoveFromCharacter()`
→ mevcut item ground adayı.

Partial:
- source `SetCount(old-count)`
- source forced flush
- new item created
- sockets copied.

Son:
`AddToGround(currentMap,currentPos)`
→ destroy event
→ dropped item forced save/flush
→ DROP log.

## Inventory — Destroy

`CInputMain::ItemDestroy`
→ `CHARACTER::RemoveItem(Cell,count)`.

Kontroller:
- CanHandleItem
- alive
- valid item
- not exchanging
- not locked
- not sealed
- optional basic-item block
- quest not running
- count > 0

Ardından:
`ITEM_MANAGER::RemoveItem / DestroyItem`
→ owner/container cleanup
→ delayed-save entry erase
→ DB destroy packet
→ ID/VID maps erase
→ `M2_DELETE(item)`.

## CItem::AddToCharacter — target validation

Fonksiyon:
`const uint16_t pos = Cell.cell`
`const uint8_t window_type = Cell.window_type`

Ancak bounds kontrolleri:
- INVENTORY
- EQUIPMENT
- BELT_INVENTORY
- DRAGON_SOUL_INVENTORY
- PREMIUM_PRIVATE_SHOP
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1

için target `pos` yerine **`m_wCell`** kullanıyor.

Yeni item `Initialize()`:
- window = RESERVED
- owner = null
- `m_wCell = 0`

Sonra:
`ch->SetItem(TItemPos(window_type,pos),this,...)`
→ `m_pOwner=ch`
→ `Save()`
→ true.

`SetItem` void döndüğü için target placement başarısız olsa bile AddToCharacter bunu algılayamaz.

### SetItem invalid-cell farkları
INVENTORY/EQUIPMENT/SWITCHBOT/SHOP/ADDITIONAL gibi bazı windowlar array erişiminden önce bounds check yapıyor.

Ancak:
**BELT_INVENTORY**
`pOld = pBeltItems[wCell]`
→ sonra pItem varsa bounds check.

**DRAGON_SOUL_INVENTORY**
`pOld = pDSItems[wCell]`
→ sonra pItem varsa bounds check.

Bu nedenle invalid cell bu iki windowda OOB read/write yoluna girebilir.

### DB load bağlantısı
`CInputDB::ItemLoad` DB'den gelen:
- INVENTORY
- DRAGON_SOUL_INVENTORY
- BELT_INVENTORY
- SWITCHBOT
- NPC_STORAGE

pozisyonlarını doğrudan `AddToCharacter(ch,TItemPos(p->window,p->pos))` yoluna aktarır.

Inventory/Belt collision getter'ları bazı korumalar sunsa da invalid pos DB verisi AddToCharacter katmanında doğru target validation ile reddedilmiyor.

## ENABLE_SWAP_SYSTEM — occupied inventory swap

`MoveItem` destination boş değilse ve stack branch'e girmediyse swap denenebilir.

Normal inventory:
- full item move dışında swap yok
- source/destination default inventory olmalı
- target base `GetItem_NEW(DestCell)`
- target locked/exchanging değil
- optional bind/unbind itemları reddedilir
- destination footprint aynı inventory page içinde kalmalı
- footprintteki itemlar source itemdan büyük olamaz
- toplam occupied+empty footprint tam item size olmalı

Special Inventory:
Aynı mekanizma special-position şartıyla çalışır.
Üst taraftaki special inventory type kontrolü source special item ile destination special type eşleşmesini zorlar.

Mutation sırası:
1. source quickslot mapping kaydedilir
2. source item `RemoveFromCharacter`
3. destination footprint itemları tek tek `RemoveFromCharacter`
4. her destination item source-corresponding hücreye `SetItem`
5. source item destination base'e `SetItem`
6. quickslotlar topluca güncellenir.

Bu akış rollback/transaksiyon kullanmıyor; ancak tüm temel geometry/lock kontrolleri mutation öncesinde yapılmış durumda.

## Switchbot — item registration ve movement

### `CHARACTER::SetItem`, SWITCHBOT
- slot < `SWITCHBOT_SLOT_COUNT`
- old+new aynı anda non-null ise return
- pItem varsa:
  `CSwitchbotManager::RegisterItem(pid,itemID,slot)`
- null ise:
  `UnregisterItem(pid,slot)`
- `pSwitchbotItems[slot]` güncellenir
- ardından item window = SWITCHBOT olur.

### Move guards
`CHARACTER::MoveItem`:
- source SWITCHBOT + active slot → reject
- destination SWITCHBOT + invalid item type → reject.

Valid item:
- weapon
- armor
- opsiyonel costume body/hair/weapon.

### UseItem guard
`CHARACTER::UseItem` SWITCHBOT source ise:
- manager bulunur ve slot active ise false
- boş inventory aranır
- generic MoveItem ile inventory'ye alınır.

Dolayısıyla UI'daki UseItem yolu active slot kilidini bypass etmiyor.

## Switchbot packet dispatch

`HEADER_CG_SWITCHBOT = 171`
→ `CInputMain::Switchbot`.

START:
- base packet size check
- extra payload = `sizeof(alternativeTable) * SWITCHBOT_ALTERNATIVE_COUNT`
- uiBytes yeterli değilse reject
- server tam sabit sayıda alternative parse eder
- `CSwitchbotManager::Start(pid,slot,vec)`.

STOP:
→ `CSwitchbotManager::Stop(pid,slot)`.

### Start server checks
Mevcut:
- slot range
- switchbot object var mı
- slot zaten active mi

Eksik:
- `m_table.items[slot] != 0`
- item ID runtime item manager'da gerçekten var mı
- item owner halen aynı player mı
- item halen SWITCHBOT window/aynı slotta mı
- en az bir alternative configured mı

### Event
`CSwitchbot::Start`
→ 0.2s event.

Her tick:
`SwitchItems()`
→ active slot
→ stored item ID
→ `ITEM_MANAGER::Find(itemID)`
→ item bulunmazsa **continue**, active flag/event değişmez
→ owner null ise tüm tick'ten return
→ hedef attr tamamlandıysa slot inactive/finished
→ kaynak yeterliyse ChangeAttribute + item update.

Bu nedenle active+missing-item state kendi kendini iyileştirmiyor.

## Special Inventory — server modeli

Feature: `ENABLE_SPECIAL_INVENTORY`.

Special Inventory ayrı window enum değil; `INVENTORY` içindeki genişletilmiş cell aralıklarıdır.

Range sırası:
1. Skillbook
2. Stone
3. Material

`INVENTORY_SLOT_COUNT`, special slot end'e kadar büyütülür.

### Item → special type
`CItem::GetSpecialInventoryType()`:
- ITEM_SKILLBOOK → SKILLBOOK
- ITEM_METIN → STONE
- ITEM_MATERIAL / ITEM_RESOURCE → MATERIAL
- VNUM 27987 → MATERIAL
- diğer → -1.

### Position → special type
`TItemPos::IsSpecialInventoryPosition`
→ INVENTORY window + special global range.

`TItemPos::GetSpecialInventoryType`
→ cell'in hangi special subrange'de olduğuna bakar.

### Empty-slot / autogive
`IsEmptySpecialItemGrid`:
- size > 1 → false
- cell kendi type range'i içinde olmalı
- grid boş veya exception item olmalı.

`GetEmptyInventory(LPITEM item)`:
special type varsa search aralığını yalnız o tipe daraltır.

### Move validation
`CHARACTER::MoveItem`:
- normal item special destination'a giremez
- source window INVENTORY ise item special type == destination special type olmalı.

Bu yüzden standard MoveItem yolu yanlış special tab/type placement'ı reddeder.
DB/internal `AddToCharacter` ise item-type ↔ special-cell type eşleşmesini ayrıca kontrol etmez; bozuk persisted row yanlış special subrange'e restore edilebilir. Şimdilik data-integrity gözlemi olarak tutuluyor.

## Switchbot — server lifecycle

Feature: `ENABLE_SWITCHBOT`.
Slot count: 7.

### Move
`CHARACTER::MoveItem`:
- active SWITCHBOT source slotu hareket ettirilemez.
- SWITCHBOT destination itemı `SwitchbotHelper::IsValidItem` ile doğrulanır.
- kabul edilen temel tipler: WEAPON, ARMOR; costume-attr feature altında BODY/HAIR/WEAPON costume.

`SetItem(SWITCHBOT,...)`:
- bounds check
- occupied slot üzerine ikinci itemı reddeder
- add → `CSwitchbotManager::RegisterItem(pid,itemID,slot)`
- remove → `UnregisterItem(pid,slot)`
- runtime item pointer `pSwitchbotItems[slot]`.

### Start/Stop
`CInputMain::Switchbot`
→ packet length check
→ START alternatives parse
→ `CSwitchbotManager::Start`
veya
→ `Stop`.

Start:
- slot bounds
- manager object var mı
- slot zaten active mi
- active=true
- alternatives kopyalanır
- event yoksa `CSwitchbot::Start()`.

Event:
`switchbot_event`
→ `SwitchItems()`
→ active slot item ID lookup
→ target attributes tamamlandıysa slot finish
→ değilse switcher/yang maliyeti
→ `ChangeAttribute()`
→ Switchbot-specific item update.

### Warp/P2P
`CHARACTER::WarpSet`
→ `SetIsWarping(true)`
→ farklı core port ise `P2PSendSwitchbot`.

`P2PSendSwitchbot`:
- event Pause
- source manager map entry erase
- complete table P2P packet ile target port'a gönderilir.

Target:
`CInputP2P::Switchbot`
→ port match
→ `P2PReceiveSwitchbot(table)`.

EnterGame:
- warping=false
- table client'e yollanır
- active slot varsa ve event yoksa Start.

### Statik kusurlar
- Cross-core P2P source path raw manager pointer'ını erase sonrası delete etmiyor → BUG-SWITCHBOT-001.
- Server START item ID/existence/ownership tekrar doğrulaması yapmıyor → BUG-CANDIDATE-SWITCHBOT-002; normal resmi UI boş slot Start'ını disable ediyor.

## Storage checkin/checkout — TItemPos trust boundary

`CInputMain::SafeboxCheckout` personal Safebox, Mall ve Guild Storage için ortak handler.

### Checkout validation
Akış:
client `TItemPos destination`
→ `IsEmptyItemGrid(destination,itemSize)`
→ DS özel kontrolü
→ Belt item-type kontrolü
→ Special Inventory type/range kontrolü
→ storage remove
→ `AddToCharacter(destination)`.

Eksik genel kontrol:
destination window için allowlist yok.

`IsEmptyItemGrid` SWITCHBOT ve ADDITIONAL_EQUIPMENT_1 windowlarını da true döndürebildiği için bunlar checkout target olabilir.

#### SWITCHBOT farkı
Normal `MoveItem`:
`DestCell.IsSwitchbotPosition()`
→ `SwitchbotHelper::IsValidItem(item)`.

SafeboxCheckout:
bu kontrol yok.

Dolayısıyla server packet seviyesinde invalid item type'ın SWITCHBOT slotuna doğrudan yerleşmesi mümkün.

#### ADDITIONAL_EQUIPMENT_1 farkı
`IsEmptyItemGrid` yalnız slot bound + empty pointer kontrolü yapıyor.
Checkout yolunda:
- `CanEquipNow`
- `FindEquipCell`
- page unlock/state
- `EquipTo`
çağrıları yok.

Item `AddToCharacter(... ADDITIONAL_EQUIPMENT_1 ...)` ile doğrudan bu window'a yazılabilir.

### Checkin validation
`SafeboxCheckin`:
client source TItemPos
→ `GetItem(source)`
→ storage restrictions
→ `RemoveFromCharacter`
→ safebox add.

Source window allowlist yok.

Bu nedenle SWITCHBOT source da kabul edilebilir.
Normal MoveItem'ın active-slot guard'ı burada çalışmaz.

## Switchbot Unregister event lifecycle

`CSwitchbot::UnregisterItem(slot)`:
- item=0
- active=false
- finished=false
- alternatives clear.

`CSwitchbotManager::UnregisterItem` sonrasında yalnız update gönderiyor.
`HasActiveSlots()==false` olduğunda running `m_pkSwitchEvent` için Stop/Pause çağrısı yok.

Event callback `SwitchItems()` çağırıp her durumda tekrar schedule edildiği için active slot kalmasa da boş tick devam edebilir.

Character destructor `ClearItem()` çalıştırdığı ve SWITCHBOT itemları RemoveFromCharacter → UnregisterItem zincirinden geçtiği için bu normal logout yaşam döngüsüyle de ilişkilidir.
