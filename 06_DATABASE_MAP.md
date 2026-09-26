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

## Character item login load

DB sorgusu player ID üzerinden yalnız character-owned windowları seçer:

`owner_id = playerID`

Window listesi:
- INVENTORY
- EQUIPMENT
- DRAGON_SOUL_INVENTORY
- BELT_INVENTORY
- opsiyonel SWITCHBOT
- opsiyonel NPC_STORAGE
- opsiyonel PREMIUM_PRIVATE_SHOP
- opsiyonel ADDITIONAL_EQUIPMENT_1

SAFEBOX / MALL / GUILDBANK bu player item load sorgusunda yoktur; kendi ayrı loader'ları vardır.

### DB cache hit
Player item cache set mevcutsa:
- cache entry'leri `TPlayerItem` dizisine kopyalanır
- `HEADER_DG_ITEM_LOAD` doğrudan gönderilir.

### DB query
Cache yoksa:
`QID_ITEM`
→ `RESULT_ITEM_LOAD`
→ `CreateItemTableFromRes`
→ her item için `owner = dwPID`
→ `HEADER_DG_ITEM_LOAD`
→ ardından `PutItemCache(item, true)`.

`true` loaded itemın hemen DB'ye tekrar yazılmasını engelleyen skip-query davranışıdır.

## Ground item persistence modeli

Ground item game-core runtime objesidir; DB'de kalıcı `GROUND` owner/window kaydı olarak tutulmuyor.

Drop:
1. item character'dan ayrılır veya split item yaratılır.
2. `AddToGround` window'u `GROUND` yapar fakat owner null kalır.
3. `Save()` / `FlushDelayedSave()`
4. `SaveSingleItem(item)`
5. `item->GetOwner() == nullptr`
6. `HEADER_GD_ITEM_DESTROY`
7. DB item row silinir/cache temizlenir.

Bu yüzden yere bırakılmış itemın yaşamı game core memory/sectree + destroy event üzerinden sürer.

Pickup:
1. runtime ground item bulunur.
2. `RemoveFromGround`.
3. `AddToCharacter`.
4. owner yeniden character olur; window/cell atanır.
5. `Save()`
6. `HEADER_GD_ITEM_SAVE`
7. DB item row yeniden yaratılır/güncellenir.

### Destroy
`ITEM_MANAGER::DestroyItem`:
- item delayed-save set'ten çıkarılır
- item ID mevcut ve skip-save değilse `HEADER_GD_ITEM_DESTROY`
- DB `QUERY_ITEM_DESTROY`
- cache varsa silinir; yoksa SQL DELETE.

## Switchbot item persistence

SWITCHBOT ayrı config DB tablosu kullanmıyor.

Item persistence normal player item tablosu üzerinden:
- owner = player PID
- window = SWITCHBOT
- pos = switchbot slot
- normal item ID/vnum/count/socket/attrs.

Inventory → SWITCHBOT:
`MoveItem`
→ source remove
→ `SetItem(SWITCHBOT)`
→ RegisterItem
→ delayed-save final state
→ `HEADER_GD_ITEM_SAVE`
→ player item cache/DB.

Login:
player item load sorgusu SWITCHBOT window'u dahil eder
→ `CInputDB::ItemLoad`
→ `AddToCharacter(SWITCHBOT,pos)`
→ `SetItem`
→ RegisterItem.

### Switchbot configuration persistence
Active/configured alternatives DB'ye yazılmıyor.
Runtime `TSwitchbotTable` game-core memory'de tutuluyor ve cross-core warp sırasında P2P ile taşınıyor.

Normal process restart sonrasında bu runtime configuration'ın kalıcı DB restore yolu bulunmadı.

## Switchbot persistence

Switchbot item ayrı tablo kullanmıyor.

Item `SWITCHBOT` slotuna taşındığında normal item persistence:
`CItem::Save`
→ delayed save
→ `ITEM_MANAGER::SaveSingleItem`
→ owner = player PID
→ window = SWITCHBOT
→ pos = switchbot slot
→ `HEADER_GD_ITEM_SAVE`.

Player item load query opsiyonel olarak:
`window='SWITCHBOT'`
satırlarını da seçer.

Game load:
`HEADER_DG_ITEM_LOAD`
→ `CInputDB::ItemLoad`
→ SWITCHBOT case
→ `item->AddToCharacter(ch,TItemPos(SWITCHBOT,pos))`
→ `SetItem`
→ `CSwitchbotManager::RegisterItem`.

Character delete item cleanup query de SWITCHBOT window'u kapsar.

Switchbot'un active/finished/alternative konfigürasyonu item DB row'unda tutulmaz.
Cross-core state `TSwitchbotTable` ile P2P üzerinden taşınır; normal login sırasında manager runtime state yoksa yalnız item slotu DB'den restore edilir.


