# 00 — Progress / Checkpoint

## Çalışma modu
- Kaynak repolar: **salt-okuma**
- Yazılabilir repo: **sbraslan/Project_Map**
- Amaç: sohbet geçmişine bağımlılığı azaltmak ve kalıcı checkpoint tutmak.

## Ana kaynak repolar
- Project_ClientSrc
- Project_ServerSRC
- Project_Binary
- Project_Game

## Son bilinen çalışma alanı
- Guild Storage client → server akışının haritalanması
- Python → C++ network binding
- Checkin / checkout packet akışları
- Server doğrulama ve persistence zinciri
- Logout / reconnect / edge-case testleri

## Bu dosyada tutulacaklar
- Son incelenen sistem
- Tamamlanan alt akışlar
- Açık sorular
- Sıradaki inceleme hedefi
- Yaklaşık ilerleme durumu

> Not: Yüzdeler yalnızca kapsam tahmini olarak kullanılacak; doğrulanmayan ilerleme yazılmayacak.

## Checkpoint — Guild Storage open/lock path

**Tarih:** 2026-09-26

### Bu tur doğrulananlar
- Client checkin/checkout send fonksiyonlarının exact gövdeleri bulundu.
- Server açılış komutu `click_guildstorage` → `ReqGuildstorageLoad()` doğrulandı.
- `GUILD_AUTH_BANK` yetki kontrolünün açılışta yapıldığı doğrulandı (`ENABLE_GUILDRENEWAL_SYSTEM` altında).
- Guild storage kilidi `guildstoragestate/guildstoragewho` ile tutuluyor.
- DB load yolu `HEADER_GD_GUILDSTORAGE_LOAD (150)` → `QUERY_SAFEBOX_LOAD(...,2)` → `GUILDBANK` item query → `HEADER_DG_GUILDSTORAGE_LOAD (52)` olarak kapatıldı.
- Client kapanış yolu `/guildstorage_close` → `do_guildstorage_close` → `CloseGuildstorage()` doğrulandı.
- Açılış hata/iptal yollarında kilit temizliğiyle ilgili ciddi statik bug yolu bulundu.

### Sıradaki
1. `CSafebox::Add/Remove/Save` ile GUILDBANK item save/delete akışını kapat.
2. Cross-channel lock senkronizasyonunu P2P seviyesinde doğrula.
3. Statik olarak bulunan stuck-lock ve item-award riskleri için oyun içi reprodüksiyon planını netleştir.

## Checkpoint — Guild Storage item persistence tamamlandı

### Yeni doğrulananlar
- `CSafebox::Add` GUILDBANK window + guild storage cell'i item'a yazar ve anında save/flush eder.
- `ITEM_MANAGER::SaveSingleItem` GUILDBANK item owner'ını **guild ID** olarak üretir.
- `HEADER_GD_ITEM_SAVE (30)` DB tarafında GUILDBANK için doğrudan `REPLACE INTO item` yoluna gider.
- Checkout sonrası `HEADER_GD_ITEM_FLUSH (35)` DB item cache varsa zorla flush eder.
- Checkin ve checkout persistence zincirleri artık uçtan uca kapalı.
- `ENABLE_SAFEBOX_MONEY` açık build için Guild Storage kapanışında kişisel safebox gold'unu 0'a yazabilen statik bug doğrulandı.

### Sıradaki
- Cross-core/channel lock için repo-geneli son P2P taraması.
- Guild üyeliği/rank değişimi sırasında açık/pending storage davranışı.
- Sonra Guild Storage haritasını “tamamlandı / oyun içi test bekliyor” durumuna geçirmek.

## Checkpoint — cross-core lock taraması tamamlandı

### Sonuç
Guild Storage open/close lock için repo-geneli P2P taramasında `guildstoragestate/guildstoragewho` değerlerini diğer game core'lara taşıyan bir mesaj bulunmadı.

Bulunan `GUILD_SUBHEADER_GG_REFRESH/REFRESH1` akışları guild UI / son checkout bilgilerini yeniliyor; storage lock state'ini kopyalamıyor.

