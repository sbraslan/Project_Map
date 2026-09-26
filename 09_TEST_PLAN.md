# 09 — Test Plan

Kod haritalaması bittikçe oyun içi testler buraya eklenir.

## Guild Storage temel testleri

### GS-T01 — Normal checkin
- Geçerli inventory item
- Geçerli guild storage slot
- Beklenen: item taşınır, server ve client state eşleşir

### GS-T02 — Normal checkout
- Geçerli storage item
- Geçerli inventory hedefi
- Beklenen: item inventory'ye gelir

### GS-T03 — Invalid slot
- Sınır dışı slot
- Beklenen: işlem reddedilir, state bozulmaz

### GS-T04 — Permission
- Yetkisiz karakter / rank
- Beklenen: işlem reddedilir

### GS-T05 — Logout race
- İşlem sırasında logout
- Beklenen: item duplicate / loss oluşmaz

### GS-T06 — Reconnect
- İşlem sonrası reconnect
- Beklenen: kalıcı state doğru yüklenir

### GS-T07 — Direct packet authorization
- Guild storage UI açılmadan checkout packet'i gönderme senaryosu
- Beklenen: server işlemi reddetmeli
- Amaç: authorization yalnız UI/open aşamasına bağımlı mı kontrol etmek

### GS-T08 — Cross-role checkout
- Storage yetkisi olmayan guild rank ile packet gönderimi
- Beklenen: server-side reddetme

### GS-T09 — Concurrent checkout
- Aynı guild storage slotuna iki guild üyesinin yakın zamanlı erişimi
- Beklenen: yalnız bir işlem başarılı olmalı; duplicate/loss olmamalı

### GS-T10 — Open request abort / stuck lock
1. Guild storage open request başlat.
2. DB cevabından önce çakışan bir pencere durumu oluşturulabilen senaryoyu dene veya kontrollü gecikme uygula.
3. Load response'un abort yoluna girmesini sağla.
4. Storage'ı tekrar açmayı dene.
Beklenen güvenli davranış: lock temizlenmiş olmalı.
Risk işareti: sürekli “already open”.

### GS-T11 — Disconnect during pending load
- Open request gönderildikten sonra, `HEADER_DG_GUILDSTORAGE_LOAD` gelmeden bağlantıyı kes.
- Yeniden bağlan ve storage aç.
- DB'de `guildstoragestate/guildstoragewho` kontrol et.
Beklenen: state 0'a dönmeli.

### GS-T12 — Cross-channel simultaneous open
- Aynı guild'den iki yetkili karakteri farklı game core/channel'larda hazırla.
- Aynı anda storage açmayı dene.
Beklenen: yalnız biri açabilmeli.

### GS-T13 — Guild ID / account ID collision
- Guild ID ile aynı numeric ID'ye sahip account safebox satırı ve non-default safebox password bulunan kontrollü test DB'si kullan.
- Guild storage aç.
Beklenen: account safebox password'u guild storage'yı etkilememeli.

### GS-T14 — Item award isolation
- Test oyuncusuna alınmamış non-mall item_award tanımla.
- Personal safebox yerine önce guild storage aç.
- Award'ın hangi window/owner'a yazıldığını kontrol et.
Beklenen: kişisel award guild bank'a taşınmamalı.

### GS-T15 — SAFEBOX_MONEY guild-close regression
**Yalnız test DB / yedekli ortamda.**
- `ENABLE_SAFEBOX_MONEY` aktif build kullan.
- Kişisel safebox gold değerini test amaçlı bilinen bir değere ayarla.
- Guild Storage aç/kapat.
- `safebox.gold` değerini tekrar kontrol et.
Beklenen güvenli davranış: kişisel safebox gold değişmemeli.
Mevcut statik kod beklentisi: 0'a yazılma riski var.

### GS-T16 — Core restart while storage open
- En az iki game core/channel bulunan kontrollü test ortamı.
- Core A'da Guild Storage açık tutulur.
- Core B yeniden başlatılır.
- DB'de `guildstoragestate/guildstoragewho` gözlenir.
- Core B veya üçüncü core'dan aynı guild storage açılmaya çalışılır.

