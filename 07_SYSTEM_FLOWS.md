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

## Special Inventory — move lifecycle

Item
→ `GetSpecialInventoryType()`
→ skillbook / stone / material veya -1
→ `GetEmptyInventory(item)`
→ type-specific special range
→ `IsEmptySpecialItemGrid`
→ yalnız size 1
→ INVENTORY-window special cell.

Manual move:
source/destination `IsValidItemPosition`
→ normal item special destination kontrolü
→ source INVENTORY ise item type == destination special type
→ grid validation
→ Remove/Set
→ normal item save pipeline.

Persisted malformed row:
DB INVENTORY + special cell
→ ItemLoad
→ AddToCharacter
→ SetItem global inventory bound
→ item-type/subrange eşleşmesi burada yeniden doğrulanmaz.
Bu durum crash yolundan çok yanlış-tab/data-integrity senaryosudur.

## Switchbot — end-to-end lifecycle

Inventory UI
→ `SendItemMovePacket(INVENTORY,src,SWITCHBOT,slot,...)`
→ server MoveItem
→ type + empty-slot validation
→ source remove
→ `SetItem(SWITCHBOT,item)`
→ manager RegisterItem
→ item save `window=SWITCHBOT`.

Configure UI
→ alternatives local CPythonSwitchbot table
→ Start packet
→ server manager active=true
→ switch event.

Event tick
→ item ID lookup
→ owner lookup
→ configured target check
→ tamamlandıysa finished=true / active=false
→ değilse switcher/yang kontrolü
→ consume
→ ChangeAttribute
→ dedicated UPDATE_ITEM packet.

Move-out:
active ise server reddeder.
inactive ise:
SWITCHBOT remove
→ manager UnregisterItem
→ slot config/active/finished temizlenir
→ inventory destination
→ item save INVENTORY state.

Cross-core warp:
source manager
→ Pause
→ complete table P2P
→ target manager SetTable
→ EnterGame
→ active slots için event restart.

Known defect:
source `P2PSendSwitchbot` manager pointer'ını map'ten erase ettikten sonra free etmiyor.

## Storage checkout → non-inventory window bypass

Safebox / Mall / Guild Storage packet
→ client-controlled destination `TItemPos`
→ common `SafeboxCheckout`
→ `IsEmptyItemGrid`.

Branch örneği — SWITCHBOT:
`window=SWITCHBOT, slot<7, slot empty`
→ IsEmptyItemGrid=true
→ non-DS
→ belt check yok
→ special type: normal item -1 == destination cell special type -1
→ storage Remove
→ `AddToCharacter(SWITCHBOT,slot)`
→ `SetItem`
→ Switchbot manager RegisterItem
→ DB save `window=SWITCHBOT`.

Bu yol normal MoveItem'taki `SwitchbotHelper::IsValidItem` kontrolüne uğramaz.

ADDITIONAL_EQUIPMENT_1:
empty valid slot
→ checkout
→ AddToCharacter
→ direct additional window placement
→ EquipTo/CanEquipNow akışı yok.

## Active Switchbot remove → orphan event flow

Running switch event
→ active slot item storage-checkin / logout ClearItem gibi MoveItem dışı bir yolla RemoveFromCharacter
→ SetItem(SWITCHBOT,null)
→ manager UnregisterItem
→ active=false + item=0
→ manager event pointer hâlâ live
→ event tick
→ hiçbir active slot yok
→ SwitchItems return
→ callback tekrar schedule.

Bu, BUG-SWITCHBOT-005'in temel akışı.


## Player Exchange — initial end-to-end flow

Client UI/Python
→ `SendExchangeStartPacket(target VID)`
→ server `CInputMain::Exchange START`
→ state/window/distance/block checks
→ iki adet `CExchange` oluşturulur ve company pointerları birbirine bağlanır.

Item offer:
`SendExchangeItemAddPacket(TItemPos, display)`
→ `CExchange::AddItem`
→ generic position validity
→ anti-give / seal / basic / locked / already-exchanging checks
→ exchange grid reservation
→ item `SetExchanging(true)`.

Accept:
iki taraf accept
→ owner `Check`
→ owner `CheckSpace`
→ company `Check`
→ company `CheckSpace`
→ DB cache connection check
→ first `Done()`
→ second `Done()`
→ Cancel/cleanup.

