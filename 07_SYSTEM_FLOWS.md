# 07 — System Flows

Uçtan uca akışlar burada tutulur.

## Guild Storage — Checkin

Mevcut doğrulanmış yüksek seviye akış:

Python UI  
→ `netSendGuildstorageCheckinPacket(...)`  
→ `TItemPos(window_type, cell)`  
→ `CPythonNetworkStream`  
→ `CG_GUILDSTORAGE_CHECKIN`  
→ Server

### Açık noktalar
- Server entry handler
- Item / slot / ownership validation
- Guild permission kontrolü
- DB persistence
- Başarısızlık response'u
- Client refresh
- Logout / reconnect sonrası tutarlılık

## Guild Storage — Checkout
Aynı mimarinin benzeri olduğu tespit edildi; ayrıntılı zincir kaynak koddan yeniden doğrulanarak doldurulacak.

## Guild Storage — Checkout doğrulandı

Python UI
→ `netSendGuildstorageCheckoutPacket(...)`
→ `CPythonNetworkStream::SendGuildstorageCheckoutPacket(...)`
→ `HEADER_CG_GUILDSTORAGE_CHECKOUT (85)`
→ `CInputMain::Analyze`
→ `SafeboxCheckout(ch, packet, 2)`
→ `ch->GetGuildstorage()`
→ storage slot item lookup
→ target inventory validation
→ `pkSafebox->Remove(...)`
→ `pkItem->AddToCharacter(...)`
→ `FlushDelayedSave(pkItem)`
→ `HEADER_GD_ITEM_FLUSH`
→ log / optional guild P2P refresh

## Guild Storage — Checkin doğrulanan server tarafı

`HEADER_CG_GUILDSTORAGE_CHECKIN (84)`
→ `SafeboxCheckin(ch, packet, 2)`
→ `ch->GetGuildstorage()`
→ source item validation
→ destination storage slot validation
→ `RemoveFromCharacter()`
→ `pkSafebox->Add(...)`
→ logs

Checkin için DB persistence alt zinciri henüz tam kapatılmadı.

## Guild Storage — tam open/load/close akışı

Guild/NPC/UI tetikleme
→ `click_guildstorage`
→ `do_click_guildstorage`
→ `SetGuildstorageOpenPosition()`
→ `ReqGuildstorageLoad()`
→ guild / window / permission / distance / overlap kontrolleri
→ `HEADER_GD_GUILDSTORAGE_LOAD (150)`
→ `SetStorageState(true, pid)`
→ DB `QUERY_SAFEBOX_LOAD(...,2)`
→ `item WHERE owner_id=guildID AND window='GUILDBANK'`
→ `HEADER_DG_GUILDSTORAGE_LOAD (52)`
→ `CInputDB::GuildstorageLoad`
→ `LoadGuildstorage`
→ `SetOpenGuildstorage(true)`
→ `HEADER_GC_GUILDSTORAGE_OPEN (141)`
→ client `RecvGuildstorageOpenPacket`
→ `CPythonGuildBank::OpenGuildBank`
→ Python `OpenGuildstorageWindow`

### Close
Python `uiguildbank.Close`
→ `/guildstorage_close`
→ `do_guildstorage_close`
→ `CloseGuildstorage`
→ `SetStorageState(false,0)`
→ client close command.

### Yetki
`GUILD_AUTH_BANK` kontrolü packet checkin/checkout handler'ında değil, **open/load request aşamasında** bulunuyor ve yalnız `ENABLE_GUILDRENEWAL_SYSTEM` build koşulu altında derleniyor.