Beklenen güvenli davranış:
Aktif Core A lock'ı korunmalı ve ikinci açılış reddedilmeli.

Mevcut statik kod beklentisi:
Core B startup DB state'i 0'a çeker; cross-core erişim riski oluşur.

### GS-T17 — Bank auth revoke while storage open
1. Oyuncu A'ya `GUILD_AUTH_BANK` ver.
2. A Guild Storage açsın.
3. Leader A'nın grade'ini bank yetkisiz grade'e değiştirsin veya mevcut grade'den bank auth bitini kaldırsın.
4. A mevcut açık pencereden item checkin ve checkout denesin.

Beklenen güvenli davranış:
Session hemen kapanmalı veya sonraki packet reddedilmeli.

Mevcut statik beklenti:
İşlemler devam edebilir.

### GS-T18 — Remove member while storage open
1. A Guild Storage açsın.
2. Yetkili B, A'yı guildden çıkarsın.
3. A:
   - item koymayı
   - item çekmeyi
   - storage kapatmayı
   - logout/reconnect'i
   ayrı ayrı denesin.
4. Core log/core dump izle.

Mevcut statik beklenti:
Birden fazla null dereference/core crash yolu mevcut.

### GS-T19 — Pending load + member removal
1. DB load gecikmesi oluştur.
2. A open request yollasın.
3. Response gelmeden A guildden çıkarılsın.
4. Response sonrası DB `guildstoragestate`, `guildstoragewho` ve karakter `m_bOpeningGuildstorage` davranışı kontrol edilsin.

Risk:
stuck lock.

### GS-T20 — Disband with stored items
1. Test guild bank'a benzersiz item ID'leri koy.
2. Guild'i disband et.
3. DB item tablosunda:
   `owner_id=<oldGuildID> AND window='GUILDBANK'`
   satırlarını kontrol et.

Beklenen güvenli davranış:
Item lifecycle açık bir politika ile cleanup/archive edilmeli.

Mevcut statik beklenti:
rows orphan kalıyor.

### GUILD-T01 — Offline member remove with PulseManager
- `ENABLE_PULSE_MANAGER` aktif test build.
- Offline guild member'ı çıkar.
- Core crash/log kontrolü.

### ITEM-T01 — Destroy count semantics
- Stack count örneğin 50 olan test itemı kullan.
- Destroy packetini count=1 / 10 gibi partial değerlerle gönder.
- Sonuç inventory + DB'de gözlenir.

Beklenen API davranışı:
requested count kadar işlem veya count alanının açıkça ignored/full-delete olarak tasarlanması.

Mevcut statik kod:
tüm item stack objesini siliyor.

### ITEM-T02 — Destroy use-after-free
- Debug/ASan mümkünse aktif test game core.
- Destroy system üzerinden normal item yok et.
- `CHARACTER::RemoveItem` dönüşündeki ChatPacket/GetName yolunu izle.
- crash / sanitizer UAF raporu / bozuk isim kontrol et.

### ITEM-T03 — Drop AddToGround failure rollback
Kontrollü test ortamında `AddToGround` false yolu üret:
- invalid/olmayan sectree veya kontrollü instrumentation.

Full ve partial stack ayrı test edilir.

Beklenen:
item/count source'a geri dönmeli veya işlem false olmalı.

Mevcut statik kod:
rollback yok, fonksiyon true dönüyor.

### ITEM-T04 — Ground DB lifecycle
1. Benzersiz item ID'li itemı inventory'de doğrula.
2. Yere bırak.
3. DB row/cache durumunu kontrol et.
4. Item hâlâ world'de iken tekrar pickup yap.
5. DB row'un player owner/window ile yeniden oluştuğunu doğrula.
6. Ek olarak item yerdeyken game core crash/restart senaryosunu kontrollü test et.

Amaç:
ground itemların DB'den bilinçli olarak çıkarıldığını ve core memory'ye bağımlı olduğunu doğrulamak.