`Done()` item transfer:
foreach offered item
→ destination empty pos lookup
→ sender quickslot sync
→ `RemoveFromCharacter`
→ `AddToCharacter(victim, destination)`
→ `FlushDelayedSave`
→ logs
→ exchange slot pointer null.

Ardından gold, sonra cheque transfer edilir.

### Atomicity boundary
Preflight yalnız tahmin yapar; commit transaction değildir.
`Done()` içindeki herhangi bir orta-adım failure daha önce taşınmış item/gold mutationlarını geri almaz.

Known divergences:
- Special Inventory: CheckSpace regular-grid, Done type-aware `GetEmptyInventory(item)`.
- Extend inventory page 4: CheckSpace reservation control-flow bug.
- Source TItemPos: semantic window allowlist yok.


## Exchange / Trade — uçtan uca

Normal item offer:
`uiexchange.SelectOwnerEmptySlot`
→ `netSendExchangeItemAddPacket`
→ `SendExchangeItemAddPacket(TItemPos, displayPos)`
→ `HEADER_CG_EXCHANGE / ITEM_ADD`
→ `CInputMain::Exchange`
→ `CExchange::AddItem`
→ item `SetExchanging(true)`
→ GC ITEM_ADD iki tarafa.

Commit:
iki taraf ACCEPT
→ `CExchange::Accept`
→ iki taraf `Check`
→ iki taraf `CheckSpace`
→ DB cache bağlantı kontrolü
→ ilk `Done`
→ ikinci `Done`
→ save/log
→ recursive `Cancel` teardown.

### Atomicity failure flow — Special Inventory
A tarafı offer listesinde önce normal item, sonra special item bulundurur.
B tarafının normal inventory'sinde preflight için alan vardır; ilgili special type inventory doludur.

`CheckSpace(A)` normal grid üzerinde iki item için de alan görür
→ accept devam eder
→ `Done(A)` normal itemı B'ye taşır
→ special item için `GetEmptyInventory(item)` special range'e gider
→ boş slot yok, `Done=false`
→ exchange cancel olur
→ ilk taşınan item rollback edilmez.

### Atomicity failure flow — page 4
B'nin page1-3'ü dolu; unlocked page4'te tek bir uygun size=1 slot kalmıştır.
A iki size=1 normal item gönderir.

`CheckSpace(A)` ilk item için page4 blank bulur fakat `s_grid4.Put()` çalışmaz
→ ikinci item aynı blank slotu tekrar bulur
→ preflight true
→ `Done(A)` ilk itemı taşır
→ ikinci item gerçek inventory'de boşluk bulamaz
→ false + cancel
→ ilk transfer rollback edilmez.

Bu iki yol aynı temel invariant ihlalini gösterir: **preflight placement modeli ile commit placement modeli eşdeğer değil ve commit rollback'sizdir.**


## Exchange — distance bypass flow

A ve B başlangıçta <=1000 mesafede -> ExchangeStart başarılı -> paired CExchange -> taraflardan biri normal MOVE ile uzaklaşır -> server movement exchange'i cancel etmez -> modified client official auto-CANCEL davranışını uygulamaz -> iki taraf ACCEPT -> Accept final mesafeyi yeniden ölçmez -> commit devam eder.

## Exchange — gold cap TOCTOU / sender-loss flow

1. Sender gold offer eder.
2. ELK_ADD receiver mevcut gold + offer için cap kontrolü yapar.
3. Receiver exchange açıkken ground ITEM_ELK pickup eder veya nearby party distribution ile gold kazanır.
4. Receiver artık GOLD_MAX - offered sınırının üstündedir fakat GOLD_MAX'in altındadır.
5. İki taraf accept eder.
6. Final Check sender funds'i doğrular; receiver cap recheck yoktur.
7. Done() sender -m_lGold uygular.
8. receiver +m_lGold POINT_GOLD overflow check'te return eder.
9. Done() void failure'ı göremez ve normal akış devam eder.

Sonuç: sender gold azalır, receiver gold artmaz.

## Exchange — persistence of partial item commit

preflight true -> Done item #1 Remove/Add -> FlushDelayedSave(item #1) -> DB HEADER_GD_ITEM_SAVE -> item #2 placement failure -> Done false -> Cancel.

