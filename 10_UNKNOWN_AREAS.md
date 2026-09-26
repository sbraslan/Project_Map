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
