# exchange — Runtime Tests

> Canonical split from legacy `09_TEST_PLAN.md`. Run only in isolated/dev data unless explicitly marked safe.

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