### ITEM-T05 — Destroy sender sequence davranışı
- Client network logging ile ardışık destroy + başka packet senaryoları test edilir.
- `SendItemDestroyPacket` sonrası `SendSequence` eksikliğinin packet dispatch üzerinde etkisi olup olmadığı ölçülür.

### ITEM-T06 — Invalid DB item position / AddToCharacter
**Yalnız kontrollü test DB'de.**

Ayrı testler:
- INVENTORY pos > valid max
- BELT_INVENTORY pos >= BELT_INVENTORY_SLOT_COUNT
- DRAGON_SOUL_INVENTORY pos >= DRAGON_SOUL_INVENTORY_MAX_NUM
- SWITCHBOT / ADDITIONAL invalid pos

Karakter login edilir.

Kontrol:
- core crash / ASan OOB
- item manager VID/ID map durumu
- owner pointer
- character slot arrays
- DB row'un sonraki save davranışı

Beklenen güvenli davranış:
invalid row karantinaya/restore listesine alınmalı; array indexing yapılmamalı.

### ITEM-T07 — Additional Equipment occupied-slot swap
`ENABLE_ADDITIONAL_EQUIPMENT_PAGE` aktif build.

- page 0 ve page 1 ayrı test
- aynı wear slotunda mevcut item varken başka item equip et
- swap sonrası iki itemın:
  - window
  - cell
  - equipped state
  - stat bonus
  - client görünümü
  - DB persistence
değerlerini kontrol et.

Amaç:
`SwapItem` shadowing için statik incelemede doğrudan runtime placement etkisi bulunmadı. Bu test artık sınıflandırma testi değil, page 0/page 1 occupied-slot swap için **regression doğrulaması** olarak tutulur.

### SWITCHBOT-T01 — Cross-core warp memory leak
- Switchbot manager oluşturmuş test karakteri.
- İki game core/channel arasında tekrarlı warp/channel change.
- Source core RSS/heap/LSan izle.
- Her transfer sonrası Switchbot state target core'da doğrulanır.
- Beklenen güvenli davranış: source object destroy edilmeli.
- Statik beklenti: allocation birikir.

### SWITCHBOT-T02 — Logout manager/event lifecycle
1. Switchbot slotuna item koy.
2. Aktif switch başlat.
3. Logout ol.
4. Server event/debug instrumentation ile PID'nin manager entry/event varlığını izle.
5. Çok sayıda farklı test PID ile tekrarla.

Kontrol:
- map size
- active event count
- CPU tick
- memory.

### SWITCHBOT-T03 — Empty slot START
Normal UI dışından kontrollü packet:
- önce PID için Switchbot manager oluştur
- slotu boşalt
- START packetini boş slot için gönder
- server manager table ve event state'ini izle.

Beklenen güvenli davranış:
START reject.

Mevcut statik beklenti:
active=true + persistent 0.2s event.

### SWITCHBOT-T04 — Stale item ID
Test instrumentation ile `table.items[slot]` runtime'da bulunmayan ID'ye ayarla ve active et.
Eventin otomatik slot disable/cleanup yapıp yapmadığını gözle.
Statik beklenti: sonsuz continue.

### SWITCHBOT-T05 — Start/stop + movement invariants
- active slotu ITEM_MOVE ile çıkarma → reject
- active slotu UseItem ile çıkarma → reject
- inactive slotu inventory'ye çıkarma → unregister
- relog → SWITCHBOT DB item restore/RegisterItem
- same-core warp ve cross-core warp ayrı test.

### SWITCHBOT-T06 — UPDATE_ITEM vnum truncation observation
VNUM >255 item kullan.
Packet capture/debug ile server update.vnum truncation'ı doğrula.
Client item index'in normal ITEM_SET state'i sayesinde doğru kalıp kalmadığını kontrol et.

### ITEM-T08 — Special Inventory type/range
Kontrollü karakter üzerinde:
- skillbook
- metin stone
- material/resource
- normal item
ile move/autogive testleri.

