# 05 — Packet Map

Client ↔ server packet eşleşmeleri burada merkezi olarak tutulur.

## Kayıt formatı

| Sistem | Yön | Packet | Client gönderici/alıcı | Server handler | Durum |
|---|---|---|---|---|---|
| Guild Storage | C→S | CG_GUILDSTORAGE_CHECKIN | doğrulanacak ayrıntı | doğrulanacak ayrıntı | kısmi |
| Guild Storage | C→S | checkout packet | doğrulanacak | doğrulanacak | açık |

## Kural
Packet adı, struct, opcode/header ve handler eşleşmesi kaynak koddan doğrulanmadan “tamamlandı” sayılmaz.

## Guild Storage — doğrulanan packet eşleşmeleri

- `HEADER_CG_GUILDSTORAGE_CHECKIN = 84`
- `HEADER_CG_GUILDSTORAGE_CHECKOUT = 85`
- `HEADER_GC_GUILDSTORAGE_OPEN = 141`
- `HEADER_GC_GUILDSTORAGE_SET = 142`
- `HEADER_GC_GUILDSTORAGE_DEL = 143`

### Client structs
`TPacketCGGuildstorageCheckin`: `bHeader`, `bSafePos`, `TItemPos ItemPos`

`TPacketCGGuildstorageCheckout`: `bHeader`, `bGuildstoragePos`, `TItemPos ItemPos`

### Server dispatch
- 84 → `SafeboxCheckin(..., 2)`
- 85 → `SafeboxCheckout(..., 2)`

Durum: **packet → server entry doğrulandı**.

## Guild Storage — DB packet katmanı

Game → DB:
- `HEADER_GD_GUILDSTORAGE_LOAD = 150`
- `HEADER_GD_GUILDSTORAGE_CHANGE_SIZE = 151`

DB → Game:
- `HEADER_DG_GUILDSTORAGE_LOAD = 52`
- `HEADER_DG_GUILDSTORAGE_CHANGE_SIZE = 53`

### Load dispatch
Game:
`ReqGuildstorageLoad()`
→ `HEADER_GD_GUILDSTORAGE_LOAD`

DB:
`CClientManager`
→ `QUERY_SAFEBOX_LOAD(..., 2)`

Result:
`RESULT_SAFEBOX_LOAD`
→ mode 2 ise `HEADER_DG_GUILDSTORAGE_LOAD`

Game:
`CInputDB::GuildstorageLoad`
→ `CHARACTER::LoadGuildstorage`
→ client `HEADER_GC_GUILDSTORAGE_OPEN`.

## Item persistence packetleri — Guild Storage ile ilişkili

- `HEADER_GD_ITEM_SAVE = 30`
- `HEADER_GD_ITEM_DESTROY = 31`
- `HEADER_GD_ITEM_FLUSH = 35`

Guild Storage checkin sırasında `HEADER_GD_ITEM_SAVE` kullanılır.

Checkout sırasında inventory durumuna `HEADER_GD_ITEM_SAVE` gönderildikten sonra item ID ile `HEADER_GD_ITEM_FLUSH` gönderilerek DB tarafındaki cache varsa zorla flush edilir.

## Inventory / Item Move packet

### Client → Game
`HEADER_CG_ITEM_MOVE = 13`

Server struct:
`command_item_move`
- `uint8_t header`
- `TItemPos Cell`
- `TItemPos CellTo`
- `uint8_t count`

Client struct aynı wire verilerini:
- `pos`
- `change_pos`
- `num`
alanlarıyla gönderiyor.

### Server → Client item state
Inventory destination/source state değişimleri `CHARACTER::SetItem` üzerinden:
- item varsa `HEADER_GC_ITEM_SET`
- item kaldırıldıysa `HEADER_GC_ITEM_DEL`
ile client'e yansıtılıyor.

Stack count değişimleri `CItem::SetCount` → `UpdatePacket()` üzerinden güncelleniyor.
