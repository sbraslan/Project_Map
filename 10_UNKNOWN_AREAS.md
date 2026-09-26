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


## Exchange / Trade — lifecycle/persistence ikinci tur durumu

### Yeni kapatılan statik alanlar
- death cancellation
- character teardown/disconnect cancellation
- normal movement davranışı
- final distance enforcement
- active W_EXCHANGE / CanWarp ilişkisi
- gold receiver-cap offer/final ordering
- ITEM_ELK pickup sırasında exchange state davranışı
- per-item FlushDelayedSave persistence
- character currency delayed-save ordering.

### Yeni buglar
- BUG-EXCHANGE-004 — final distance recheck yok; official UI koruması server-side invariant değil.
- BUG-EXCHANGE-005 — gold cap TOCTOU; sender debit receiver credit olmadan kalabilir.

### Severity güncellemesi
BUG-EXCHANGE-001 ve BUG-EXCHANGE-002 partial item transferleri FlushDelayedSave nedeniyle DB cache'e taşınabilir; persistence impact doğrulandı.

### Hâlâ açık
1. BUG-EXCHANGE-003 active Switchbot ve Additional Equipment runtime etkisi.
2. Uninitialized CG Exchange packet alanlarının wire/runtime sonucu.
3. SendExchangeItemDelPacket client tarafında assert-only olmasının gerçek UI/feature etkisi.
4. Direct WarpSet çağıran ve CanWarp bypass eden özel callerların exchange açısından hızlı caller audit'i.
5. Cheque balance'ın exchange açıkken değişebildiği tüm dış yollar.
6. Runtime test EX-T01..EX-T09.

### Sonraki mantıklı adım
Exchange kalan edge caller auditlerini kısa turda kapat, subsystem için statik completion checkpoint oluştur; ardından Shop/Private Shop item-transfer subsystemine geç.


## Exchange static completion checkpoint

Exchange subsystem ana statik haritası **completion** seviyesine alındı.

Runtime'a bırakılanlar:
- EX-T01 Special Inventory partial transfer
- EX-T02 page4 partial transfer
- EX-T03 active Switchbot source
- EX-T04 Additional Equipment source
- EX-T05 uninitialized packet wire observation
- EX-T06 remote distance accept
- EX-T07 gold cap TOCTOU
- EX-T08 persistence
- EX-T09 lifecycle sanity.

Non-blocking observations:
- client SendExchangeItemDelPacket assert-only; official UI item-removal akışı görünmüyor, cancel ile çıkılıyor.
- direct WarpSet callerları yeni subsystem taramalarında karşılaşılırsa Exchange invariantı açısından tekrar işaretlenecek.
- cheque dış mutation yolları yeni caller bulunduğunda incelenecek.

### Sıradaki statik subsystem
Shop / Private Shop: buy/sell, owner/guest state, item reservation, currency debit-credit, DB persistence ve premium private shop ayrımları.


## Shop / Premium Private Shop — first pass

### Mapped
- active premium/offline/search feature flags
- CShopManager::Buy distance/closed gate
- CShop::Buy funds + inventory + transfer ordering
- per-item FlushDelayedSave
- game -> DB sale packet
- DB ShopSaleResult stash/item/cache update
- stash max behavior
- personal_shop tax split across game and DB.

### New bugs
- BUG-SHOP-001 — stash cap silent proceeds clipping
- BUG-SHOP-002 — premium private shop personal_shop tax accounting mismatch.

### Open next
1. shop listing/open validation and client-controlled price/count/display position
2. premium TransferItems source-window validation
3. private-shop search remote-buy path and distance/guest bypass semantics
4. withdraw + rollback game/DB handshake
5. remove/edit/close item return path
6. NPC Sell source TItemPos/special inventory behavior
7. crash/recovery ordering across item/player/shop cache.


## Shop / Premium Private Shop — second pass sonrası kalanlar

### Statik olarak çözülen
- initial OpenMyShop validations
- source window allowlist
- add-item source checks
- remove-item transfer path
- PrivateShopSearchBuy same-map/range enforcement
- stash withdraw request/result/rollback
- NPC Sell temel zinciri
- close/save last-item davranışı

### Runtime / son statik açık
1. BUG-SHOP-004 slot==size crafted remove packet testi.
2. BUG-SHOP-005 bCount=81 kontrollü test ve item recovery sonucu.
3. BUG-SHOP-006 duplicate display_pos kontrollü test.
4. BUG-SHOP-007 withdraw request-response arasında gold/cheque mutation testi.
5. empty search result ASan testi.
6. GAME<->DB sale fault-injection.
7. DB boot/load shop reconstruction + item bind son turu.
8. item expiration -> RemoveItemByID -> DB consistency.
9. runtime configte SHOP_PRICE_3X_TAX açılırsa high-price overflow testi.