Doğrula:
- normal item special range'e giremez
- special item yalnız kendi range'ine gider
- special item size > 1 varsa special grid reddeder
- relog sonrası window=INVENTORY ve special cell korunur.

Ek corruption testi:
DB'de special itemı yanlış special subrange pos'a koy.
Relog sonrası restore davranışını ve manuel move ile recovery'yi izle.

### SWITCHBOT-T01 — Basic lifecycle
1. uygun item INVENTORY → SWITCHBOT
2. alternative configure
3. Start
4. attribute değişimi
5. Stop/finish
6. SWITCHBOT → INVENTORY
7. relog.

Kontrol:
- DB window/pos
- manager item ID
- active/finished
- client refresh
- item attrs
- save/load.

### SWITCHBOT-T02 — Active item move lock
Switchbot active iken itemı:
- inventory'ye
- başka switchbot slotuna
taşımayı dene.

Beklenen:
server reddeder, item/persistence değişmez.

### SWITCHBOT-T03 — Cross-core warp leak
Test sunucusunda core/channel port değişimi üreten warp yap.

Her tekrar öncesi/sonrası:
- process RSS/heap
- switchbot manager object count
- active event count
izlenir.

Beklenen mevcut statik koda göre:
source core'da erase edilen `CSwitchbot` object free edilmez.

### SWITCHBOT-T04 — P2P state resume ordering
Active switchbot ile cross-core warp:
- table target core'a ulaşıyor mu
- EnterGame öncesi/sonrası arrival sırası
- active slot event yeniden başlıyor mu
- duplicate event oluşuyor mu
kontrol edilir.

### SWITCHBOT-T05 — Invalid/empty START hardening
Yalnız kontrollü test clientı ile:
- manager var fakat seçilen slot boş
- stale item ID
- slot out-of-range
START senaryoları gönder.

Beklenen güvenli davranış:
START reddedilmeli ve periyodik event bırakılmamalı.

### SWITCHBOT-T06 — Client boundary
Python binding'e slot == SWITCHBOT_SLOT_COUNT ile Start/Stop çağrısı ver.
Server'ın range check ile işlemi reddettiğini ve state değişmediğini doğrula.

### ITEM-T09 — Storage TItemPos destination allowlist
Kontrollü test clientı/Python console ile Safebox, Mall ve Guild Storage checkout için 3-arg window-aware binding kullan.

Destination matris:
- INVENTORY
- special inventory doğru/yanlış subrange
- BELT
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1
- DRAGON_SOUL_INVENTORY.

SWITCHBOT:
- geçerli weapon
- normal potion/material
- costume/non-supported type
ayrı denenir.

Kontrol:
- server reject/accept
- item window/cell
- manager registration
- DB persistence
- relog state.

Beklenen güvenli tasarım:
yalnız storage özelliğinin açıkça desteklediği destination windowları kabul edilmeli ve hedef window'un semantic validator'ı yeniden çalışmalı.

### SWITCHBOT-T07 — Remove active item outside MoveItem
Active tek slot ile:
1. normal logout
2. kontrollü SafeboxCheckin source=SWITCHBOT
3. item expiration/removal mümkünse
senaryoları ayrı test et.

İzle:
- `HasActiveSlots`
- `IsSwitching`
- event count
- CPU/tick
- manager map entry.

Mevcut statik beklenti:
Unregister active flag'i temizler fakat event Stop edilmediği için empty recurring event kalabilir.

### ITEM-T10 — Additional Equipment direct checkout
`ENABLE_ADDITIONAL_EQUIPMENT_PAGE` build.

Safebox/Mall/Guild Storage checkout destination:
`ADDITIONAL_EQUIPMENT_1`.

Uygun ve uygunsuz item ile:
- slot pointer
- item window/cell
- IsEquipped
- stat bonus
- client visibility
- page unlock state
- relog persistence
kontrol edilir.


### ITEM-T11 — Special Inventory bWindow bounds
Yalnız izole development server'da test et.

Extend Inventory request ve upgrade için special-state açıkken sınır değerleri:
- valid: 0, 1, 2
- invalid: 3 ve 255

