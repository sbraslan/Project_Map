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