## Shop static completion sonrası yalnız runtime doğrulama

Shop ana kod haritası kapatıldı. Kalan runtime testleri:
- SH-T01: AddMyShopItem targetPos 79/80/89 sınırları (ASan tercih).
- SH-T02: initial MyShop display_pos 80 ve duplicate display_pos.
- SH-T03: bCount 80/81 boundary.
- SH-T04: ClosePlayerShop regular inventory available + special tab full partial-close.
- SH-T05: withdraw request sonrası gold/cheque değiştirip DB response TOCTOU.
- SH-T06: empty private-shop-search result ASan.
- SH-T07: GAME->DB sale packet fault injection.
- SH-T08: shop cache flush DELETE sonrası crash / INSERT öncesi recovery.
- SH-T09: persisted missing price metadata / pos=80 MyShopInfoLoad ASan.
- SH-T10: official client cheque withdraw 255/256/300 behavior.

Statik Shop keşfi için yeni açık alan kalmadı; yeni Shop çalışması bu testlerden bulgu çıkarsa açılmalı.


## Safebox / Mall — first pass sonrası açık alanlar

### Statik olarak çözülen
- checkin/checkout primary flow
- account-ID item persistence
- DB safebox/mall load
- money save/withdraw primary flow
- Mall close money overwrite
- safebox internal stack branch
- explicit TItemPos trust boundary

### Runtime testleri
- SAFEBOX-T01: Mall aç/kapat öncesi ve sonrası persisted safebox.gold.
- SAFEBOX-T02: player gold ~1.5B + withdraw >=0.7B cap/rollback.
- SAFEBOX-T03: crafted occupied-stack partial merge.
- SAFEBOX-T04: full destination stack + count=0 crafted move.
- SAFEBOX-T05: checkout -> SWITCHBOT / ADDITIONAL_EQUIPMENT.
- SAFEBOX-T06: unsupported source window -> checkin.

### Kalan kısa statik tur
1. malformed/overlapping DB row -> CSafebox::Add grid behavior
2. SAFEBOX/MALL item expiration/removal
3. ItemAward full/duplicate placement and isolation
4. packet slot/count narrowing
5. disconnect/logout/CloseSafebox/CloseMall persistence ordering


## Safebox / Mall — second pass sonrası kalanlar

### Aktif runtime testleri
- SAFEBOX-T03: crafted occupied-stack partial merge; source remainder DB sonucu.
- SAFEBOX-T04: full destination stack + count=0 crafted move.
- SAFEBOX-T05: checkout destination SWITCHBOT.
- SAFEBOX-T06: checkout destination ADDITIONAL_EQUIPMENT_1.
- SAFEBOX-T07: SWITCHBOT source -> safebox checkin.
- SAFEBOX-T08: malformed persisted multi-size item bottom-boundary; open + remove under ASan.
- SAFEBOX-T09: overlapping / duplicate persisted positions; open/close/reload consistency.

### Build-dependent, şimdilik çalıştırma
`ENABLE_SAFEBOX_MONEY` kapalı olduğu için:
- eski SAFEBOX-T01 Mall gold overwrite
- eski SAFEBOX-T02 gold withdraw overflow
mevcut build'de applicable değildir. Feature açılırsa tekrar aktive edilmeli.

### Kalan statik
1. password/load request pending-state lifecycle
2. aynı account safebox'ının iki karakter/core üzerinden concurrent open ihtimali
3. warp/channel-change ve open safebox cleanup
4. static completion checkpoint.


## Safebox / Mall — static completion checkpoint

Safebox/Mall statik keşfi kapatıldı.

### Runtime-only kalan
- SB-T01 crafted partial stack merge.
- SB-T02 full destination stack + count=0.
- SB-T03 checkout -> SWITCHBOT.
- SB-T04 checkout -> ADDITIONAL_EQUIPMENT_1.
- SB-T05 SWITCHBOT source -> checkin.
- SB-T06 malformed bottom-boundary persisted item under ASan.
- SB-T07 duplicate/overlap persisted positions.
- SB-T08 force DB safebox item-query failure -> opening flag recovery.
- SB-T09 Mall remote/open-during-other-window gameplay policy.
- SB-T10 same-account reconnect race only if global login layer permits duplicate active session.

Money tests yalnız ENABLE_SAFEBOX_MONEY yeniden açılırsa uygulanmalı.


## Mailbox — first pass sonrası açık alanlar