İzle:
- server log/crash
- ASan/UBSan varsa array OOB
- `bSpecialInventoryStage[3]` komşu state
- key consumption
- stage mutation
- player save/load sonucu.

Beklenti: invalid window server tarafında hiçbir state'e dokunmadan reddedilmeli.

### ITEM-T12 — Locked special slot persistence restore
Kontrollü test karakteri ve yedek DB ile:
1. special stage=0 bırak.
2. aynı special type'ın ilk 45 açık slotunun üstünde fakat static type range içinde bir persisted item position hazırla.
3. login/item load çalıştır.
4. item owner/window/cell, grid pointer, client visibility ve relog persistence kontrol et.

Beklenti: locked position restore edilmemeli; item güvenli recovery inventory path'ine alınmalı veya explicit reject/restore queue uygulanmalı.

### ITEM-T13 — Special type size > 1 dataset audit
Runtime server `item_proto` tablosu veya unpacked proto export üzerinde:
- ITEM_SKILLBOOK
- ITEM_METIN
- ITEM_MATERIAL
- ITEM_RESOURCE
- vnum 27987

için `size > 1` satırları ara.

Kodun mevcut invariant'ı: `IsEmptySpecialItemGrid(..., bSize > 1) -> false`. Dataset'te böyle item varsa special auto-placement/movement uyumsuzluğu ayrıca sınıflandırılmalı.


## Exchange / Trade — izole runtime test matrisi

Bu testler yalnız local/dev server ve disposable test karakterleriyle yapılmalı.

### EX-T01 — Special Inventory preflight/commit mismatch
Amaç: BUG-EXCHANGE-001 partial transfer etkisini doğrulamak.

Hazırlık:
- alıcı normal inventory'de yeterli boş alan,
- hedef special type inventory tamamen dolu,
- gönderici offer sırasına önce normal item, ardından aynı special type item ekler.

Beklenen güvenli davranış: accept öncesi trade tamamen reddedilmeli veya hiçbir item hareket etmeden rollback olmalı.
Bug göstergesi: ilk normal item alıcıya geçer, special item space failure verir ve exchange kapanır.

### EX-T02 — Page4 tek-slot / iki item
Amaç: BUG-EXCHANGE-002'yi doğrulamak.

Hazırlık:
- alıcı page1-3 dolu,
- page4 unlock edilmiş,
- page4'te yalnız 1 adet size=1 boş slot,
- gönderici 2 adet size=1 normal item offer eder.

Beklenen güvenli davranış: preflight ikinci item için space yok diyerek commit öncesi reddetmeli.
Bug göstergesi: accept başlar, ilk item transfer olur, ikinci itemda `Exchange::Done : Cannot find blank position` görülür ve ilk item rollback olmaz.

### EX-T03 — Switchbot source window
Amaç: BUG-EXCHANGE-003 server trust boundary etkisini doğrulamak.

Modified test client/Python ile ITEM_ADD `TItemPos(SWITCHBOT, slot)` gönder.
- inactive valid switchbot item
- active switchbot item
ayrı test edilmeli.

Beklenen güvenli davranış: exchange source allowlist nedeniyle packet reddedilmeli.
Mevcut statik beklenti: AddItem generic validity ile itemı kabul edebilir.

### EX-T04 — Additional Equipment source
Modified test client ile `TItemPos(ADDITIONAL_EQUIPMENT_1, wearCell)` offer et.
Aktif/inaktif equipment page ve normalde `CanUnequipNow` engeli bulunan durumları ayrı test et.

### EX-T05 — packet initialization
Official clientte START sonrası ITEM_ADD/ACCEPT paketlerini debug packet dump ile izle; subheader dışı alanların deterministic zero olup olmadığını doğrula. Server `arg1` prelookup nedeniyle sporadic early-return logları ayrıca takip edilmeli.


### EXCHANGE-T01 — Special Inventory preflight/commit divergence
İzole development server ve disposable karakterlerle:
1. recipient regular inventory'de yeterli boşluk bırak.
2. ilgili special tabı doldur veya target type için kullanılabilir special slot bırakma.
3. sender trade'e önce normal tradeable item, sonra special-type tradeable item eklesin.
4. iki taraf accept etsin.

