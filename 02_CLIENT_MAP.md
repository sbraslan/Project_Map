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

## Switchbot — client UI ve network

### Item move
`root/uiswitchbot.py::SelectEmptySlot`:
- INVENTORY → SWITCHBOT:
  `SendItemMovePacket(INVENTORY,src,SWITCHBOT,dst,count)`
- SWITCHBOT → SWITCHBOT:
  aynı generic move packet.

`SelectItemSlot` active slot ise drag başlatmıyor.

`UseItemSlot`:
`SendItemUsePacket(SWITCHBOT,slot)`.
Client active kontrolü burada yok; server `CHARACTER::UseItem` active Switchbot slotunu reddediyor.

### Start/Stop UI
`SwitchbotWindow::SetActive`
→ `switchbot.Start(selectedSlot)` / `Stop`.

Normal UI START butonunu:
- slot boşsa
- hiçbir attribute configured değilse
disable ediyor.

Bu yalnız client-side UX kontrolüdür; server packet handler aynı şartları bağımsız doğrulamıyor.

### Network
`SendSwitchbotStartPacket`:
- `HEADER_CG_SWITCHBOT`
- START subheader
- slot
- sabit `SWITCHBOT_ALTERNATIVE_COUNT` alternative table
- `SendSequence()`

STOP:
- header/subheader/slot
- `SendSequence()`.

### UPDATE_ITEM
Client receiver:
`SUBHEADER_GC_SWITCHBOT_UPDATE_ITEM`
→ count
→ sockets
→ attributes
→ Yohara random attrs
→ UI refresh.

Packet struct içindeki `uint8_t vnum` alanı receiver tarafından okunmuş struct içinde bulunmasına rağmen item index güncellemesi için kullanılmıyor.

## Switchbot — client zinciri

### UI
`Project_Binary/root/uiswitchbot.py`

Item yerleştirme:
- INVENTORY → SWITCHBOT: window-aware `SendItemMovePacket`
- SWITCHBOT → SWITCHBOT: aynı move packet yolu
- SWITCHBOT → INVENTORY: inventory UI tarafından aynı generic move sistemi.

UI Start koruması:
`__RefreshButtons()`
→ selected slot boşsa veya hiçbir attribute configure edilmemişse Start/Stop disable.

### Python module
`Project_ClientSrc/UserInterface/PythonSwitchbot.cpp`

`switchbot.Start(slot)`
→ configured alternatives alınır
→ `CPythonNetworkStream::SendSwitchbotStartPacket`.

`switchbot.Stop(slot)`
→ `SendSwitchbotStopPacket`.

Not:
Binding tarafında `if (bSlot > SWITCHBOT_SLOT_COUNT)` kullanılıyor.
Bu nedenle tam `SLOT_COUNT` değeri client binding'den geçebilir; server tarafı `slot < SLOT_COUNT` ile yeniden doğrular.

### Network
`PythonNetworkStreamPhaseGame.cpp`

Start:
`HEADER_CG_SWITCHBOT`
→ `SUBHEADER_CG_SWITCHBOT_START`
→ 5 alternative table
→ `SendSequence()`.

Stop:
`HEADER_CG_SWITCHBOT`
→ `SUBHEADER_CG_SWITCHBOT_STOP`
→ `SendSequence()`.

Receive:
`HEADER_GC_SWITCHBOT`
→ UPDATE: local CPythonSwitchbot table refresh
→ UPDATE_ITEM: SWITCHBOT TItemPos üzerindeki count/socket/attribute refresh
→ SEND_ATTRIBUTE_INFORMATION: allowed attribute/max-value map refresh.

## Safebox/Mall/Guild Storage — window-aware binding

`PythonNetworkStreamModule.cpp` storage bindingleri hem legacy cell-only hem window-aware form destekliyor.

### Safebox checkin
2 arg:
- source window otomatik INVENTORY

3 arg:
- source `window_type`
- source cell
- safebox slot

### Safebox checkout
2 arg:
- safebox slot
- destination cell
- `TItemPos` default ctor nedeniyle window INVENTORY

3 arg:
- safebox slot
- destination `window_type`
- destination cell

### Mall checkout
Aynı 2/3 arg destination modeli.

### Guild Storage
Checkin ve checkout da 3 arg formunda client Python katmanından explicit window type kabul ediyor.

Bu binding esnekliği nedeniyle server'ın yalnız normal UI davranışına güvenmemesi gerekir; supported window enumları Python tarafında doğrudan üretilebilir.


## Extend Inventory — client packet boundary

`PythonNetworkStreamModule.cpp`:
- `net.SendExtendInvenButtonClick(step, bWindow)`
- `net.SendExtendInvenUpgrade(bWindow)`

Python binding `bWindow` değerini `uint8_t` olarak alıyor; 0..2 allowlist kontrolü yapmıyor.

`PythonNetworkStreamPhaseGame.cpp`:
- `bWindow == 10` ise normal inventory (`bSpecialState=false`)
- diğer tüm values için `bSpecialState=true`

Dolayısıyla normal UI 0/1/2 üretse bile Python/network boundary modified client tarafından 3..255 special-window değerlerini üretebilir. Server bunun için kendi bounds validation'ını yapmak zorunda.


## Exchange / Trade client zinciri

UI: `Project_Binary/root/uiexchange.py`

Item add:
`SelectOwnerEmptySlot`
→ attached slot type yalnız Inventory veya Dragon Soul ise
→ `m2netm2g.SendExchangeItemAddPacket(window, sourceCell, displayCell)`.

Binding: `Project_ClientSrc/UserInterface/PythonNetworkStreamModule.cpp`
- `netSendExchangeItemAddPacket` explicit `uint8_t window_type`, `uint16_t cell`, `display_pos` alır.
- Binding katmanında Inventory/Dragon Soul allowlist yoktur.

Network: `PythonNetworkStreamPhaseGame.cpp`
- `SendExchangeStartPacket`
- `SendExchangeElkAddPacket`
- `SendExchangeItemAddPacket`
- `SendExchangeAcceptPacket`
- `SendExchangeExitPacket`.

Official Python UI source-type'i kısıtlasa da C++ binding değiştirilmiş Python/client tarafından başka valid TItemPos windowlarıyla çağrılabilir; server trust boundary buna göre ele alınmalıdır.

### Client packet initialization observation
`SendExchange*` fonksiyonları `TPacketCGExchange packet;` kullanıyor, `{}` ile zero-init etmiyor. Her subheader yalnız kendi kullandığı alanları doldurduğundan diğer alanlar wire'da uninitialized kalabilir. Server `CInputMain::Exchange` switch öncesinde `arg1` okuyup character lookup yaptığı için bu salt cosmetic değildir; nadir nondeterministic early-return davranışı oluşturabilir.


## Player Exchange — client send boundary

Python network binding exchange item eklerken source'u iki parçalı alır:
- `window_type` (`uint8_t`)
- `cell` (`uint16_t`)

ve doğrudan `TItemPos(window_type, cell)` üretir.

Client binding tarafında trade source için INVENTORY/DRAGON_SOUL allowlist yoktur. Normal UI güvenli window gönderebilir; modified Python/client ise server `IsValidItemPosition()` tarafından desteklenen başka windowları gönderebilir.

Bu sınır BUG-EXCHANGE-003 için client-side giriş noktasıdır.
