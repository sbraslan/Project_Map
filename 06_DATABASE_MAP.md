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