Kontrol:
- CheckSpace sonucu
- ilk item ownership değişimi
- ikinci special item için `GetEmptyInventory(item)` sonucu
- Cancel sonrası iki itemın gerçek owner/window/cell ve DB state'i.

Beklenti: hiçbir item kısmi transfer olmamalı; bug mevcut kodda partial mutation riski gösteriyor.

### EXCHANGE-T02 — Extend inventory page-4 reservation
Recipient'ın boş alanını yalnız 4. inventory sayfasında kontrollü bırak.
Birden fazla size=1 ve ardından multi-slot item ile CheckSpace simülasyonunu gerçek `GetEmptyInventory` sonuçlarıyla karşılaştır.

Özellikle `s_grid4.Put` çağrısının her accepted item için reserve edip etmediğini logla.

### EXCHANGE-T03 — Unsupported source windows
Yalnız test client/dev server:
- SWITCHBOT source
- ADDITIONAL_EQUIPMENT_1 source
ile ITEM_ADD gönder.

Beklenti: production fix sonrası exchange server yalnız açıkça desteklenen source windowları kabul etmeli.

Kontrol: item `SetExchanging`, Switchbot register/event state, additional-equipment effects ve Cancel sonrası state.

### EXCHANGE-T04 — Gold overflow TOCTOU
1. recipient gold'u `GOLD_MAX - offer - delta` seviyesine getir.
2. sender gold offer eklesin; offer-time overflow check geçsin.
3. final accept öncesinde recipient dev ortamında normal reachable bir gold gain yolu ile bakiyesini artırıp `recipient + offer >= GOLD_MAX` yap.
4. accept et.

Kontrol:
- sender gold before/after
- recipient gold before/after
- overflow log
- Done return/success UI.

Beklenti: currency transfer atomic olmalı; sender debit recipient credit başarısızken kalıcılaşmamalı.

### EXCHANGE-T05 — Cheque late overflow rollback
Offer oluşturulduktan sonra recipient cheque bakiyesini limite yaklaştır ve final accept et.

Kontrol: cheque check false olduğunda daha önce taşınan item/gold mutationlarının geri dönüp dönmediği.


### EX-T06 — final distance server enforcement
Amaç: BUG-EXCHANGE-004 runtime doğrulaması.

Dev client ile iki karakter <=1000 range içinde trade açsın; client-side auto-CANCEL devre dışı bırakılarak taraflardan biri >1000 uzaklaşsın; iki taraf accept etsin.

Beklenen güvenli davranış: server final ACCEPT sırasında mesafeyi ölçüp transactionı reddetmeli. Statik mevcut beklenti: mesafe recheck olmadığı için commit devam eder.

### EX-T07 — gold cap TOCTOU
Amaç: BUG-EXCHANGE-005 doğrulaması.

Disposable karakterlerle: receiver başlangıçta cap kontrolünü geçecek gold seviyesinde olsun; sender gold offer etsin; offer sonrası receiver ground gold pickup/party distribution ile GOLD_MAX - offered üstüne çıksın fakat GOLD_MAX altında kalsın; iki taraf accept etsin.

İzlenecekler: sender/receiver gold before-after, OVERFLOW_GOLD log, exchange success/end packetleri, DB player save sonucu.

Bug göstergesi: sender debit gerçekleşir, receiver addition overflow'da reddedilir.

### EX-T08 — partial transfer persistence
EX-T01 veya EX-T02 ile partial item transfer oluştur. Ardından iki karakteri logout/login yap; gerekirse local DB cache/game restart sonrası item owner/position kontrol et.

Bug göstergesi: önce taşınmış item yeni owner'da kalır; failed remainder eski owner'dadır.

### EX-T09 — lifecycle cancellation sanity
Ayrı ayrı participant disconnect, participant death ve standard CanWarp kullanan portal/channel change test et.

