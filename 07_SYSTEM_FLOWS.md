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

## Guild Storage — item checkin persistence

Client packet
→ `SafeboxCheckin(...,2)`
→ inventory item validation
→ `RemoveFromCharacter`
→ `Guildstorage::Add`
→ window = GUILDBANK
→ owner pointer = opening character
→ `SaveSingleItem` owner normalization = guild ID
→ `HEADER_GD_ITEM_SAVE`
→ DB direct `REPLACE item`
→ persisted as `owner_id=guildID, window=GUILDBANK`.

## Guild Storage — item checkout persistence

`SafeboxCheckout(...,2)`
→ `Guildstorage::Remove`
→ `AddToCharacter`
→ inventory window / player owner
→ `FlushDelayedSave`
→ `HEADER_GD_ITEM_SAVE`
→ `HEADER_GD_ITEM_FLUSH`
→ DB state forced toward current inventory ownership.

## Guild Storage — multi-core lock davranışı

Core A:
`ReqGuildstorageLoad`
→ local `SetStorageState(true,pid)`
→ SQL state=1

Core B:
- kendi `CGuild::m_data.guildstoragestate` alanı P2P ile güncellenmiyor.
- bu nedenle daha önce false yüklediyse `IsStorageOpen()` false kalabilir.

Sonuç:
DB'de state yazılması tek başına game core B'nin runtime state'ini senkronize etmiyor.

### Core startup etkisi
Yeni/non-auth core başlarken:
`InitializeDonate()`
→ DB'deki **tüm** guild storage state'lerini 0 yapıyor.

Bu reset crash sonrası stale lock temizlemeye yarıyor gibi görünse de başka core'da halen açık storage varsa DB lock'ını da silebilir.

## Guild Storage — permission revocation race

Başlangıç:
member grade → `GUILD_AUTH_BANK` var
→ `ReqGuildstorageLoad()`
→ permission geçer
→ storage açılır

Sonra leader:
- member grade değiştirir veya
- grade auth içinden `GUILD_AUTH_BANK` bitini kaldırır

Mevcut açık session:
- kapanmaz
- checkin handler `GetGuildstorage()` var mı diye bakar
- checkout handler `GetGuildstorage()` var mı diye bakar
- anlık `HasGradeAuth(... GUILD_AUTH_BANK)` kontrolü yok

Dolayısıyla session kapatılana kadar eski yetki fiilen devam edebilir.

## Guild Storage — member removal while open

Storage açık
→ `RemoveMember(pid)`
→ `SetGuild(nullptr)`
→ `m_pkGuildstorage` yaşamaya devam eder

Sonraki checkin:
`CSafebox::Add`
→ `FlushDelayedSave`
→ `SaveSingleItem`
→ GUILDBANK owner çözümü:
`item->GetOwner()->GetGuild()->GetID()`
→ guild pointer null ise crash riski.

Sonraki checkout:
- item local storage'dan çıkarılıp inventory'ye eklenebilir
- ardından GuildLog yolunda
`ch->GetGuild()->GetID()`
→ null dereference riski.

Client normal close:
`/guildstorage_close`
→ server `GetGuild()==nullptr` ise erken return
→ storage nesnesi kapanmaz.

Disconnect:
`CHARACTER::Disconnect`
→ `CloseGuildstorage()`
→ `GetGuild()->SetStorageState(false,0)`
→ guild pointer null ise crash riski.
