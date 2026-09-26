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

## Inventory validation — yeni durum

### Kapatılan
- `AddToCharacter` bounds logic
- fresh item başlangıç state'i
- `SetItem` invalid target davranışı
- DB ItemLoad ile bağlantısı

### Yeni bug
- BUG-ITEM-004: target yerine old `m_wCell` doğrulaması

### Açık
- Additional Equipment SwapItem shadowing runtime etkisi
- `ENABLE_SWAP_SYSTEM` inventory multi-slot swap algoritması
- Special Inventory exact range/type doğrulaması
- Switchbot move/save lifecycle.

## Swap durumu
- Normal inventory multi-slot swap: statik harita tamamlandı.
- Special Inventory multi-slot swap: statik harita tamamlandı.
- Direkt duplication/loss yolu bulunmadı.
- Additional Equipment `SwapItem` shadowing runtime etkisi açık.
- Runtime stress: farklı item size kombinasyonları ve quickslot senkronizasyonu yine test edilmeli.

### Sıradaki
- Switchbot item move/save lifecycle
- Special Inventory type/range modeli
- ardından Item subsystem genel checkpoint.

## Switchbot statik haritalama durumu

### Kapatılan
- UI item move/use
- active slot protection
- valid item types
- SWITCHBOT SetItem/Register/Unregister
- player-item DB persistence
- login reconstruction
- START/STOP parser
- attribute config payload
- event loop
- item switching/completion
- same runtime manager state
- P2P cross-core transfer
- EnterGame resume
- logout lifecycle incelemesi

### Bulunan
- cross-core transfer object leak
- Initialize/destructor raw pointer leak
- logout cleanup/event leak
- empty/stale-slot START recurring event
- UPDATE_ITEM 8-bit vnum observation

### Kalan Item subsystem
- Special Inventory exact ranges/type mapping
- Additional Equipment runtime swap testi statik olarak açık
- sonra Inventory/Item ana modülü completion checkpoint'e alınabilir.

## Special Inventory / Switchbot — yeni durum

### Special Inventory statik olarak kapatılan
- cell range modeli
- position → type mapping
- item → type mapping
- auto empty-slot search
- size=1 kuralı
- MoveItem type/range enforcement
- login persistence'ın INVENTORY window içinde çalışması.

### Special Inventory açık
- yanlış special subrange persisted row runtime testi
- extend-special-inventory feature kombinasyonlarının ayrı build testi
- gerçek proto içinde special tip olup size>1 olan item var mı veri taraması.

### Switchbot statik olarak kapatılan
- UI item move
- Start/Stop Python/C++ zinciri
- CG/GC packetleri
- server manager
- event loop
- item save/load
- active move lock
- item-type validation
- cross-core P2P state transferi.

### Yeni bulgular
- BUG-SWITCHBOT-001: cross-core source manager pointer leak
- BUG-SWITCHBOT-004: server START item-state revalidation eksikliği
- OBS-SWITCHBOT-002: client slot upper-bound off-by-one
- OBS-SWITCHBOT-001: UPDATE_ITEM vnum 8-bit fakat receiver kullanmıyor.

### Switchbot açık
- cross-core leak runtime ölçümü
- P2PReceive ↔ EnterGame ordering testi
- malformed/custom START resource etkisi
- logout/reconnect ve same-core warp event davranışı.

### Item subsystem sıradaki
1. Additional Equipment SwapItem runtime etkisi
2. AddToCharacter kullanan internal caller'ların son taraması
3. Special Inventory extend-feature build kombinasyonu
4. Item subsystem genel checkpoint
5. sonra bir sonraki ana sisteme geçiş.

## Storage TItemPos caller audit — yeni durum

### Kapatılan
- Safebox/Mall/Guild Storage common checkout handler
- window-aware client Python bindings
- checkout occupancy/type checks
- SWITCHBOT semantic validation farkı
- ADDITIONAL_EQUIPMENT direct placement farkı
- checkin source-window davranışı
- Switchbot Unregister event davranışı
- logout ClearItem bağlantısı.

### Yeni buglar
- BUG-ITEM-006: storage source/destination window allowlist eksikliği
- BUG-SWITCHBOT-005: last-active Unregister event'i Stop etmiyor.

### Açık runtime
- official UI bu window kombinasyonlarını üretebiliyor mu
- modified Python ile server sonucu
- Additional Equipment direct checkout state corruption etkisi
- Switchbot invalid item + START kombinasyonunun gerçek item-type etkisi
- logout sonrası event sayısının profiler ile ölçümü.

### AddToCharacter caller taramasında görülen diğer ana sınıflar
- refine replacement: mevcut validated cell reuse
- pickup: GetEmptyInventory / GetEmptyDragonSoulInventory sonucu
- exchange/shop: precomputed empty position
- quest rewards: empty-position helper
- safebox/guild storage/mall: client TItemPos — **yüksek değerli trust boundary bulundu**.

Bu nedenle BUG-ITEM-004 için en doğrudan crash trigger hâlâ bozuk DB/internal invalid pos; storage yolu ise bounds'tan çok window-semantic bypass sınıfına ayrıldı.


## Canonical checkpoint — Swap/caller audit sonrası

### Statik olarak çözülen
- Additional Equipment `SwapItem` shadowing etkisi: kod kusuru var, fakat mevcut call/control-flow'da doğrudan runtime item placement bugı gösterilemedi.
- `AddToCharacter` kalan ana internal caller sınıfları tarandı.
- Dragon Soul caller target validation'ı doğrulandı.
- exchange/shop/quest/refine/fishing/mining yolları validated/precomputed cell sınıfına alındı.

