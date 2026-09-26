# 08 — Bug Candidates

Bu dosyada sadece **aday** problemler tutulur. Kaynak kod + davranış testi doğrulanmadan “kesin bug” denmez.

## Kayıt formatı

### BUG-CANDIDATE-XXX
- Sistem:
- Repo / dosya:
- Belirti:
- Teknik neden:
- Etki:
- Exploit ihtimali:
- Reprodüksiyon:
- Kaynak kanıt:
- Oyun içi test:
- Durum: açık / doğrulandı / yanlış alarm / düzeltildi

## Öncelikli test alanları
- Guild Storage slot sınırları
- Invalid window_type / cell
- Eşzamanlı checkin / checkout
- Logout sırasında işlem
- Reconnect sonrası item/state tutarlılığı
- Permission bypass ihtimali

### BUG-CANDIDATE-GS-001 — Guild storage ve safebox mantığının ortak handler kullanması
- Sistem: Guild Storage
- Konum: `CInputMain::SafeboxCheckin / SafeboxCheckout`
- Gözlem: Guild Storage, `bMall == 2` ile safebox kod yolunu yeniden kullanıyor.
- Risk: bazı kontroller safebox semantiğine göre yazılmış olabilir; guild permission / concurrency kuralları handler içinde görünmüyor.
- Durum: **inceleme gerekli**, henüz bug olarak doğrulanmadı.

### BUG-CANDIDATE-GS-002 — Checkout permission kontrolü
- Gözlem: incelenen `SafeboxCheckout` gövdesinde doğrudan guild rank/permission kontrolü görünmedi.
- Not: permission daha erken storage açılışında uygulanıyor olabilir.
- Risk: packet doğrudan gönderilebiliyorsa server-side authorization eksikliği oluşabilir.
- Durum: **kritik doğrulama bekliyor**; exploit iddiası henüz yok.

### BUG-CANDIDATE-GS-003 — Load abort sonrası guild storage kilidinin takılı kalması
- Statik durum: **yüksek güven / kod yolu doğrulandı**
- Runtime test: bekliyor

Akış:
1. `ReqGuildstorageLoad()` DB request gönderir.
2. Hemen `SetStorageState(true,pid)` çağrılır.
3. Ancak gerçek `SetOpenGuildstorage(true)`, DB cevabı sonrasında `LoadGuildstorage()` içinde olur.
4. `CInputDB::GuildstorageLoad` cevabı geldiğinde başka pencere açılmışsa `CancelSafeboxLoad()` çağrılıyor.
5. `CancelSafeboxLoad()` yalnız `m_bOpeningSafebox=false` yapıyor; **`m_bOpeningGuildstorage` veya guild storage state'i temizlemiyor**.
6. Disconnect cleanup ise yalnız `IsOpenGuildstorage()==true` olduğunda state'i kapatıyor.

Sonuç adayı:
DB request ile gerçek open arasındaki hata/iptal yolları `guildstoragestate=1` bırakabilir ve sonraki açılışlar `IsStorageOpen()` nedeniyle reddedilebilir.

### BUG-CANDIDATE-GS-004 — Cross-core/channel lock senkronizasyonu
- Statik durum: **yüksek güvenli mimari risk**
- `SetStorageState` local `CGuild::m_data` değerini değiştiriyor ve SQL UPDATE yapıyor.
- İncelenen kullanımlarda bu open/close state için P2P broadcast bulunmadı.
- Başka game core/channel kendi `CGuild` objesinde eski `guildstoragestate` değerini tutabilir.
- Sonuç: aynı guild bank'ın farklı core/channel üzerinden eşzamanlı açılabilme ihtimali.
- Runtime multi-channel test gerekli.

### BUG-CANDIDATE-GS-005 — Guild ID / safebox account ID password collision
- DB guild storage load, önce `safebox.account_id = guildID` sorgusu yapıyor.
- Request password sabit `000000`.
- Guild storage için password check bypass kodu mevcut fakat yorum satırında.
- Eğer aynı numeric ID'ye sahip safebox kaydında farklı password varsa guild storage load yanlışlıkla `WRONG_PASSWORD` yoluna girebilir.
- Runtime/DB fixture testi gerekli.

### BUG-CANDIDATE-GS-006 — Item award'ın guild bank'a yönlenmesi
- `RESULT_SAFEBOX_LOAD` item-award işleme kodu `bMall=2` için de aktif.
- Mode 2'de non-mall award atlanmıyor.
- Insert sırasında `owner_id = guildID`, `window='GUILDBANK'`.
- Potansiyel sonuç: oyuncunun kişisel safebox award'ı guild storage açılırken guild bank item'ına dönüşebilir.
- Runtime testi gerekli.
