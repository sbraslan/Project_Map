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
