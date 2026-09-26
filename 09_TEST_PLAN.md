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
