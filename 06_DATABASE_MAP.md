# 06 — Database Map

DB / persistence / cache / save akışları burada tutulur.

## Kayıt formatı
- Sistem:
- Server entry:
- DB çağrısı:
- Query / packet:
- Table / alan:
- Commit zamanı:
- Failure path:
- Reconnect etkisi:

## Öncelik
Guild Storage işlemlerinin kalıcı veri zincirini doğrulamak.

## Guild Storage — item persistence

### Checkout
Başarılı checkout sonrası:
1. storage item kaldırılır
2. item character inventory'ye eklenir
3. `ITEM_MANAGER::Instance().FlushDelayedSave(pkItem)`
4. item ID alınır
5. `HEADER_GD_ITEM_FLUSH` DB packet'i gönderilir

### Guild state persistence
`CGuild::SetStorageState(bool val, uint32_t pid)`
→ `guildstoragestate`
→ `guildstoragewho`
→ doğrudan `UPDATE guild...`

`CGuild::SetGuildstorage(int val)`
→ guild storage seviyesi/değeri
→ doğrudan `UPDATE guild...`

### Açık alan
Checkin sırasında `CSafebox::Add` sonrasında item ownership/window/cell bilgisinin hangi DB katmanında kaydedildiği ayrıca izlenecek.

## Guild Storage — DB load modeli doğrulandı

Guild Storage özel bir loader yerine safebox loader'ını kullanıyor:

`HEADER_GD_GUILDSTORAGE_LOAD`
→ `QUERY_SAFEBOX_LOAD(..., bMall=2)`

### Kimlik modeli
`TSafeboxLoadPacket.dwID` = **guild ID**

DB loader bunu `pi->account_id` alanına koyuyor.

Item sorgusu:
`item.owner_id = guild ID`
ve
`window = 'GUILDBANK'`

Dolayısıyla GUILDBANK item sahipliği DB'de karakter/account yerine guild ID üzerinden modellenmiş.

### Dikkat: safebox metadata sorgusu da ortak
İlk sorgu:
`SELECT account_id, size, password, gold FROM safebox WHERE account_id = guildID`

Guild storage password olarak `"000000"` gönderiyor. Guild storage için password kontrolünü atlayacak eski koşullar kodda yorum satırına alınmış durumda.

### Item award davranışı
`RESULT_SAFEBOX_LOAD` içinde `ItemAwardManager::GetByLogin(currentLogin)` guild-bank modu için de çalışıyor.

Mode 2'de:
- mall award'lar atlanıyor
- non-mall award'lar atlanmıyor
- yeni item oluşturulursa `owner_id = guildID`
- `window = 'GUILDBANK'`

Bu davranış ayrıca bug adayı olarak kaydedildi.