Item #1 için rollback veya compensating DB save yoktur. Bu, BUG-EXCHANGE-001/002'nin reconnect/restart sonrasında da kalıcı olabilmesine neden olur.


## Shop / Premium Private Shop — transaction flow

### Premium PC Shop purchase
Buyer request
→ shop manager / PrivateShopSearchBuy
→ `CShop::Buy(ch,pos,...)`
→ item slot + owner validation
→ buyer gold/cheque validation
→ type-aware destination lookup
→ buyer currency debit
→ shop item ownership transfer to buyer
→ item save flush
→ local shop slot clear/update
→ GAME sends `SHOP_SUBHEADER_GD_BUY(pid,pos)`
→ DB `ShopSaleResult`
→ DB-side stored listing price added to seller stash
→ DB-side listing removed
→ shop cache saved
→ optional online seller sale-info sync.

Important boundary: buyer/item commit precedes seller-stash DB processing; there is no shared transaction or synchronous commit acknowledgement.

### NPC shop buy
Server creates item
→ target slot found
→ cost debited
→ item added/flushed.

### ShopEx
Server creates item
→ target slot found
→ selected currency validated
→ selected currency removed
→ item added/flushed.

### NPC sell
inventory cell item
→ anti-sell/locked/sealed checks
→ price/tax + overflow check
→ item count/remove
→ gold credit.

### Premium listing source
Initial OpenMyShop and AddMyShopItem both restrict actual listed source items to INVENTORY or DRAGON_SOUL_INVENTORY; Premium add-item packet does not inherit the Exchange unsupported-window problem.


## Premium Private Shop — sale flow

buyer Buy packet
-> CShopManager::Buy
-> CShop::Buy
-> funds + destination space validation
-> buyer debit full listed price
-> local personal_shop tax computes net dwPrice
-> shop item transferred to buyer
-> item FlushDelayedSave to DB item state
-> SHOP_SUBHEADER_GD_BUY(sellerPid, displayPos)
-> DB looks up cached sold item
-> DB credits sold.price / sold.cheque to stash
-> removes shop item + cache update.

### Stash-cap loss flow
seller stash close to GOLD_MAX/CHEQUE_MAX
-> buyer purchases another item
-> buyer full debit + receives item
-> DB Alter*Stash adds sale
-> value clamped to max
-> excess seller proceeds disappear.

### Premium tax bypass flow
personal_shop event tax > 0
-> game buyer debit uses full price
-> game local dwPrice reduced by tax
-> net dwPrice is not sent to DB
-> DB credits cached sold.price full amount
-> seller stash receives pre-tax listed price.


## Safebox / Mall — ownership and money flow

### Checkin
UI/Python -> SAFEBOX_CHECKIN -> `SafeboxCheckin` -> source `GetItem(TItemPos)` -> validation -> `RemoveFromCharacter` -> `CSafebox::Add` -> item window SAFEBOX -> forced save/flush -> DB owner account ID.

### Checkout
UI/Python -> SAFEBOX_CHECKOUT -> `SafeboxCheckout` -> source safebox item -> destination validation -> `CSafebox::Remove` -> `AddToCharacter` -> save/flush.

SAFEBOX_IMPROVING auto target uses `GetEmptyInventory(item)`; explicit target remains client-controlled and is BUG-SAFEBOX-004 / BUG-ITEM-006 trust boundary.

### Mall gold overwrite
`LoadMall(gold=0) -> CloseMall -> CSafebox::Save -> HEADER_GD_SAFEBOX_SAVE(dwGold=0) -> UPDATE safebox.gold=0`.

### Money withdraw
request -> signed-int cap precheck -> Safebox debit -> player PointChange credit. If first check overflows but PointChange rejects actual cap, no rollback restores stash.

### Internal stack
SAFEBOX_ITEM_MOVE -> `CSafebox::MoveItem` -> stack capacity normalization -> erroneous sourceCount>=movedCount Remove(source) -> ownerless remainder -> persistence can emit ITEM_DESTROY.


## Safebox / Mall — item persistence ve trust boundary

### Checkin
Client Python TItemPos
→ SendSafeBoxCheckinPacket
→ HEADER_CG_SAFEBOX_CHECKIN
→ CInputMain::SafeboxCheckin
→ character GetItem(source)
→ RemoveFromCharacter
→ CSafebox::Add
→ item window SAFEBOX + owner account_id
→ Save + FlushDelayedSave.

