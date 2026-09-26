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
