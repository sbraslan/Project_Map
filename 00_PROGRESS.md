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


## Checkpoint — Additional Equipment SwapItem + AddToCharacter caller audit

**Tarih:** 2026-09-26

### Additional Equipment SwapItem sonucu
`CHARACTER::SwapItem(wCell, wDestCell)` başındaki `srcCell/destCell` yeniden tanımlamaları gerçekten inner-scope shadowing oluşturuyor; outer değerler `INVENTORY/EQUIPMENT` olarak kalıyor.

Ancak mevcut gerçek çağrı zincirinde bu kusurun tek başına yanlış page'e item taşıdığı gösterilemedi:
- repo içinde aktif çağrı `EquipItem/UseItem` yolundan inventory item → wear cell swap'ı,
- gerçek occupied target seçimi `CheckAdditionalEquipment(wDestCell)` ile tekrar yapılıyor,
- Additional page itemı `GetAdditionalEquipmentItem(wDestCell)` ile okunuyor,
- yeni item `CItem::EquipTo` → `GetWear/SetWear` → `CheckAdditionalEquipment(bWearCell)` yoluyla aktif page'e yazılıyor.

Bu nedenle eski `BUG-CANDIDATE-ITEM-005` runtime bug seviyesinden düşürüldü; kod kalitesi / gelecekte refactor riski olarak gözlem tutuluyor.

### AddToCharacter caller audit
Kalan ana caller sınıfları tekrar tarandı:
- Dragon Soul PullOut: `IsValidCellForThisItem` + fallback `GetEmptyDragonSoulInventory`
- refine / fishing rod / mining pick replacement: kaldırılan mevcut itemın daha önce geçerli olan hücresini reuse ediyor
- exchange / shop: önceden hesaplanan boş inventory/DS hücresini kullanıyor
- quest reward/create yolları: `GetEmptyInventory` sonucu kullanıyor
- DB ItemLoad: persisted `window/pos` değerini doğrudan reconstruct ediyor → BUG-ITEM-004 için ana statik risk sınırı
- Safebox/Mall/Guild Storage: client-controlled `TItemPos` → BUG-ITEM-006 semantic-window bypass sınırı

### Sonuç
`AddToCharacter` caller audit statik olarak büyük ölçüde kapandı. Yeni yüksek güvenli caller kaynaklı duplication/loss yolu bulunmadı.

### Sıradaki
1. Special Inventory extend-feature build kombinasyonlarını tara.
2. Proto/data tarafında special-inventory type olup size > 1 item var mı kontrol et.
3. Inventory/Item subsystem completion checkpoint oluştur.


## Checkpoint — Special Inventory extended build / trust-boundary audit

**Tarih:** 2026-09-26

### Aktif build kombinasyonu
Server build'de birlikte aktif:
- `ENABLE_SPECIAL_INVENTORY`
- `ENABLE_EXTEND_INVEN_SYSTEM`
- `ENABLE_EXTEND_INVEN_ITEM_UPGRADE`
- `ENABLE_EXTEND_INVEN_ITEM_UPGRADE_SPECIAL_INV`

Her special type'ın statik address range'i 4 × 45 slotu kapsıyor. Ancak kullanılabilir üst sınır character state içindeki `bSpecialInventoryStage[3]` ile dinamik:
- başlangıç: 45 açık slot
- her stage: +5 slot
- type'lar: Skillbook / Stone / Material

Normal placement:
`MoveItem -> IsEmptyItemGrid -> IsEmptySpecialItemGrid`
zinciriyle `GetExtendSpecialInvenMax(type)` üst sınırını uyguluyor. Kilitli special slot normal MoveItem yolundan reddediliyor.

### Yeni buglar
- **BUG-ITEM-007:** Special Inventory extend request/upgrade paketindeki client-controlled `bWindow` server'da 0..2 doğrulanmadan `bSpecialInventoryStage[bWindow]` ve ilişkili hesaplarda kullanılıyor. OOB read; upgrade akışında koşullar sağlanırsa OOB write riski.
- **BUG-ITEM-008:** DB ItemLoad, persisted INVENTORY position'ı doğrudan `AddToCharacter` ile restore ediyor. `IsValidItemPosition` tüm statik special range'i geçerli sayıyor ve `SetItem` locked-stage sınırını placement engeli olarak uygulamıyor. Malformed/legacy DB row açılmamış special slotta item restore edebilir.

