# 10 — Unknown Areas

Henüz haritalanmamış veya doğrulanmamış alanlar burada tutulur.

## Guild Storage
- Server packet handler dosyası ve fonksiyonu
- Checkout packet exact adı / struct'ı
- DB persistence zinciri
- Failure / rollback davranışı
- Permission modelinin tam akışı
- Channel / multi-player concurrency etkileri
- Reconnect senaryosu

## Genel
Haritalama sırasında bulunan her bilinmeyen alan önce buraya yazılır; çözüldüğünde ilgili ana dosyaya taşınır.

## Yeni açık alanlar
- Guild Storage açılışını yapan server fonksiyonu
- Açılışta `SetStorageState` / `storagePidOpen` kullanım zinciri
- Guild rank / permission kontrolünün tam konumu
- Aynı storage'a iki oyuncu erişimini engelleyen server-side mekanizma
- `CSafebox::Add/Remove` içindeki DB save zinciri
- Client `SendGuildstorageCheckinPacket / CheckoutPacket` gövdelerinin exact dosya ve send yapısı

## Guild Storage — kalan teknik boşluklar
- `CSafebox::Add`, `Remove`, `Save` içinde GUILDBANK için item window/owner değişiminin exact save yolu
- `ITEM_MANAGER` delayed-save → DB item update zinciri
- Cross-core/channel guild storage state için gizli/başka bir P2P senkronizasyonu var mı: repo genelinde son doğrulama
- `guildstoragestate=1` stale kaldığında startup/reload/reset mekanizması var mı
- Guild üyeliği değişirken açık/pending guild storage davranışı

## Kapatılan alanlar
- `CSafebox::Add/Remove` GUILDBANK lifecycle: doğrulandı.
- delayed save → `HEADER_GD_ITEM_SAVE`: doğrulandı.
- GUILDBANK owner normalization → guild ID: doğrulandı.
- DB `REPLACE item` persistence: doğrulandı.
- checkout `HEADER_GD_ITEM_FLUSH` → cache flush: doğrulandı.

## Hâlâ açık
- Cross-core/channel storage lock için repo-geneli P2P son kontrolü.
- Guild üyeliği/rank değişimi pending/open request sırasında ne oluyor.
- Startup sonrası stale `guildstoragestate` reset mekanizması.

## Çözülen concurrency soruları
- Storage lock P2P senkronizasyonu: incelenen repo kapsamında **yok**.
- Startup stale-state reset: var; `InitializeDonate()` tüm `guildstoragestate` alanlarını 0 yapıyor.

## Kalan Guild Storage soruları
- Guild member guild'den atılırsa/ayrılırsa açık veya pending storage nesnesi nasıl davranıyor.
- Rank yetkisi storage açıkken kaldırılırsa mevcut session item packetleri kabul edilmeye devam ediyor mu.
- Bu statik concurrency açıklarının oyun içi çoklu-core reprodüksiyonu.

## Guild Storage statik haritalama durumu

Artık kapatılan ana alanlar:
- Client UI/binding/send/receive
- game packet dispatch
- open/load/close
- guild permission
- item checkin/checkout
- item persistence
- DB load/save
- lock/state
- cross-core davranış
- disconnect/warp cleanup
- member grade/auth değişimi
- member removal
- guild disband
- startup state reset

## Runtime doğrulama bekleyen kritik alanlar
1. Cross-core simultaneous open
2. Core restart while another core has storage open
3. Permission revoke while session open
4. Member remove/disband while session open
5. Pending-load stuck lock
6. SAFEBOX_MONEY overwrite
7. ItemAward → GUILDBANK isolation
8. Disband orphan items

Bu noktadan sonra Guild Storage için ek salt-okuma getirisi düşük; runtime testleri daha değerli.

## Inventory / Item Move — yeni çalışma alanı

### Çözülen
- Python UI → binding → client network send
- `HEADER_CG_ITEM_MOVE`
- server dispatch
- `CHARACTER::MoveItem` temel validation
- stack
- split
- normal full move
- equip / unequip
- delayed-save persistence mantığı
- quickslot sync ana dalları

### Açık alanlar
- Login sırasındaki item load → inventory/equipment reconstruction
- Pickup / ground ownership
- Drop / destroy
- `ENABLE_SWAP_SYSTEM` edge-case matrisi
- Special Inventory exact type/cell mapping
- Switchbot move lifecycle
- Additional Equipment Page etkileşimi
- invalid destination / rollback davranışlarının bütün caller'larda doğrulanması

## Inventory — load alanı durumu

### Artık çözülen
- DB/cache item query
- `QID_ITEM`
- `RESULT_ITEM_LOAD`
- `HEADER_DG_ITEM_LOAD(42)`
- game `ItemLoad`
- item metadata restore
- inventory/equipment reconstruction
- collision recovery
- no-space → ground recovery
- load sırasında skip-save modeli

### Sonraki açık alanlar
- Pickup
- Drop
- Destroy
- full swap edge cases
- malformed/invalid item position davranışı
- Special Inventory / Switchbot ayrı lifecycle detayları

## Inventory — pickup/drop/destroy durumu

### Çözülen
- pickup packet + server dispatch
- distance + ownership
- stack pickup
- party pickup distribution
- full/partial drop
- ground lifecycle
- DB delete/recreate persistence modeli
- destroy packet → object delete → DB delete

### Bulunan buglar
- destroy use-after-free
- destroy count ignored
- failed AddToGround rollback eksikliği
- destroy sender SendSequence farkı gözlem olarak tutuluyor

### Kalan yüksek değerli alanlar
- `AddToCharacter` target validation
- SwapItem / Additional Equipment
- Special Inventory exact type/cell rules
- Switchbot move lifecycle
- malformed TItemPos ve rollback sınırları.
