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
