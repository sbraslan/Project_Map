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
