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
