# 02 — Client Map

Client tarafındaki sistem, dosya, sınıf ve fonksiyon eşleşmeleri burada tutulur.

## Kayıt formatı

### Sistem
- Repo:
- Dosya:
- Sınıf / fonksiyon:
- Çağıran:
- Çağrılan:
- Packet:
- Not:
- Kanıt durumu: doğrulandı / kısmi / açık

## Guild Storage
İlk doğrulanmış zincirlerden biri Python UI → C++ network binding → CPythonNetworkStream yönündedir.

Ayrıntılı fonksiyon ve dosya eşleşmeleri sonraki salt-okuma turunda kaynak repodan yeniden doğrulanarak buraya işlenecek.

## Guild Storage — client send/receive zinciri doğrulandı

### Checkin
`PythonNetworkStreamModule.cpp`
→ `netSendGuildstorageCheckinPacket(...)`
→ `CPythonNetworkStream::SendGuildstorageCheckinPacket(...)`

`PythonNetworkStreamPhaseGameItem.cpp`:
- `TPacketCGGuildstorageCheckin`
- header: `HEADER_CG_GUILDSTORAGE_CHECKIN`
- source: `TItemPos InventoryPos`
- destination: `bSafePos`
- `Send(...)` → `SendSequence()`

### Checkout
`netSendGuildstorageCheckoutPacket(...)`
→ `CPythonNetworkStream::SendGuildstorageCheckoutPacket(...)`

Packet:
- header: `HEADER_CG_GUILDSTORAGE_CHECKOUT`
- source guild slot: `bGuildstoragePos`
- destination: `TItemPos ItemPos`

### Server → client
`HEADER_GC_GUILDSTORAGE_OPEN`
→ `RecvGuildstorageOpenPacket()`
→ `CPythonGuildBank::OpenGuildBank(size)`
→ Python `OpenGuildstorageWindow(size)`

`HEADER_GC_GUILDSTORAGE_SET`
→ `RecvGuildstorageItemSetPacket()`
→ local guild bank item data
→ `RefreshGuildstorage`

`HEADER_GC_GUILDSTORAGE_DEL`
→ `RecvGuildstorageItemDelPacket()`
→ local item delete
→ refresh

### UI close
`root/uiguildbank.py::Close()`
→ `SendChatPacket("/guildstorage_close")`
→ server command handler.

Client ayrıca 1000 birim mesafe aşımında UI'ı kapatıp close komutunu gönderiyor.

## Inventory / Item Move — client zinciri

### UI
`Project_Binary/root/uiinventory.py`

Inventory/equipment slot eventleri:
- `SelectEmptySlot`
- `SelectItemSlot`
- `UseItemSlot`

Normal inventory taşıma:
`__SendMoveItemPacket(src,dst,count)`
→ private-shop build/edit kontrolü
→ `m2netm2g.SendItemMovePacket(src,dst,count)`

5-parametreli window-aware kullanım da mevcut; örneğin SWITCHBOT:
`SendItemMovePacket(SWITCHBOT, src, INVENTORY, dst, count)`.

### Python → C++ binding
`UserInterface/PythonNetworkStreamModule.cpp::netSendItemMovePacket`

Desteklenen imzalar:
- 3 arg: source cell, destination cell, count
- 5 arg: source window, source cell, destination window, destination cell, count

→ `CPythonNetworkStream::SendItemMovePacket(Cell, ChangeCell, count)`

### C++ network send
`PythonNetworkStreamPhaseGameItem.cpp::SendItemMovePacket`

Client-side kontroller:
- `__CanActMainInstance()`
- equipment source ise exchange/shop sırasında equip cell hareketi engellenir
- equipment source ve player attacking ise gönderim yapılmaz

Packet:
- `header = HEADER_CG_ITEM_MOVE`
- `pos = source TItemPos`
- `change_pos = destination TItemPos`
- `num = count`

→ `Send`
→ `SendSequence`.

## Inventory — pickup / drop / destroy client zinciri

### Pickup
Python:
`netSendItemPickUpPacket(vid)`
→ `CPythonNetworkStream::SendItemPickUpPacket(vid)`
→ `HEADER_CG_ITEM_PICKUP`
→ `SendSequence()`.

### Drop
Legacy:
`netSendItemDropPacket`
→ `SendItemDropPacket`
→ `HEADER_CG_ITEM_DROP`.

Count-aware:
`netSendItemDropPacketNew`
→ `SendItemDropPacketNew`
→ `HEADER_CG_ITEM_DROP2`.

### Destroy
`netSendItemDestroyPacket(Cell,count)`
→ `SendItemDestroyPacket(Cell,0,count)`
→ `HEADER_CG_ITEM_DESTROY`.

Not:
Destroy sender `Send(...)` sonrası doğrudan `true` dönüyor; diğer yakın item sender'larının aksine `SendSequence()` çağırmıyor. Bu şimdilik **davranış farkı** olarak kaydedildi, tek başına bug ilan edilmedi.