Beklenti: disconnect/death exchange'i cancel eder; active W_EXCHANGE nedeniyle standard CanWarp reddedilir.


## Shop / Premium Private Shop runtime tests

### SHOP-T01 — stash cap clipping
Use disposable local characters/shop. Set seller shop gold stash just below GOLD_MAX and/or cheque stash just below CHEQUE_MAX; list item whose price exceeds remaining stash capacity; buy it.

Track buyer debit/item receipt, seller stash before-after, DB shop cache and sale notification.

Bug indicator: sale succeeds but stash increase is less than sale proceeds because it clamps at max.

### SHOP-T02 — premium personal_shop tax
Set personal_shop event flag to a visible nonzero tax. Buy a premium private shop item.

Compare buyer debit, seller stash delta, logged local net dwPrice and listed sold.price.

Bug indicator: seller stash delta equals full listed price instead of post-tax value.

### SHOP-T03 — sale persistence ordering
Dev/fault-injection only: instrument boundaries after item FlushDelayedSave and before/after SHOP_SUBHEADER_GD_BUY and buyer Save. Verify reconnect/restart consistency. Do not run on production data.

## Recovered test plan — Ticket / Dungeon Info / Battle Pass

### Ticket
- TICKET-T01 foreign valid ticket ID -> PAGE_REPLY.
- TICKET-T02 SQL metacharacters/apostrophe in create/reply/reason on disposable DB.
- TICKET-T03 deterministic ticket-ID collision.
- TICKET-T04 staff sort modes 0/5/255 and high pages.
- TICKET-T05 client log request id == vector size under ASan.
- TICKET-T06 crafted non-NUL fixed-char subpacket.

### Dungeon Info
- DUNGEON-T01 WARP/RANK 255 and vector-size boundary.
- DUNGEON-T02 GC index 255 on client under ASan.
- DUNGEON-T03 multi-entry reload then inspect stale client slots.
- DUNGEON-T04 mismatch LEVEL_LIMIT vs ENTRY_BASE_POSITION counts.
- DUNGEON-T05 exceed required-item/boss-drop config capacities.
- DUNGEON-T06 POINT_MAX_NUM+1 bonus entries.

### Battle Pass
- BP-T01 complete mission, then invoke any reachable SetExt caller again at threshold; check duplicate reward.
- BP-T02 capture MISSION_UPDATE packets; verify bMissionType nondeterminism/mismatch.
- BP-T03 long season/config name against RequestOpen under ASan.
- BP-T04 open pass repeatedly to expose dangling season_name behavior under ASan/UBSan.
- BP-T05 remove playerindex row then request final reward.
- BP-T06 Event Manager start/stop season with configured ID >1 and inspect active ID.
- BP-T07 process boot before Event Manager population; inspect uninitialized array values.
- BP-T08 season rollover/reload with existing mission/playerindex rows.

## Battle Pass — static-complete runtime matrix

- BP-T09: complete a mission, wait until the reward item has left ITEM_MANAGER delayed-save queue / is visible in DB, then hard-kill GAME before clean logout; relog and verify whether mission can reward again. Covers BUG-BPASS-008.
- BP-T10: inject crash/failure after `battlepass_playerindex.battlepass_completed=1` UPDATE but before/during `BattlePassReward`; relog and retry final claim. Covers BUG-BPASS-009.
- BP-T11: repeated login/logout with many historical `battlepass_missions` rows under ASan/LSan or RSS monitoring; verify leaked `TPlayerExtBattlePassMission` objects. Covers BUG-BPASS-010.
- BP-T12: fresh CHARACTER first ranking request before any setter; instrument/read `m_dwLastExtBattlePassOpenRankingTime`. Covers BUG-BPASS-011.
- BP-T13: run `battlepass_set_mission` as GM_IMPLEMENTOR twice on an already completed mission with value >= target; verify duplicate mission reward. Covers BUG-BPASS-002.
- BP-T14: temporarily configure BattlePassID 2 in isolated environment, start through Event Manager and verify active ID remains 1. Covers BUG-BPASS-007.
