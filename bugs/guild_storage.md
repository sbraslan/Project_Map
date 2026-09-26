# guild storage — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

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

### BUG-GS-007 — ENABLE_SAFEBOX_MONEY altında Guild Storage kapanışı kişisel safebox gold'unu sıfırlayabilir
- Statik durum: **kod yolu doğrulandı**
- Build koşulu: `ENABLE_GUILDSTORAGE_SYSTEM && ENABLE_SAFEBOX_MONEY`

Kanıt zinciri:
1. `LoadGuildstorage` → `new CSafebox(this, iSize, 0)`
2. constructor → `m_lGold = 0`
3. `CloseGuildstorage` → `m_pkGuildstorage->Save()`
4. ortak `CSafebox::Save()` → `dwID = character account ID`, `dwGold = 0`
5. DB → `UPDATE safebox SET gold='0' WHERE account_id=<character account>`

Sonuç:
Guild Storage kapatmak, kişisel safebox tablosundaki gold değerini sıfırlayabilir.

Not:
Bu bug yalnız ilgili compile flag aktifse çalışır. Runtime testi veri kaybı riski nedeniyle yalnız kontrollü test DB'sinde yapılmalı.

### BUG-GS-004 — Cross-core Guild Storage lock senkronizasyonu yok
- Statik durum: **mimari kod yolu doğrulandı**
- Runtime çoklu-core testi: bekliyor

`SetStorageState` local guild objesi + SQL UPDATE yapıyor.
Repo-geneli incelenen Guild P2P subheader'larında open/close lock state aktarımı yok.
`GUILD_SUBHEADER_GG_REFRESH/REFRESH1` farklı guild bilgi refresh işlerine ayrılmış.

Risk:
Farklı game core/channel'lardaki aynı guild objeleri farklı `guildstoragestate` değerleri tutabilir ve aynı guild bank'ı eşzamanlı açabilir.

### BUG-GS-008 — Yeni game core startup tüm Guild Storage kilitlerini DB'de sıfırlıyor
- Statik durum: **kod yolu doğrulandı**
- `main.cpp`: her non-auth game startup → `guild_manager.InitializeDonate()`
- `InitializeDonate()`: `UPDATE guild SET guildstoragestate = 0`

Risk senaryosu:
1. Core A'da guild storage aktif ve state=1.
2. Core B restart/startup yapar.
3. Core B global SQL ile state'i 0 yapar.
4. Yeni yüklenen/yenilenen runtime state üzerinden ikinci erişim mümkün hale gelebilir.

Ek risk:
Bu reset `guildstoragewho` alanını temizlemiyor; state=0 iken eski PID kalabilir.

### BUG-GS-009 — Yetki kaldırıldıktan sonra açık Guild Storage erişimi devam ediyor
- Statik durum: **yüksek güven / kod yolu doğrulandı**
- Build: özellikle `ENABLE_GUILDRENEWAL_SYSTEM && ENABLE_GUILDSTORAGE_SYSTEM`

Açılışta `GUILD_AUTH_BANK` kontrolü var.
Ancak `SafeboxCheckin(..., bMall=2)` ve `SafeboxCheckout(..., bMall=2)` packet yollarında:
- current guild membership
- current member grade
- current `GUILD_AUTH_BANK`
yeniden kontrol edilmiyor.

`ChangeMemberGrade` ve `ChangeGradeAuth` açık sessionı kapatmıyor.

Sonuç:
Bir oyuncunun bank yetkisi storage açıkken kaldırılırsa mevcut oturum üzerinden item koyma/çekme devam edebilir.

### BUG-GS-010 — Guildden çıkarılan/disband edilen açık-storage kullanıcısında null dereference/core crash
- Statik durum: **çok yüksek güven / birden fazla crash yolu doğrulandı**

Koşul:
`m_pkGuildstorage != nullptr`
ve ardından
`ch->SetGuild(nullptr)`

Bu durum `RemoveMember` ve `Disband` sırasında oluşabiliyor.

Crash yolları:
1. Checkin → `SaveSingleItem` → GUILDBANK owner → `GetGuild()->GetID()`.
2. Checkout sonunda GuildLog → `ch->GetGuild()->GetID()`.
3. Disconnect → `CloseGuildstorage()` → `GetGuild()->SetStorageState(...)`.

Ek problem:
`/guildstorage_close` komutu `GetGuild()==nullptr` ise erken return ediyor; açık storage objesi temizlenmiyor.

Bu nedenle guild officer'ın storage kullanan üyeyi çıkarması bile core crash'e dönüşebilir.

### BUG-GS-003 ek doğrulama — Pending load sırasında guild üyeliği kaybı
DB load cevabı geldiğinde:
`gID = ch->GetGuild() ? guildID : 0`
ve response guildID ile eşleşmezse fonksiyon direkt return ediyor.

Bu yolda:
- `m_bOpeningGuildstorage` temizlenmiyor
- daha önce set edilmiş guild storage lock temizlenmiyor

Dolayısıyla pending-load removal senaryosu stuck lock riskini güçlendiriyor.

### BUG-GS-011 — Guild disband sonrası orphan GUILDBANK itemları
- Statik durum: **yüksek güven**

Guild disband DB cleanup:
- guild
- guild_grade
- guild_member
- guild_comment

siliniyor.

Ancak guild ID owner'lı ve `window=GUILDBANK` item satırları silinmiyor.

Kaynakta ayrıca:
`//ADD_DELETE_FUNCTION_FOR_GUILD_ITEMS_IN_STORAGE`
yorumu mevcut fakat karşılığında kod yok.

Sonuç:
Disband sonrası eski guild itemları DB'de orphan kalabilir. Guild ID yeniden kullanılabilen/migrate edilen bir ortamda eski itemların başka guild tarafından görülmesi riski ayrıca test edilmeli.