Ayrıca her non-auth game core başlangıcında `CGuildManager::InitializeDonate()` çağrılıp:
`UPDATE guild SET guildstoragestate = 0`
çalıştırıldığı doğrulandı.

Bu nedenle stale-lock reset mekanizması var; fakat çalışan başka core'daki aktif storage kilidini DB seviyesinde de sıfırlayabildiği için ayrı concurrency riski oluşturuyor.

## Checkpoint — Guild Storage membership/permission lifecycle tamamlandı

### Bu tur doğrulananlar
- `GUILD_AUTH_BANK` yalnız storage açılışında kontrol ediliyor.
- Açık storage üzerindeki checkin/checkout packetlerinde anlık guild üyeliği veya bank yetkisi yeniden doğrulanmıyor.
- `ChangeMemberGrade` ve `ChangeGradeAuth` açık Guild Storage oturumlarını kapatmıyor.
- `RemoveMember` online karakterde doğrudan `SetGuild(nullptr)` yapıyor; açık `m_pkGuildstorage` nesnesini kapatmıyor.
- `SetGuild(nullptr)` yalnız pointer değiştiriyor; storage cleanup yapmıyor.
- Guild disband da online üyelerde `SetGuild(nullptr)` yapıyor ve storage session cleanup yapmıyor.
- Pending Guild Storage load sırasında üyelik kaybı cevabı ID mismatch ile bırakıyor; opening flag / eski guild lock cleanup yok.
- DB guild disband akışında `GUILDBANK` item satırları silinmiyor.

### Yeni yüksek öncelikli bulgular
- Yetki kaldırıldıktan sonra açık Guild Storage erişimi devam edebilir.
- Guildden çıkarılan/disband edilen ve storage açık kalan karakterde null-pointer/core crash yolları var.
- Disband sonrası orphan GUILDBANK item kayıtları kalabilir.

### Sıradaki
Guild Storage için artık ana statik haritalama tamamlanmış kabul edilebilir. Bundan sonraki adım runtime test matrisi ve sonra diğer sistem modüllerine geçiş.

## Checkpoint — Inventory / Item Move temel zinciri başlatıldı

### Kapsam
Guild Storage statik haritalaması tamamlandıktan sonra ortak omurga olan **Inventory / Item Move** sistemine geçildi.

### Bu tur doğrulanan uçtan uca zincir
`Project_Binary/root/uiinventory.py`
→ `__SendMoveItemPacket(...)`
→ Python `m2netm2g.SendItemMovePacket(...)`
→ `Project_ClientSrc/PythonNetworkStreamModule.cpp::netSendItemMovePacket`
→ `CPythonNetworkStream::SendItemMovePacket`
→ `HEADER_CG_ITEM_MOVE = 13`
→ `CInputMain::ItemMove`
→ `CHARACTER::MoveItem`

### MoveItem içinde kapatılan ana dallar
- source/destination validity
- item lock / exchange
- `CanHandleItem()`
- belt inventory kuralları
- special inventory tip kontrolü
- switchbot aktif-item kontrolü
- equip / unequip
- Dragon Soul routing
- stack
- swap
- full move
- split
- quickslot sync

### Persistence sonucu
- stack: `SetCount()` → `Save()` → delayed save
- split: source `SetCount()` + new item `AddToCharacter()`
- full move: `RemoveFromCharacter()` save'i delayed queue'ya item pointer'ını koyar; ardından `SetItem()` aynı item'ın owner/window/cell bilgisini destination'a çevirir. Delayed save çalıştığında **son destination state** kaydedilir.
- equip: `EquipTo()` sonunda `Save()`
- unequip: `AddToCharacter()` sonunda `Save()`

### Sıradaki
- Inventory load/login zinciri
- item pickup/drop/destroy
- special inventory / switchbot sınırları
- swap sisteminin tüm varyantları ve hata senaryoları

## Checkpoint — Inventory login/load zinciri tamamlandı

### DB → game item reconstruction
Player login sırasında itemlar DB/cache'den:
`HEADER_DG_ITEM_LOAD = 42`
ile game'e gönderiliyor.

