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