### Runtime kritik
- MAIL-T01: negative iYang packet; sender gold delta.
- MAIL-T02: negative iWon packet; sender cheque delta.
- MAIL-T03: direct WRITE without prior CHECK_NAME.
- MAIL-T04: > MAILBOX_MAX_MAIL direct write flood.
- MAIL-T05: keep mailbox open across DB backup sort/erase, then claim by old index.
- MAIL-T06: large Yang attachment around 430M+ tax arithmetic.
- MAIL-T07: source SWITCHBOT / ADDITIONAL_EQUIPMENT_1 send.
- MAIL-T08: DB process crash before periodic backup after successful send.

### Kalan statik
- backup TRUNCATE/INSERT transaction safety
- SQL string escaping for title/message/from/name
- packet fixed-char NUL termination
- expiry/delete/confirm index behavior
- boot loader
- messenger/block enforcement
- warp/logout/system-close lifecycle.


## Mailbox — static completion sonrası runtime test matrisi

- MAIL-T01 negative iYang -> sender gold delta.
- MAIL-T02 negative iWon -> sender cheque delta.
- MAIL-T03 direct WRITE without confirm / nonexistent target.
- MAIL-T04 mailbox >90 flood and client load behavior.
- MAIL-T05 open snapshot + incoming mail + periodic backup sort -> old index GET.
- MAIL-T06 local delete/expired erase + backup -> index shift.
- MAIL-T07 430M+ / 1B+ Yang receive tax/cap arithmetic.
- MAIL-T08 source SWITCHBOT.
- MAIL-T09 source ADDITIONAL_EQUIPMENT_1 / irremovable item.
- MAIL-T10 GAME write then DB process crash before backup.
- MAIL-T11 DB restart with pre-populated mailbox SQL table; verify InitializeMailBoxTable early return.
- MAIL-T12 kill DB during TRUNCATE->INSERT backup.
- MAIL-T13 apostrophe/quote in title/message during backup.
- MAIL-T14 crafted non-NUL packet string under ASan.
- MAIL-T15 receiver grant with DB GET fault injection.
- MAIL-T16 same-process warp while mailbox remains open after cooldown.
- MAIL-T17 block-list policy verification if mailbox blocking is intended.

Statik Mailbox keşfi kapalı; yalnız test sonucu yeni edge çıkarsa tekrar açılmalı.

## Recovered legacy systems — remaining work

### Ticket System
Core static path is mapped. Remaining runtime/security checks:
- TICKET-T01 crafted foreign ticket ID -> PAGE_REPLY disclosure.
- TICKET-T02 apostrophe/backslash payloads in title/content/reply/reason against isolated test DB.
- TICKET-T03 force deterministic ID collision and observe create loop.
- TICKET-T04 staff mode 5/255 against ASan/debug build.
- TICKET-T05 client ticketLoadLogs(id == vector size).
- TICKET-T06 packet fixed-char non-NUL termination audit/runtime test.

### Dungeon Info
Core static path is mapped. Remaining:
- DUNGEON-T01 WARP/RANK index 255 and index >= server vector size.
- DUNGEON-T02 client GC dungeon index 255 under ASan.
- DUNGEON-T03 reload after multiple entries; verify stale slots from Clear().
- DUNGEON-T04 malformed config: level-limit count != entry-position count.
- DUNGEON-T05 config required-item/boss-drop over packet capacity.
- DUNGEON-T06 vecBonus count POINT_MAX_NUM+1.
- confirm Python getter wrappers cannot create additional independent slot/type OOB beyond current findings.

### Battle Pass — static audit still open
Priority:
1. locate every `SetExtBattlePassMissionProgress` caller and classify whether repeat-completed invocation is reachable.
2. map every `UpdateExtBattlePassMissionProgress` gameplay caller by mission type.
3. map `TPlayerExtBattlePassMission` load/create/save/free lifecycle.
4. map `player.battlepass_playerindex` create/load/completed/season rollover behavior.
5. map Event Manager P2P receive side for `TPacketGGEventBattlePass` and reload/start/stop ordering.
6. determine actual configured battle-pass IDs in Project_Game locale/config and impact of bool-as-ID.
7. verify mission update bMissionType bug in live UI: progress packet should be dropped/misdirected when stack byte does not equal mission type.
8. final-reward crash test with missing index row.
9. decide STATIC COMPLETE only after caller+persistence+season lifecycle close.

## Battle Pass — closed static scope

Battle Pass caller, persistence, playerindex, P2P/Event Manager and current Project_Game config audits are closed.

**Status: STATIC COMPLETE.**

Only runtime/fault-injection items BP-T01..BP-T14 remain. Do not re-open static Battle Pass scanning unless:
- runtime behavior contradicts the map;
- source commit changes;
- a new Battle Pass feature/season implementation is added.

Next static target: **Achievement System**.