### Hâlâ gerçek açık alanlar
1. Special Inventory extend-feature build kombinasyonları.
2. Proto/data içinde special-inventory type + size > 1 item olup olmadığı.
3. BUG-ITEM-004 için kontrollü malformed DB row runtime testi.
4. BUG-ITEM-006 storage window bypass runtime testi.
5. Switchbot runtime/profiler testleri.

### Item subsystem durumu
Statik ana akış haritası completion'a çok yakın. Yukarıdaki Special Inventory veri/build taraması bittikten sonra Inventory/Item subsystem genel checkpoint'e alınabilir.


## Canonical checkpoint — Special Inventory extended audit sonrası

### Statik olarak kapatılan
- Aktif Special Inventory + Extend Inventory feature kombinasyonu doğrulandı.
- Static address range ile per-character unlocked max ayrımı haritalandı.
- Normal MoveItem'in locked special destination'ı dynamic max ile reddettiği doğrulandı.
- special stage load/save/client-sync zinciri doğrulandı.
- client -> server extend inventory packet boundary haritalandı.
- special item type source mapping kapatıldı.

### Yeni doğrulanmış buglar
- BUG-ITEM-007: unvalidated special `bWindow` -> OOB array/index access.
- BUG-ITEM-008: persisted DB row -> locked special slot restore invariant bypass.

### Veri nedeniyle açık kalan
Gerçek `item_proto` dataset'i server repo içinde text olarak yok. Project_Binary'de locale başına compiled `item_proto` blob var.
Bu nedenle special-type + `size > 1` gerçek veri kontrolü runtime DB veya unpacked proto export gerektiriyor.

### Inventory/Item subsystem statik durum
Ana static code-flow haritası **completion seviyesine ulaştı**.
Kalan işler artık ağırlıklı runtime doğrulama:
- BUG-ITEM-004 malformed target
- BUG-ITEM-006 storage semantic-window bypass
- BUG-ITEM-007 invalid extend window
- BUG-ITEM-008 locked special DB restore
- Switchbot event/lifecycle profiler testleri
- dataset size audit.

Sonraki statik haritalama turu yeni subsystem'e geçebilir; Item subsystem'e yalnız test sonuçları veya yeni cross-system caller çıktığında geri dönülmeli.


## Exchange / Trade — yeni statik çalışma alanı

Inventory/Item ana statik haritası completion seviyesine alındıktan sonra Exchange subsystem başlatıldı.

### Statik olarak çözülen
- UI → Python binding → C++ send → CG packet → CInputMain → CExchange zinciri
- offer item state / display grid
- dual-accept gate
- `Check` / `CheckSpace` / `Done` ayrımı
- Special Inventory placement-domain mismatch
- extended page4 space-simulation bug
- source TItemPos allowlist eksikliği
- basic currency offer/final-check ayrımı.

### Yeni buglar
- BUG-EXCHANGE-001 — Special Inventory CheckSpace/Done mismatch + partial transfer
- BUG-EXCHANGE-002 — page4 reservation control-flow bug + partial transfer
- BUG-EXCHANGE-003 — SWITCHBOT / Additional Equipment source-window semantic bypass.

### Açık Exchange alanları
1. `Done()` iki taraflı commit sırasının tüm failure noktaları ve rollback etkisi.
2. Gold/Cheque receiver max değerlerinin accept anında yeniden doğrulanması.
3. `PointChange(POINT_GOLD/POINT_CHEQUE)` clamp/overflow semantiği.
4. disconnect / death / warp / distance değişimi sırasında Exchange lifecycle.
5. DB delayed-save / FlushDelayedSave / character Save ordering'i.
6. source-window bypass'ın active Switchbot ve Additional Equipment runtime etkisi.
7. GC exchange state ile client UI cleanup/accept reset senkronu.
8. uninitialized CG packet alanlarının gerçek wire davranışı.

### Sonraki checkpoint hedefi
Exchange transaction/lifecycle haritasını kapat; ardından Shop/Private Shop veya bir sonraki item-transfer subsystemine geç.


## Canonical checkpoint — Player Exchange static audit sonrası

### Statik olarak kapatılan
- client Python send boundary
- CG exchange packet/subheader map
- ExchangeStart session/window/distance guards
- AddItem validation and source TItemPos semantics
- SetExchanging offer lifecycle
- Check + CheckSpace preflight
- Done commit order
- gold/cheque handling
- Accept two-sided sequencing
- Cancel/disconnect cleanup
- item/currency persistence boundary.

### Doğrulanmış Exchange bugları
- BUG-EXCHANGE-001: Special Inventory CheckSpace/Done divergence + partial commit.
- BUG-EXCHANGE-002: inventory page4 reservation control-flow bug + capacity overestimate.
- BUG-EXCHANGE-003: exchange source window allowlist missing; SWITCHBOT/ADDITIONAL_EQUIPMENT_1 reachable by crafted client.
- BUG-EXCHANGE-004: gold overflow TOCTOU; sender debit can survive failed recipient credit. Cheque late-overflow can abort after earlier mutations.

### Observation
- OBS-EXCHANGE-001: cheque-enabled AddGold uses `&&` in insufficient/existing-offer conditions; final Check limits completed-transfer impact.

### Runtime kalan
- EXCHANGE-T01 special inventory partial commit
- EXCHANGE-T02 page4 reservation
- EXCHANGE-T03 unsupported source windows
- EXCHANGE-T04 gold overflow TOCTOU
- EXCHANGE-T05 cheque late overflow rollback.

### Exchange subsystem status
Static code-flow map **completion seviyesinde**. Yeni statik subsystem'e geçilebilir; Exchange'e runtime test sonuçlarında geri dön.
