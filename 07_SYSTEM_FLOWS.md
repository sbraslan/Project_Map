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

## Inventory / Item Move — uçtan uca

`uiinventory.py`
→ `SendItemMovePacket`
→ `netSendItemMovePacket`
→ `CPythonNetworkStream::SendItemMovePacket`
→ `HEADER_CG_ITEM_MOVE(13)`
→ `CInputMain::ItemMove`
→ `CHARACTER::MoveItem`

`MoveItem` sonucu dört ana kola ayrılıyor:

**STACK**
→ source/target `SetCount`
→ update packet
→ delayed save

**SPLIT**
→ source count azalt
→ new item yarat
→ sockets copy
→ `AddToCharacter(dest)`
→ delayed save

**EQUIP**
→ `EquipItem`
→ `EquipTo`
→ equipment state + attributes/effects
→ delayed save

**NORMAL MOVE**
→ `RemoveFromCharacter`
→ delayed-save pointer queue
→ `SetItem(dest)`
→ owner/window/cell destination state
→ manager save cycle
→ `HEADER_GD_ITEM_SAVE`
→ DB cache / item table.

Quickslot mapping INVENTORY↔BELT ve aynı-window taşımalarda ayrıca senkronize ediliyor.

## Inventory — login/load ters persistence zinciri

DB/cache
→ `TPlayerItem`
→ `HEADER_DG_ITEM_LOAD(42)`
→ `CInputDB::ItemLoad`
→ `CreateItem(vnum,count,id)`
→ metadata restore
→ destination window reconstruction
→ INVENTORY/BELT/SWITCHBOT/etc: `AddToCharacter`
→ EQUIPMENT: `EquipTo`
→ client item/equipment packets
→ `SetItemLoaded`

Load sırasında `SetSkipSave(true)` sayesinde `AddToCharacter/EquipTo` içindeki normal `Save()` çağrıları DB'ye geri yazılmaz.

Collision:
DB target occupied / equip restore fail
→ restore queue
→ free inventory slot
→ yoksa ground + temporary ownership.

## Inventory — Drop persistence akışı

**Full stack**
Inventory item
→ `RemoveFromCharacter`
→ owner null / RESERVED
→ `AddToGround`
→ GROUND + sectree
→ flush
→ `SaveSingleItem(owner=null)`
→ `HEADER_GD_ITEM_DESTROY`
→ DB row silinir
→ runtime ground object yaşamaya devam eder.

**Partial stack**
source `SetCount`
→ source DB save/flush
→ new item(count)
→ sockets copy
→ `AddToGround`
→ new ground item DB persistence'tan çıkarılır.

## Inventory — Pickup persistence akışı

Ground VID
→ distance
→ ownership
→ stack merge veya empty slot
→ `RemoveFromGround`
→ `AddToCharacter`
→ player owner/window/cell
→ delayed save
→ `HEADER_GD_ITEM_SAVE`
→ player item row DB'ye geri yazılır.

## Inventory — Destroy

Client DESTROY packet
→ `CInputMain::ItemDestroy`
→ `CHARACTER::RemoveItem`
→ `ITEM_MANAGER::RemoveItem`
→ `DestroyItem`
→ `HEADER_GD_ITEM_DESTROY`
→ cache/SQL delete
→ item C++ object delete.

## Invalid persisted item position → login risk akışı

DB item row
→ valid window enum fakat invalid/out-of-range `pos`
→ `HEADER_DG_ITEM_LOAD`
→ `CInputDB::ItemLoad`
→ fresh `CItem` (`m_wCell=0`)
→ `AddToCharacter(window, invalidPos)`
→ yanlış bounds check: old `m_wCell` kontrol edilir
→ passes
→ `SetItem(window,invalidPos)`.

BELT/DS:
→ array index invalidPos bounds check öncesi kullanılabilir
→ OOB memory access / core crash riski.

INVENTORY vb.:
→ `SetItem` erken return edebilir
→ AddToCharacter yine owner set eder ve true döner
→ item container'a yerleşmeden owner/state inconsistency oluşabilir.

Normal `MoveItem` client yolu bu riskten farklıdır:
→ destination önce `IsValidItemPosition` ile doğrulanır.

## Inventory occupied-target multi-slot swap

Source item footprint S
→ occupied destination footprint D
→ preflight checks
→ D içindeki itemları map'e topla
→ size equality
→ source remove
→ D items remove
→ D items → S footprint
→ source → D base
→ quickslot sync
→ delayed save manager final positions serialize eder.

Önemli persistence detayı:
`SetItem` explicit Save çağırmasa da her swapped item öncesinde `RemoveFromCharacter` yaptığı için delayed-save set'te bulunur.

## Switchbot — full runtime flow

Inventory item
→ generic ITEM_MOVE
→ destination SWITCHBOT
→ `SetItem`
→ `RegisterItem(pid,itemID,slot)`
→ manager object gerekirse `new CSwitchbot`
→ table.items[slot]=itemID
→ item DB window=SWITCHBOT.

START
→ `HEADER_CG_SWITCHBOT(171)`
→ fixed alternatives parse
→ `Manager::Start`
→ active=true
→ config copy
→ event_create(0.2s).

Event
→ `SwitchItems`
→ item ID resolve
→ target attributes check
→ switcher/gold consume
→ `ChangeAttribute`
→ UPDATE_ITEM.

Completion
→ active=false
→ finished=true
→ no active slot kalırsa Stop/event_cancel.

### Inter-core warp
Character warp:
→ `SetIsWarping(pid,true)`
→ target port farklıysa `P2PSendSwitchbot`
→ event Pause
→ source manager map entry erase
→ full table `HEADER_GG_SWITCHBOT(31)`
→ target core `P2PReceiveSwitchbot`
→ new/existing object SetTable
→ EnterGame
→ SetIsWarping(false)
→ active slot varsa Start/resume.

Kaynak core'da erase edilen eski object delete edilmediği için leak oluşur.

### Normal logout
`CHARACTER::Disconnect` içinde Switchbot manager için Stop/Pause/erase/delete çağrısı bulunmuyor.

Itemların character lifecycle sırasında kaldırılması table slotlarını unregister edebilse bile manager object'in kendisini kaldıran lifecycle yok.
Aktif stale state/event kalırsa item lookup null döndükçe event 0.2s cadence ile devam eder.