Explicit source window allowlist yoktur.

### Checkout
HEADER_CG_SAFEBOX_CHECKOUT / HEADER_CG_MALL_CHECKOUT
→ CInputMain::SafeboxCheckout
→ safebox Get(slot)
→ destination IsEmptyItemGrid
→ Remove(safebox slot)
→ AddToCharacter(destination)
→ FlushDelayedSave + ITEM_FLUSH.

ENABLE_SAFEBOX_IMPROVING official path INVENTORY,cell=0 gönderdiğinde server GetEmptyInventory/GetEmptyDragonSoulInventory ile güvenli auto destination seçer.

Crafted explicit destination path ise SWITCHBOT / ADDITIONAL_EQUIPMENT_1 gibi IsEmptyItemGrid tarafından tanınan windowlara ulaşabilir.

### Crafted stack-loss flow
SAFEBOX ITEM_MOVE source -> occupied compatible destination
→ count clamp
→ sourceCount >= count şartı
→ source Remove (partial transferde bile)
→ owner null / RESERVED
→ source SetCount(remainder)
→ delayed save owner-null
→ ITEM_DESTROY
→ remainder kaybı.

### Malformed persisted-row flow
DB item(window=SAFEBOX/MALL, invalid multi-size/overlap position)
→ LoadSafebox/LoadMall top-left check only
→ CSafebox::Add
→ CGrid::Put false ignored
→ m_pkItems[pos] still set
→ later Remove
→ CGrid::Get(pos,1,itemSize)
→ invalid bottom-height state'te grid OOB write riski.


## Safebox — load state lifecycle

NPC/UI password command
→ CHARACTER::ReqSafeboxLoad
→ distance/rate/pending checks
→ m_bOpeningSafebox=true
→ HEADER_GD_SAFEBOX_LOAD
→ DB password row
→ DB SAFEBOX item rows
→ HEADER_DG_SAFEBOX_LOAD
→ CInputDB::SafeboxLoad
→ conflicting-window recheck
→ LoadSafebox
→ SetOpenSafebox(true) / W_SAFEBOX
→ user CloseSafebox
→ SetOpenSafebox(false), m_bOpeningSafebox=false.

Wrong password ve conflict response flag'i temizler.
DB item-query failure GAME'e response üretmezse pending flag session boyunca takılabilir.

## Mall — weaker load policy

/mall_password
→ only password format + existing mall + throttle
→ HEADER_GD_MALL_LOAD
→ same DB password validation
→ MALL item rows
→ CInputDB::MallLoad
→ CHARACTER::LoadMall.

Mall load personal Safebox gibi W_SAFEBOX/open-position/conflicting-window state'ini set/enforce etmez.
Bu nedenle Mall access policy ayrı trust boundary olarak ele alınmalıdır.


## Mailbox — sender transaction flow

Official intended:
WRITE_CONFIRM(name)
→ DB CHECK_NAME
→ existence + mail-count result
→ client WRITE(name,title,message,itemPos,Yang,Won)
→ GAME CMailBox::Write
→ optional item RemoveFromCharacter + DestroyItem
→ sender Yang/Won debit
→ DB HEADER_GD_MAILBOX_WRITE
→ DB m_map_mailbox[name].emplace_back
→ client POST_WRITE_OK.

Security issue: WRITE packet itself is not bound to a successful confirm transaction.

## Mailbox — receiver claim flow

Mailbox open
→ DB sorts mailbox vector
→ GAME receives snapshot vecMailBox
→ GET_ITEMS(index)
→ GAME uses local snapshot[index]
→ CreateItem/AutoGiveItem
→ GiveGold/GiveCheque
→ clear local attachment
→ DB MAILBOX_GET(name,index)
→ DB clears m_map_mailbox[name][index].

Identity is positional index only.
Periodic DB erase/sort or new incoming mail can make local index and DB index refer to different logical mails.

## Mailbox persistence

Runtime source of truth in DB process:
`m_map_mailbox`.

Mutations do not immediately write SQL.
Periodic `MAILBOX_BACKUP` rewrites SQL table from the entire map.
This creates a long crash window between user-visible success and durable persistence.
