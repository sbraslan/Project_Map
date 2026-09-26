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

## Guild Storage — checkin persistence zinciri tamamlandı

Inventory
→ `RemoveFromCharacter()`
→ Guild `CSafebox::Add`
→ `SetWindow(GUILDBANK)`
→ storage cell
→ `CItem::Save()`
→ `ITEM_MANAGER::DelayedSave`
→ `FlushDelayedSave`
→ `SaveSingleItem`

`SaveSingleItem` GUILDBANK için:
- `TPlayerItem.window = GUILDBANK`
- `TPlayerItem.owner = item->GetOwner()->GetGuild()->GetID()`
- `TPlayerItem.pos = guild storage slot`
- `HEADER_GD_ITEM_SAVE`

DB `QUERY_ITEM_SAVE`:
- GUILDBANK, SAFEBOX/MALL gibi normal character item cache yolundan ayrılıyor.
- varsa eski item cache kaydı kaldırılıyor.
- doğrudan `REPLACE INTO item(... owner_id, window, pos ...)` çalıştırılıyor.

Son DB modeli:
- `owner_id = guild ID`
- `window = GUILDBANK`
- `pos = guild storage slot`

## Guild Storage — checkout persistence zinciri tamamlandı

GUILDBANK
→ `CSafebox::Remove`
→ geçici RESERVED/no-owner hali
→ `AddToCharacter`
→ INVENTORY / character owner
→ `Save`
→ `FlushDelayedSave`
→ `HEADER_GD_ITEM_SAVE`
→ ardından `HEADER_GD_ITEM_FLUSH (35)`

DB:
`QUERY_ITEM_FLUSH(itemID)`
→ `GetItemCache(itemID)`
→ varsa `CItemCache::Flush()`.

### ENABLE_SAFEBOX_MONEY özel durumu
`LoadGuildstorage`, Guild Storage `CSafebox` nesnesini **gold=0** ile oluşturuyor.

`CloseGuildstorage` ise `ENABLE_SAFEBOX_MONEY` altında ortak `CSafebox::Save()` çağırıyor.

Ortak `CSafebox::Save()`:
- `dwID = opening character account ID`
- `dwGold = m_lGold`
- `HEADER_GD_SAFEBOX_SAVE`

DB:
`UPDATE safebox SET gold=<dwGold> WHERE account_id=<character account ID>`

Guild Storage nesnesinin gold'u 0 olduğu için bu yol kişisel safebox gold alanına 0 yazabilir.

## Guild disband — GUILDBANK item cleanup eksikliği

DB `CClientManager::GuildDisband` şunları siliyor:
- `guild`
- `guild_grade`
- `guild_member`
- `guild_comment`

Ancak:
`DELETE FROM item WHERE owner_id=<guildID> AND window='GUILDBANK'`
benzeri bir cleanup bulunmadı.

Game tarafı `RequestDisband` sonunda açıkça:
`//ADD_DELETE_FUNCTION_FOR_GUILD_ITEMS_IN_STORAGE`
yorumunu taşıyor ancak implementasyon yok.

Sonuç:
Disband edilen guild ID'sine bağlı GUILDBANK item rows orphan olarak DB'de kalabilir.

## Inventory / Item Move — persistence modeli

Normal character itemlarında `ITEM_MANAGER::SaveSingleItem`:
- `TPlayerItem.id = item ID`
- `window = current item window`
- `pos = current cell`
- `count = current count`
- owner switch'in default kolunda `character player ID`
- `HEADER_GD_ITEM_SAVE`

Normal INVENTORY/EQUIPMENT/BELT vb. itemlar DB tarafında player item cache yoluna gider.

### Full move için önemli delayed-save davranışı
`RemoveFromCharacter()`:
1. source `SetItem(..., nullptr)`
2. `m_pOwner=null`
3. cell=0
4. window=RESERVED
5. `Save()` → item pointer delayed-save set'e girer

Ardından aynı synchronous `MoveItem` çağrısında:
`SetItem(destination,item)`
→ `SetCell(character,destination)`
→ destination window.

Delayed-save set bir snapshot değil **item pointer** tuttuğu için, manager daha sonra `SaveSingleItem(item)` çağırdığında destination owner/window/cell state'i serialize edilir.

### Stack
`CItem::SetCount`:
- `UpdatePacket()`
- `Save()`
yapar.
Source ve target stack değişimleri delayed persistence'a gider.

### Split
Source count save edilir.
Yeni item `AddToCharacter` sonunda `Save()` ile destination state'i kaydeder.

### Equip
`CItem::EquipTo` sonunda `Save()`; window/cell equipment state'i persistence'a gider.