Game:
`CInputDB::ItemLoad`
→ her `TPlayerItem` için `CreateItem`
→ sockets/attributes/random/seal/change-look/set/growth-pet data uygulanır
→ `SetLastOwnerPID(p->owner)`
→ window'a göre `AddToCharacter`, `EquipTo`, `EquipToDB` vb.

Load boyunca `SetSkipSave(true)` kullanıldığı için mevcut DB itemını tekrar save etme döngüsü engelleniyor; item kurulduktan sonra false'a dönüyor.

Slot çakışması/equip başarısızlığı durumunda item restore listesine alınır:
- boş inventory slotu varsa oraya taşınır
- yer yoksa karakterin bulunduğu yere ground item olarak bırakılır
- 180 sn ownership verilir
- destroy event başlatılır.

Bu bölümle Inventory save ↔ load çift yönlü temel persistence haritası kapanmış oldu.

## Checkpoint — Inventory pickup / drop / destroy tamamlandı

### Bu tur kapatılanlar
- Client pickup/drop/destroy binding ve packet gönderimleri.
- Server dispatch:
  - ITEM_PICKUP
  - ITEM_DROP / ITEM_DROP2
  - ITEM_DESTROY
- Ground ownership + pickup distance.
- Stack-merge pickup.
- Party pickup dağıtımı.
- Full/partial drop.
- Ground item persistence.
- Destroy → DB item delete zinciri.

### Yeni doğrulanan item bugları
- **BUG-ITEM-001:** Destroy sonrası freed item pointer üzerinden `GetName()` çağrısı — use-after-free.
- **BUG-ITEM-002:** Destroy packetindeki `count` server fonksiyonuna kadar geliyor fakat kullanılmıyor; tüm stack siliniyor.
- **BUG-ITEM-003:** `DropItem` içinde `AddToGround()` başarısız olursa source item/count için rollback yok ve fonksiyon yine true dönüyor.

### Persistence özeti
Character item drop:
`RemoveFromCharacter / split`
→ `AddToGround`
→ owner null
→ delayed save / flush
→ `SaveSingleItem`
→ `HEADER_GD_ITEM_DESTROY`
→ DB character item row silinir.

Pickup:
ground runtime item
→ `RemoveFromGround`
→ `AddToCharacter`
→ owner/player window geri atanır
→ `HEADER_GD_ITEM_SAVE`
→ DB row yeniden oluşur/güncellenir.

### Sıradaki
- `AddToCharacter` target-position validation incelemesi
- Swap / Additional Equipment edge-case'leri
- Special Inventory / Switchbot hareket sınırları.

## Checkpoint — AddToCharacter validation ve Swap incelendi

### Yeni doğrulama
`CItem::AddToCharacter(ch, Cell)` target cell'i `pos = Cell.cell` olarak almasına rağmen bounds kontrollerinin tamamında **`pos` yerine mevcut `m_wCell`** değerini kullanıyor.

Fresh item başlangıcı:
- `m_wCell = 0`
- `m_bWindow = RESERVED_WINDOW`

Bu nedenle yeni/detached item için geçersiz destination cell çoğunlukla AddToCharacter'ın ön kontrolünden geçebilir.

`CHARACTER::SetItem` sonraki katmanda çoğu window'u kontrol etse de:
- BELT_INVENTORY: `pBeltItems[wCell]` bounds check'ten önce okunuyor
- DRAGON_SOUL_INVENTORY: `pDSItems[wCell]` bounds check'ten önce okunuyor

Dolayısıyla bozuk target cell OOB erişime dönebilir.

Normal client `MoveItem` yolu `IsValidItemPosition(DestCell)` ile korunuyor. Risk daha çok DB restore / internal caller / bozuk persistence verisi.

### Swap
`ENABLE_ADDITIONAL_EQUIPMENT_PAGE` altında `SwapItem` başındaki `srcCell/destCell` yeniden tanımlamaları inner-scope shadowing nedeniyle outer değişkenleri değiştirmiyor. Bu kod kusuru ayrıca gözlem/bug adayı olarak kaydedildi.

