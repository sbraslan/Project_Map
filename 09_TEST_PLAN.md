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
`SwapItem` başındaki shadowed `srcCell/destCell` kodunun runtime etkisini belirlemek.

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