## Player Exchange persistence / atomicity boundary

Exchange has no DB-level transaction spanning both characters.

Item commit:
`RemoveFromCharacter / AddToCharacter`
→ item Save
→ `ITEM_MANAGER::FlushDelayedSave(item)` per transferred item.

Currency commit:
`PointChange` mutates runtime character state;
`Accept()` calls character `Save()` conditionally after `Done()`.

Two exchange sides are committed sequentially:
- first `Done()`
- optional first owner Save
- second `Done()`
- optional second owner Save.

If first side succeeds and second side fails, no compensating DB/runtime rollback exists.
If one item succeeds and a later item fails inside the same `Done()`, already flushed item ownership is not restored.

This persistence shape is central to BUG-EXCHANGE-001/002/004.


## Exchange / Trade persistence ordering

Exchange item transferi DB açısından transactional değildir.

CExchange::Done() her item için:
1. item->RemoveFromCharacter()
2. item->AddToCharacter(victim, ...)
3. ITEM_MANAGER::FlushDelayedSave(item)

RemoveFromCharacter ve AddToCharacter itemı delayed-save setine koyar. Ardından FlushDelayedSave itemı setten çıkarıp doğrudan SaveSingleItem çağırır.

SaveSingleItem owner mevcutsa TPlayerItem oluşturur ve HEADER_GD_ITEM_SAVE ile yeni owner/window/pos bilgisini DB cache bağlantısına gönderir.

Bu flush final exchange'in tamamının başarı durumunu beklemez.

### Atomicity sonucu
BUG-EXCHANGE-001 veya BUG-EXCHANGE-002 nedeniyle Done() ortada fail ederse, daha önce taşınmış itemların owner/position save packetleri çoktan gönderilmiş olabilir. Cancel() yalnız exchange state/flags temizler; item ownership rollback yapmaz.

Dolayısıyla partial-transfer riski persistence katmanında da mevcuttur.

### Currency ordering
Gold/Cheque değişiklikleri PointChange ile memory state'i değiştirir. CExchange::Accept başarılı Done() sonrasında currency kullanan karakterler için Save() çağırır.

CHARACTER::Save() -> CHARACTER_MANAGER::DelayedSave. Bu, per-item FlushDelayedSave ile aynı DB transaction değildir.

Özet: item saves per-item immediate flush; player currency saves delayed character save; cross-character atomic transaction yok.


## Premium Private Shop DB transaction path

Sale message from game:
HEADER_GD_SHOP / SHOP_SUBHEADER_GD_BUY / seller pid / display pos.

DB ShopSaleResult uses its cached shop table as source of truth:
- FindItem(displayPos)
- sold = shopTable->items[arrIndex]
- AlterGoldStash(sold.price,true)
- AlterChequeStash(sold.cheque,true)
- optional online sale notification
- RemoveItem(displayPos)
- close if no items
- PutShopCache(GetCacheTable()).

### Stash limits
GOLD_MAX = 2,000,000,000
CHEQUE_MAX = 1,000.

Shop::AlterGoldStash and AlterChequeStash clamp after mutation. There is no sale-side capacity rejection before buyer payment/item transfer.

### Tax data gap
SHOP_SUBHEADER_GD_BUY contains no net price/tax. Therefore DB credits cached listed price, not game-side post-tax dwPrice.

### Cross-layer atomicity observation
Game item owner transfer is immediately ITEM_MANAGER::FlushDelayedSave(item) before SHOP_SUBHEADER_GD_BUY is sent. Buyer currency uses CHARACTER::Save() delayed at end of Buy(). Shop stash/table is a separate DB shop cache mutation. No single commit/rollback primitive spans these three state stores.

## Recovered DB map — Ticket / Dungeon Info / Battle Pass

### Ticket schema
Direct game-server SQL:
- `ticket.list`: ticket identity, owner, title/content, priority, date, status.
- `ticket.reply`: ticket replies.
- `ticket.user_restricted`: account restriction + reason.

No DB-process transaction layer; game server performs synchronous DirectQuery.
User-controlled strings are currently serialized without proper SQL escaping.

### Dungeon Info
- Ranking reads `player.dungeon_ranking` joined with `player.player` and `account.account`.
- Dungeon runtime/config metadata itself comes from locale `dungeon_info.txt` and quest/event flags rather than a DB cache.

### Battle Pass
Observed persistence:
- `player.battlepass_playerindex`
  - player_id
  - player_name
  - battlepass_type
  - battlepass_id
  - start_time
  - battlepass_completed
  - end_time
- mission-level persistence is represented by `TPlayerExtBattlePassMission` and still requires complete load/save table mapping.

Known lifecycle:
- first mission creation can INSERT playerindex row.
- final reward SELECTs completed flag then UPDATEs it to 1 before granting final reward.
- zero-row SELECT is not guarded (BUG-BPASS-005).