## Checkpoint — ENABLE_SWAP_SYSTEM multi-slot swap kapatıldı

### Sonuç
Normal ve Special Inventory occupied-target swap algoritması incelendi.

Akış:
- destination grid doluysa full-stack zorunlu
- source/destination inventory türü doğrulanır
- target base item `GetItem_NEW` ile bulunur
- self-swap / lock / exchange kontrolleri
- destination footprint içindeki itemlar `moveItemMap` ile toplanır
- toplam footprint `sizeLeft` ile kaynak item boyuna eşitlenir
- source item kaldırılır
- destination item(lar) kaldırılır
- destination itemlar source footprint'e
- source item destination base'e yerleştirilir
- quickslotlar en sonda senkronize edilir

Bu blokta statik olarak doğrudan duplicate yolu bulunmadı.

Persistence:
Her `RemoveFromCharacter()` item pointer'ını delayed-save'e sokar; sonraki `SetItem()` owner/cell/window state'ini değiştirir. Manager save cycle son yerleşimi yazar.

Additional Equipment `SwapItem` shadowing adayı açık kalıyor.

## Checkpoint — Switchbot item lifecycle / event / P2P haritası tamamlandı

### Bu tur doğrulanan zincirler
- Inventory ↔ SWITCHBOT item move
- `SetItem(SWITCHBOT)` → Register/Unregister
- DB persistence: `owner_id=playerID, window=SWITCHBOT, pos=slot`
- login ItemLoad → AddToCharacter → RegisterItem
- START/STOP variable packet parser
- active-slot movement/use protections
- 0.2s switch event
- attribute completion / resource consumption
- inter-core warp transfer
- EnterGame resume

### Yeni bulgular
- **BUG-SWITCHBOT-001:** inter-core warp source tarafında `CSwitchbot*` map'ten erase ediliyor fakat delete edilmiyor.
- **BUG-SWITCHBOT-002:** manager `Initialize()` yalnız raw-pointer map'i `clear()` ediyor; destructor da aynı yolu kullanıyor, owned objects delete edilmiyor.
- **BUG-SWITCHBOT-003:** normal logout/disconnect'te manager cleanup yok. PID'ye ait Switchbot object/event runtime'da kalabiliyor.
- **BUG-SWITCHBOT-004:** START server-side slot item varlığını doğrulamıyor. Empty/stale slot active yapılıp 0.2s event sonsuza kadar dönmeye bırakılabiliyor.
- `TSwitchbotUpdateItem.vnum` server/client'ta `uint8_t`; ancak mevcut client receiver bu alanı kullanmıyor. Şimdilik observation.

### Sıradaki
- Special Inventory type/range modeli
- sonra Item subsystem için genel completion checkpoint.

## Checkpoint — Special Inventory + Switchbot lifecycle

### Special Inventory statik haritası kapatıldı
Special Inventory ayrı bir item window değildir; `INVENTORY` window içinde üç hücre aralığıdır:
- Skillbook
- Stone
- Material

`TItemPos::IsSpecialInventoryPosition()` yalnız INVENTORY + special range kontrolü yapar.
`TItemPos::GetSpecialInventoryType()` hücre aralığından tipi türetir.

Item tarafı:
- `ITEM_SKILLBOOK` → Skillbook
- `ITEM_METIN` → Stone
- `ITEM_MATERIAL` / `ITEM_RESOURCE` → Material
- VNUM 27987 ayrıca Material olarak zorlanır.

`IsEmptySpecialItemGrid` size > 1 itemları reddeder.
`GetEmptyInventory(item)` special item için yalnız kendi special range'ini tarar.
`MoveItem` source INVENTORY olduğunda item special type ile destination special type eşleşmesini zorlar.

### Switchbot move/save lifecycle kapatıldı
Slot sayısı: 7.

Normal UI:
`root/uiswitchbot.py`
→ item INVENTORY ↔ SWITCHBOT için normal `SendItemMovePacket`
→ boş slot veya attribute konfigürasyonu yoksa Start düğmesi disable.