### Proto/data sonucu
Düz metin server-side `item_proto` mevcut repolarda bulunmadı. Project_Binary içinde locale başına derlenmiş `item_proto` blobları var.
Special type source eşlemesi:
- ITEM_SKILLBOOK -> Skillbook
- ITEM_METIN -> Stone
- ITEM_MATERIAL / ITEM_RESOURCE -> Material
- vnum 27987 -> Material

Bu yüzden gerçek dataset içinde special-type + size > 1 item bulunup bulunmadığı GitHub text source'dan güvenilir biçimde doğrulanamıyor; runtime DB/proto export testi olarak bırakıldı.

### Sıradaki
1. BUG-ITEM-007 için izole dev-server boundary testi.
2. BUG-ITEM-008 için controlled DB-row restore testi.
3. Inventory/Item statik haritasını completion checkpoint'e al ve sonraki subsystem'e geç.


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


## Checkpoint — Safebox / Mall static audit başladı

**Tarih:** 2026-09-26

Shop static completion sonrasında personal Safebox / Item Mall subsystemine geçildi.

### Bu tur haritalanan
- official Safebox UI slot events
- Python binding / C++ send boundary
- SAFEBOX_CHECKIN / CHECKOUT / ITEM_MOVE
- MALL_CHECKOUT
- Safebox money deposit/withdraw
- CSafebox Add/Remove/MoveItem/Save/ChangeSize
- DB safebox load/save
- SAFEBOX/MALL account-ID item persistence

### Yeni doğrulanmış buglar
- BUG-SAFEBOX-001: Mall close, Mall gold=0 değerini generic Safebox Save ile persisted safebox.gold üzerine yazıyor.
- BUG-SAFEBOX-002: withdraw signed-int overflow + safebox debit-before-player-credit rollback eksikliği.
- BUG-SAFEBOX-003: crafted occupied-target stack move partial/zero transferde source itemı yanlış kaldırıyor.
- BUG-SAFEBOX-004 / BUG-ITEM-006: semantic source/destination window allowlist eksikliği.

### Official client ayrımı
Official UI safebox->safebox MoveItem packetini boş hedefe count=0 ile yollar. Occupied target stack packetini üretmediği için BUG-SAFEBOX-003 normal UI değil modified-client/server-validation sınıfıdır.

### Sıradaki
1. malformed/overlap DB load + grid reconstruction
2. SAFEBOX/MALL item expiration/removal
3. ItemAward placement/account-window isolation
4. packet width/slot boundaries
5. logout/close persistence ordering


## Checkpoint — Safebox / Mall second pass + active build classification

**Tarih:** 2026-09-26

### Build durumu
`ENABLE_SAFEBOX_IMPROVING` açık; `ENABLE_SAFEBOX_MONEY` kapalı.

Bu nedenle ilk turdaki money bugları mevcut binary/server build için runtime önceliğinden çıkarıldı:
- BUG-SAFEBOX-001 dormant
- BUG-SAFEBOX-002 dormant

### Bu tur kapatılan
- malformed/overlap DB load + CGrid semantics
- SAFEBOX/MALL item expiry/remove
- ItemAward Safebox-vs-Mall routing
- packet source-slot width
- logout/disconnect CloseSafebox/CloseMall ordering
- stack-loss persistence etkisi
- Switchbot / Additional Equipment semantic checkout reachability

### Yeni aktif bug
**BUG-SAFEBOX-005**:
Persisted invalid/overlap item rowunda CSafebox::Add CGrid::Put(false) sonucunu yok sayıyor. Runtime pointer grid reservation olmadan kuruluyor. Bottom-boundary multi-size state sonradan Remove edildiğinde CGrid::Get height bounds kontrolü yapmadan grid buffer dışına yazabilir.

### Safebox/Mall statik durum
Ana client -> game -> DB -> reload -> close/expiry akışı **static completion'a yakın**.

Kalan kısa statik tur:
1. safebox password/load-request lifecycle + pending-load state
2. account-level same-safebox multi-character/concurrent-open davranışı
3. channel/warp/open-window cleanup
4. ardından Safebox/Mall STATIC COMPLETE checkpoint.
