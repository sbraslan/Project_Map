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

## Login item load packet

DB → Game:
`HEADER_DG_ITEM_LOAD = 42`

Payload:
- önce `uint32_t itemCount`
- ardından `TPlayerItem[itemCount]`

DB `RESULT_ITEM_LOAD` bu paketi üretir.
Game input dispatch:
`HEADER_DG_ITEM_LOAD`
→ `CInputDB::ItemLoad`.

## Inventory pickup/drop/destroy packetleri

Client → Game:
- `HEADER_CG_ITEM_DROP = 12`
- `HEADER_CG_ITEM_PICKUP = 15`
- `HEADER_CG_ITEM_DROP2 = 20`
- `HEADER_CG_ITEM_DESTROY = 25` (`ENABLE_DESTROY_SYSTEM`)

### DROP2
`TPacketCGItemDrop2`
- header
- `TItemPos Cell`
- `uint32_t gold`
- `uint8_t count`

### DESTROY
`TPacketCGItemDestroy`
- header
- `TItemPos Cell`
- `uint32_t gold`
- `uint8_t count`

### PICKUP
`TPacketCGItemPickup`
- header
- `uint32_t vid`

Game → DB item deletion:
- `HEADER_GD_ITEM_DESTROY = 31`
Payload:
- item ID
- last owner PID.

DB:
`QUERY_ITEM_DESTROY`
→ item cache delete veya
→ `DELETE FROM item WHERE id=<itemID>`.

## Switchbot packetleri

### Client → Game
`HEADER_CG_SWITCHBOT = 171`

`TPacketCGSwitchbot`:
- uint8 header
- int size
- uint8 subheader
- uint8 slot

Subheader:
- START
- STOP

START ardından sabit sayıda `TSwitchbotAttributeAlternativeTable` taşır.

### Game → Client
`HEADER_GC_SWITCHBOT = 180`

Subheaders:
- UPDATE
- UPDATE_ITEM
- SEND_ATTRIBUTE_INFORMATION

### Game ↔ Game / P2P
`HEADER_GG_SWITCHBOT = 31`

`TPacketGGSwitchbot`:
- header
- target port
- full `TSwitchbotTable`.

### UPDATE_ITEM struct observation
Server ve client'ta:
`TSwitchbotUpdateItem.vnum` = `uint8_t`.

Server:
`update.vnum = item->GetVnum()`
ile 32-bit vnum'u 8-bit alana daraltıyor.

Fakat mevcut client `RecvSwitchbotPacket/UPDATE_ITEM` bu `vnum` alanını item index set etmek için kullanmıyor; yalnız count/sockets/attrs güncelliyor.
Bu nedenle şimdilik wire-format kusuru/ölü alan gözlemi olarak tutuluyor.


## Extend Inventory packets

### Client -> Game
`HEADER_CG_EXTEND_INVEN_REQUEST = 140`

`TPacketCGSendExtendInvenRequest`:
- header
- `bStepIndex`
- `bWindow`
- `bSpecialState`

`HEADER_CG_EXTEND_INVEN_UPGRADE = 141`

`TPacketCGSendExtendInvenUpgrade`:
- header
- `bWindow`
- `bSpecialState`

Client convention:
- `bWindow == 10` -> normal inventory
- otherwise -> special inventory

Server dispatch trusts `bSpecialState` and forwards `bWindow` without 0..2 validation. Therefore packet-level malformed special window reaches array-indexed character state. See BUG-ITEM-007.

### Game -> Client
`HEADER_GC_EXTEND_INVEN_INFO = 177` contains normal stage/max plus `bExtendSpecialStage[3]` and `bExtendSpecialMax[3]`.

`HEADER_GC_EXTEND_INVEN_RESULT = 178` reports key/result state.


## Exchange / Trade packetleri

CG ana packet: `TPacketCGExchange`
- `header = HEADER_CG_EXCHANGE`
- `sub_header`
- `uint32_t arg1`
- `uint8_t arg2`
- `TItemPos Pos`
- cheque build'de `uint32_t cheque`.

CG subheaderlar:
- START — `arg1 = target VID`
- ITEM_ADD — `Pos = source TItemPos`, `arg2 = exchange display slot`
- ITEM_DEL
- ELK_ADD — `arg1 = gold`, cheque build'de `cheque`
- ACCEPT
- CANCEL.

GC exchange packeti offer state, item metadata, sockets/attrs ve build'e bağlı ek item verilerini iki tarafa gönderir.

### Trust boundary notu
ITEM_ADD source `Pos.window_type` packetten gelir. Official UI yalnız inventory/DS üretse de Python binding explicit window_type kabul eder ve server genel `IsValidItemPosition` dışında exchange-specific window allowlist uygulamaz. Bkz. BUG-EXCHANGE-003.

### Initialization notu
Client `TPacketCGExchange` yerel değişkenleri zero-init edilmiyor. Bazı subheaderlarda `arg1` dahil kullanılmayan alanlar initialize edilmeden gönderilir. Server handler switch öncesi `arg1` okuduğu için packet deterministic initialization ayrıca test edilmelidir.


## Player Exchange packet boundary

`HEADER_CG_EXCHANGE = 27`.

CG subheaders:
- START
- ITEM_ADD
- ITEM_DEL
- ELK_ADD
- ACCEPT
- CANCEL.

`TPacketCGExchange` içinde source item konumu tam `TItemPos Pos` olarak gelir; ITEM_ADD server'a `window_type + cell` taşır.

Server ITEM_ADD path:
`CInputMain::Exchange -> CExchange::AddItem(pinfo->Pos, pinfo->arg2)`.

Packet boundary semantic allowlist uygulamaz. Generic `TItemPos::IsValidItemPosition()` SWITCHBOT ve ADDITIONAL_EQUIPMENT_1 gibi windowları da valid saydığı için exchange-specific source validation eksiktir.

ELK_ADD ayrıca gold/cheque recipient-overflow check'ini yalnız offer packet işlendiği anda yapar; final ACCEPT öncesi aynı snapshot yeniden doğrulanmaz.