Server:
`MoveItem`
→ active Switchbot source item hareketini reddeder
→ destination SWITCHBOT ise yalnız weapon/armor ve destek açıksa uygun costume tipleri kabul edilir
→ `SetItem(SWITCHBOT,...)`
→ `CSwitchbotManager::RegisterItem/UnregisterItem`.

Persistence:
SWITCHBOT itemları normal player item row'u olarak `window='SWITCHBOT'` ile kaydedilir.
Login item query SWITCHBOT window'u yükler.
`ItemLoad` → `AddToCharacter(SWITCHBOT,pos)` → manager yeniden register edilir.

### Switchbot runtime
Python/C++:
`switchbot.Start(slot)`
→ `HEADER_CG_SWITCHBOT` / START
→ server `CInputMain::Switchbot`
→ `CSwitchbotManager::Start`
→ event
→ `SwitchItems`
→ attribute change
→ özel Switchbot update packet.

Cross-core warp:
`WarpSet`
→ manager warping=true
→ `P2PSendSwitchbot`
→ event Pause
→ table `HEADER_GG_SWITCHBOT` ile target core'a taşınır
→ target `P2PReceiveSwitchbot`
→ EnterGame sırasında event gerekiyorsa yeniden başlatılır.

### Yeni bug / adaylar
- **BUG-SWITCHBOT-001:** `P2PSendSwitchbot` raw `CSwitchbot*` pointer'ını map'ten erase ediyor fakat delete etmiyor → her cross-core transferde memory leak.
- **BUG-SWITCHBOT-004:** server START slotta gerçek item bulunduğunu doğrulamıyor. Normal UI boş slotu engelliyor; custom/malformed client yolu runtime resource testine açık.
- **OBS-SWITCHBOT-002:** client Python binding slot kontrolü `bSlot > SWITCHBOT_SLOT_COUNT`; eşit değer client katmanından geçse de server `ValidPosition` tarafından reddediliyor.

### Sıradaki
- Switchbot cross-core runtime testi
- Additional Equipment SwapItem runtime etkisi
- Item subsystem genel checkpoint ve kalan internal AddToCharacter caller taraması.

## Checkpoint — storage TItemPos caller audit

`AddToCharacter` internal caller taraması storage yollarına genişletildi.

### Yeni doğrulama
Aynı server fonksiyonu:
`CInputMain::SafeboxCheckout`

şu üç kaynaktan çağrılıyor:
- personal Safebox
- Item Mall
- Guild Storage.

Checkout packet içindeki destination `TItemPos` client-controlled.

Server:
`IsEmptyItemGrid(p->ItemPos,...)`
ile occupancy/range kontrolü yapıyor fakat destination window için explicit allowlist uygulamıyor.

`IsEmptyItemGrid` ise:
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1
windowlarını geçerli destination olarak destekliyor.

Sonuç:
normal MoveItem yolundaki bazı semantik kontroller storage checkout'ta atlanabiliyor.

Özellikle:
- SWITCHBOT destination için `SwitchbotHelper::IsValidItem` çağrılmıyor.
- ADDITIONAL_EQUIPMENT_1 destination için equip eligibility / page-state kontrolü yapılmıyor.

Client Python binding de checkout için 3 arg formunda `window_type` değerini doğrudan kabul ediyor.

### Checkin yönü
`SafeboxCheckin` source `TItemPos` için de window allowlist kullanmıyor.
Active SWITCHBOT itemı normal MoveItem engeline uğramadan checkin yoluna sokulabilir.

### Switchbot event detayı
`CSwitchbotManager::UnregisterItem`:
- item ID'yi sıfırlar
- active=false yapar
- config temizler
fakat son active slot kaldırıldığında running event'i Stop etmez.

Bu nedenle logout/ClearItem veya alternatif remove yolu sırasında boş event yaşamaya devam edebilir.

### Yeni kayıtlar
- BUG-ITEM-006: storage checkin/checkout destination/source window validation gap.
- BUG-SWITCHBOT-005: UnregisterItem son active slotta event'i durdurmuyor.

### Sıradaki
- storage-window bypass runtime matrisi
- Additional Equipment özel etkisi
- AddToCharacter kalan internal caller sınıflandırması.
