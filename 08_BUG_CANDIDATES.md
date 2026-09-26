# 08 — Bug Candidates

Bu dosyada sadece **aday** problemler tutulur. Kaynak kod + davranış testi doğrulanmadan “kesin bug” denmez.

## Kayıt formatı

### BUG-CANDIDATE-XXX
- Sistem:
- Repo / dosya:
- Belirti:
- Teknik neden:
- Etki:
- Exploit ihtimali:
- Reprodüksiyon:
- Kaynak kanıt:
- Oyun içi test:
- Durum: açık / doğrulandı / yanlış alarm / düzeltildi

## Öncelikli test alanları
- Guild Storage slot sınırları
- Invalid window_type / cell
- Eşzamanlı checkin / checkout
- Logout sırasında işlem
- Reconnect sonrası item/state tutarlılığı
- Permission bypass ihtimali

### BUG-CANDIDATE-GS-001 — Guild storage ve safebox mantığının ortak handler kullanması
- Sistem: Guild Storage
- Konum: `CInputMain::SafeboxCheckin / SafeboxCheckout`
- Gözlem: Guild Storage, `bMall == 2` ile safebox kod yolunu yeniden kullanıyor.
- Risk: bazı kontroller safebox semantiğine göre yazılmış olabilir; guild permission / concurrency kuralları handler içinde görünmüyor.
- Durum: **inceleme gerekli**, henüz bug olarak doğrulanmadı.

### BUG-CANDIDATE-GS-002 — Checkout permission kontrolü
- Gözlem: incelenen `SafeboxCheckout` gövdesinde doğrudan guild rank/permission kontrolü görünmedi.
- Not: permission daha erken storage açılışında uygulanıyor olabilir.
- Risk: packet doğrudan gönderilebiliyorsa server-side authorization eksikliği oluşabilir.
- Durum: **kritik doğrulama bekliyor**; exploit iddiası henüz yok.

### BUG-CANDIDATE-GS-003 — Load abort sonrası guild storage kilidinin takılı kalması
- Statik durum: **yüksek güven / kod yolu doğrulandı**
- Runtime test: bekliyor

Akış:
1. `ReqGuildstorageLoad()` DB request gönderir.
2. Hemen `SetStorageState(true,pid)` çağrılır.
3. Ancak gerçek `SetOpenGuildstorage(true)`, DB cevabı sonrasında `LoadGuildstorage()` içinde olur.
4. `CInputDB::GuildstorageLoad` cevabı geldiğinde başka pencere açılmışsa `CancelSafeboxLoad()` çağrılıyor.
5. `CancelSafeboxLoad()` yalnız `m_bOpeningSafebox=false` yapıyor; **`m_bOpeningGuildstorage` veya guild storage state'i temizlemiyor**.
6. Disconnect cleanup ise yalnız `IsOpenGuildstorage()==true` olduğunda state'i kapatıyor.

Sonuç adayı:
DB request ile gerçek open arasındaki hata/iptal yolları `guildstoragestate=1` bırakabilir ve sonraki açılışlar `IsStorageOpen()` nedeniyle reddedilebilir.

### BUG-CANDIDATE-GS-004 — Cross-core/channel lock senkronizasyonu
- Statik durum: **yüksek güvenli mimari risk**
- `SetStorageState` local `CGuild::m_data` değerini değiştiriyor ve SQL UPDATE yapıyor.
- İncelenen kullanımlarda bu open/close state için P2P broadcast bulunmadı.
- Başka game core/channel kendi `CGuild` objesinde eski `guildstoragestate` değerini tutabilir.
- Sonuç: aynı guild bank'ın farklı core/channel üzerinden eşzamanlı açılabilme ihtimali.
- Runtime multi-channel test gerekli.

### BUG-CANDIDATE-GS-005 — Guild ID / safebox account ID password collision
- DB guild storage load, önce `safebox.account_id = guildID` sorgusu yapıyor.
- Request password sabit `000000`.
- Guild storage için password check bypass kodu mevcut fakat yorum satırında.
- Eğer aynı numeric ID'ye sahip safebox kaydında farklı password varsa guild storage load yanlışlıkla `WRONG_PASSWORD` yoluna girebilir.
- Runtime/DB fixture testi gerekli.

### BUG-CANDIDATE-GS-006 — Item award'ın guild bank'a yönlenmesi
- `RESULT_SAFEBOX_LOAD` item-award işleme kodu `bMall=2` için de aktif.
- Mode 2'de non-mall award atlanmıyor.
- Insert sırasında `owner_id = guildID`, `window='GUILDBANK'`.
- Potansiyel sonuç: oyuncunun kişisel safebox award'ı guild storage açılırken guild bank item'ına dönüşebilir.
- Runtime testi gerekli.

### BUG-GS-007 — ENABLE_SAFEBOX_MONEY altında Guild Storage kapanışı kişisel safebox gold'unu sıfırlayabilir
- Statik durum: **kod yolu doğrulandı**
- Build koşulu: `ENABLE_GUILDSTORAGE_SYSTEM && ENABLE_SAFEBOX_MONEY`

Kanıt zinciri:
1. `LoadGuildstorage` → `new CSafebox(this, iSize, 0)`
2. constructor → `m_lGold = 0`
3. `CloseGuildstorage` → `m_pkGuildstorage->Save()`
4. ortak `CSafebox::Save()` → `dwID = character account ID`, `dwGold = 0`
5. DB → `UPDATE safebox SET gold='0' WHERE account_id=<character account>`

Sonuç:
Guild Storage kapatmak, kişisel safebox tablosundaki gold değerini sıfırlayabilir.

Not:
Bu bug yalnız ilgili compile flag aktifse çalışır. Runtime testi veri kaybı riski nedeniyle yalnız kontrollü test DB'sinde yapılmalı.

### BUG-GS-004 — Cross-core Guild Storage lock senkronizasyonu yok
- Statik durum: **mimari kod yolu doğrulandı**
- Runtime çoklu-core testi: bekliyor

`SetStorageState` local guild objesi + SQL UPDATE yapıyor.
Repo-geneli incelenen Guild P2P subheader'larında open/close lock state aktarımı yok.
`GUILD_SUBHEADER_GG_REFRESH/REFRESH1` farklı guild bilgi refresh işlerine ayrılmış.

Risk:
Farklı game core/channel'lardaki aynı guild objeleri farklı `guildstoragestate` değerleri tutabilir ve aynı guild bank'ı eşzamanlı açabilir.

### BUG-GS-008 — Yeni game core startup tüm Guild Storage kilitlerini DB'de sıfırlıyor
- Statik durum: **kod yolu doğrulandı**
- `main.cpp`: her non-auth game startup → `guild_manager.InitializeDonate()`
- `InitializeDonate()`: `UPDATE guild SET guildstoragestate = 0`

Risk senaryosu:
1. Core A'da guild storage aktif ve state=1.
2. Core B restart/startup yapar.
3. Core B global SQL ile state'i 0 yapar.
4. Yeni yüklenen/yenilenen runtime state üzerinden ikinci erişim mümkün hale gelebilir.

Ek risk:
Bu reset `guildstoragewho` alanını temizlemiyor; state=0 iken eski PID kalabilir.

### BUG-GS-009 — Yetki kaldırıldıktan sonra açık Guild Storage erişimi devam ediyor
- Statik durum: **yüksek güven / kod yolu doğrulandı**
- Build: özellikle `ENABLE_GUILDRENEWAL_SYSTEM && ENABLE_GUILDSTORAGE_SYSTEM`

Açılışta `GUILD_AUTH_BANK` kontrolü var.
Ancak `SafeboxCheckin(..., bMall=2)` ve `SafeboxCheckout(..., bMall=2)` packet yollarında:
- current guild membership
- current member grade
- current `GUILD_AUTH_BANK`
yeniden kontrol edilmiyor.

`ChangeMemberGrade` ve `ChangeGradeAuth` açık sessionı kapatmıyor.

Sonuç:
Bir oyuncunun bank yetkisi storage açıkken kaldırılırsa mevcut oturum üzerinden item koyma/çekme devam edebilir.

### BUG-GS-010 — Guildden çıkarılan/disband edilen açık-storage kullanıcısında null dereference/core crash
- Statik durum: **çok yüksek güven / birden fazla crash yolu doğrulandı**

Koşul:
`m_pkGuildstorage != nullptr`
ve ardından
`ch->SetGuild(nullptr)`

Bu durum `RemoveMember` ve `Disband` sırasında oluşabiliyor.

Crash yolları:
1. Checkin → `SaveSingleItem` → GUILDBANK owner → `GetGuild()->GetID()`.
2. Checkout sonunda GuildLog → `ch->GetGuild()->GetID()`.
3. Disconnect → `CloseGuildstorage()` → `GetGuild()->SetStorageState(...)`.

Ek problem:
`/guildstorage_close` komutu `GetGuild()==nullptr` ise erken return ediyor; açık storage objesi temizlenmiyor.

Bu nedenle guild officer'ın storage kullanan üyeyi çıkarması bile core crash'e dönüşebilir.

### BUG-GS-003 ek doğrulama — Pending load sırasında guild üyeliği kaybı
DB load cevabı geldiğinde:
`gID = ch->GetGuild() ? guildID : 0`
ve response guildID ile eşleşmezse fonksiyon direkt return ediyor.

Bu yolda:
- `m_bOpeningGuildstorage` temizlenmiyor
- daha önce set edilmiş guild storage lock temizlenmiyor

Dolayısıyla pending-load removal senaryosu stuck lock riskini güçlendiriyor.

### BUG-GS-011 — Guild disband sonrası orphan GUILDBANK itemları
- Statik durum: **yüksek güven**

Guild disband DB cleanup:
- guild
- guild_grade
- guild_member
- guild_comment

siliniyor.

Ancak guild ID owner'lı ve `window=GUILDBANK` item satırları silinmiyor.

Kaynakta ayrıca:
`//ADD_DELETE_FUNCTION_FOR_GUILD_ITEMS_IN_STORAGE`
yorumu mevcut fakat karşılığında kod yok.

Sonuç:
Disband sonrası eski guild itemları DB'de orphan kalabilir. Guild ID yeniden kullanılabilen/migrate edilen bir ortamda eski itemların başka guild tarafından görülmesi riski ayrıca test edilmeli.

### BUG-GUILD-001 — Offline member removal + ENABLE_PULSE_MANAGER null pointer
`CGuild::RemoveMember`:
`LPCHARACTER ch = FindByPID(pid)`
sonrasında `if (ch)` kontrolünden **önce**
`ch->GetPlayerID()`
kullanıyor.

`ENABLE_PULSE_MANAGER` aktif build'de offline member remove işlemi null dereference riski taşıyor.
Guild Storage dışı genel guild bug'ı olarak ayrıca kaydedildi.

### BUG-ITEM-001 — Destroy sonrası use-after-free
- Statik durum: **çok yüksek güven / doğrudan kod yolu doğrulandı**
- Build: `ENABLE_DESTROY_SYSTEM`

`CHARACTER::RemoveItem`:
1. item pointer alınır.
2. `ITEM_MANAGER::RemoveItem(item, "DESTROY")` veya doğrudan `DestroyItem(item)`.
3. `RemoveItem` sonunda `M2_DESTROY_ITEM(item)`.
4. `DestroyItem` sonunda `M2_DELETE(item)`.
5. Caller daha sonra:
   `ChatPacket(..., item->GetName())`
   çağırır.

Sonuç:
Silinmiş C++ nesnesine erişim. Allocator/build/timing'e göre crash, bozuk item adı veya görünürde sorunsuz davranış oluşabilir.

### BUG-ITEM-002 — Destroy packetindeki count yok sayılıyor
- Statik durum: **doğrulandı**

Client:
`SendItemDestroyPacket(Cell, ..., count)`

Packet:
`TPacketCGItemDestroy.count`

Server:
`CInputMain::ItemDestroy`
→ `RemoveItem(Cell, count)`

Ancak `CHARACTER::RemoveItem(..., uint8_t bCount)` içinde `bCount` stack azaltımı için kullanılmıyor.
Item count yalnız `>0` kontrol ediliyor ve ardından item objesi tamamen destroy ediliyor.

Sonuç:
UI/API partial destroy count gönderse bile tüm stack silinir.

### BUG-ITEM-003 — AddToGround başarısızlığında DropItem rollback yok
- Statik durum: **yüksek güven / hata yolu doğrulandı**

Full drop:
- item önce `RemoveFromCharacter()` ile inventory'den çıkarılıyor.

Partial drop:
- source count önce azaltılıyor
- yeni split item yaratılıyor.

Sonra:
`pkItemToDrop->AddToGround(...)`

Eğer bu false dönerse:
- source değişimi geri alınmıyor
- full-drop item envantere geri eklenmiyor
- partial source count geri artırılmıyor
- yaratılan detached item cleanup/rollback yapılmıyor
- fonksiyon sonunda yine `true` dönüyor.

Trigger düşük frekanslı olabilir; `AddToGround` başarısızlığı map index 0, zaten sectree'de olma, owner pointer kalması veya geçersiz sectree/koordinat ile oluşabilir.

### OBS-ITEM-001 — Destroy sender SendSequence çağırmıyor
`SendItemDestroyPacket`, packet `Send` başarılı olduktan sonra diğer item action sender'larının aksine `SendSequence()` çağırmadan true dönüyor.

Şimdilik yalnız gözlem:
Network transport'ın mevcut davranışında bunun gerçek paket kaybı/flush problemi oluşturup oluşturmadığı runtime/transport incelemesi gerektiriyor.

### BUG-ITEM-004 — AddToCharacter yanlış değişkenle target-cell bounds check yapıyor
- Statik durum: **doğrulandı**
- Etki: internal/DB-corruption kaynaklı crash veya inconsistent item state

`CItem::AddToCharacter(ch, Cell)`:
`pos = Cell.cell` alıyor ancak tüm overflow kontrollerinde `pos` yerine **`m_wCell`** kontrol ediyor.

Fresh/detached itemlarda `m_wCell=0` olduğu için invalid target kolayca ilk doğrulamayı geçebilir.

İkinci katman `CHARACTER::SetItem`:
- BELT: `pBeltItems[wCell]` bounds check öncesi erişiliyor.
- Dragon Soul: `pDSItems[wCell]` bounds check öncesi erişiliyor.

Bu iki window için invalid persisted target OOB erişim/core crash üretebilir.

INVENTORY gibi erken-return yapan windowlarda ise başka problem oluşur:
`SetItem` başarısızlığı AddToCharacter tarafından görülemez (void), fakat fonksiyon sonrasında `m_pOwner=ch`, `Save()`, `return true` yapar.

Güvenlik sınırı:
Normal client `CHARACTER::MoveItem` destination'ı önceden doğrular; doğrudan ITEM_MOVE exploit'i olarak işaretlenmemeli. En güçlü mevcut trigger bozuk DB item position veya yanlış internal caller'dır.

### OBS-ITEM-002 — Additional Equipment SwapItem variable shadowing
- Statik durum: **kod kusuru doğrulandı; mevcut çağrı zincirinde doğrudan runtime etkisi bulunamadı**
- Build: `ENABLE_ADDITIONAL_EQUIPMENT_PAGE`

`SwapItem` başında outer:
`TItemPos srcCell(INVENTORY,...), destCell(EQUIPMENT,...)`

oluşturuluyor; if/else içindeki aynı isimli tanımlar inner-scope olduğu için outer değerleri değiştirmiyor.

Buna rağmen mevcut swap akışında gerçek Additional Equipment hedefi daha sonra bağımsız olarak belirleniyor:
- `CheckAdditionalEquipment(wDestCell)`
- `GetAdditionalEquipmentItem(wDestCell)`
- `CItem::EquipTo`
- `GetWear/SetWear`
- `CheckAdditionalEquipment(bWearCell)`

Repo içinde görülen aktif çağrı da inventory itemı mevcut wear slotundaki itemla değiştiren `EquipItem` yoludur.

Bu nedenle shadowing şu an için item duplication/loss/misplacement bugı olarak doğrulanmadı. Refactor sırasında yanlış güvenlik varsayımı yaratabileceği için cleanup/maintainability gözlemi olarak tutulur.

### BUG-SWITCHBOT-001 — Inter-core warp CSwitchbot memory leak
- Statik durum: **çok yüksek güven / doğrudan ownership leak**

`P2PSendSwitchbot`:
1. raw pointer `pkSwitchbot = FindSwitchbot(pid)`
2. `Pause()`
3. `m_map_Switchbots.erase(pid)`
4. table kopyalanıp P2P gönderilir
5. fonksiyon biter

`delete pkSwitchbot` yok.
Map raw pointer taşıdığı için erase ownership'i serbest bırakmıyor.

Her cross-core transfer source core'da bir `CSwitchbot` allocation sızdırabilir.

### BUG-SWITCHBOT-002 — Manager Initialize/destructor raw-pointer leak
- Statik durum: **doğrulandı**

Manager objectleri `new CSwitchbot()` ile allocate ediyor.

`CSwitchbotManager::Initialize()` yalnız:
`m_map_Switchbots.clear()`

Destructor:
`~CSwitchbotManager() { Initialize(); }`

Map içindeki raw pointerlar delete edilmiyor.
Process kapanışında OS memory reclaim olsa da graceful reinitialize/test/reload yollarında destructor ownership modeli hatalı; ayrıca CSwitchbot destructor event cancellation'ı da çalışmıyor.

### BUG-SWITCHBOT-003 — Normal logout sonrası Switchbot manager entry/event cleanup yok
- Statik durum: **yüksek güven**

`CHARACTER::Disconnect` içinde Switchbot:
- Stop
- Pause
- Remove
- erase
- delete

çağrılarından hiçbiri yok.

Manager entry PID bazında runtime map'te kalıyor.

Eğer active state stale kalırsa:
`switchbot_event`
→ her 0.2s `SwitchItems`
→ item ID artık ITEM_MANAGER'da yok
→ `continue`
→ event tekrar schedule edilir.

Sonuç:
Uzun uptime'da Switchbot kullanmış/offline olmuş PID'ler için memory ve potansiyel recurring-event birikimi.

### BUG-SWITCHBOT-004 — START empty/stale slotu active event'e çevirebiliyor
- Statik durum: **doğrulandı**
- Trigger: normal UI engelliyor; custom/malformed client packet gerekir.

Server START:
- slot range var
- manager var
- already-active check var
- fakat registered item existence check yok.

Daha önce Switchbot manager oluşturmuş oyuncu boş slot için START gönderebilir:
`items[slot] == 0`
→ active=true
→ event starts
→ `ITEM_MANAGER::Find(0)` null
→ continue
→ active true kalır
→ event her 0.2s tekrar çalışır.

Aynı durum stale/nonexistent item ID için de geçerli.

Logout cleanup eksikliğiyle birleştiğinde recurring background event oyuncu çıktıktan sonra da kalabilir.

### OBS-SWITCHBOT-001 — UPDATE_ITEM vnum 8-bit
Server/client `TSwitchbotUpdateItem.vnum` alanı uint8_t iken item VNUM 32-bit.
Server assignment truncation yapıyor.

Ancak mevcut client receiver bu alanı kullanmıyor; item index normal ITEM_SET state'inden geliyor.
Şu an için kullanıcı-visible bug olarak sınıflandırılmadı, ileride bu alan kullanılmaya başlanırsa protokol hatasına dönüşür.

### BUG-ITEM-006 — Storage checkin/checkout window allowlist eksikliği
- Statik durum: **doğrulandı**
- Etki: client-controlled TItemPos ile normal MoveItem semantic guard'larının bypass edilmesi
- Handler: personal Safebox + Mall + Guild Storage ortak yolu

`SafeboxCheckout` destination için `IsEmptyItemGrid` kullanıyor ancak allowed destination window listesi tanımlamıyor.

`IsEmptyItemGrid` şu windowları da kabul ediyor:
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1.

#### SWITCHBOT
Normal MoveItem destination'da:
`SwitchbotHelper::IsValidItem(item)`
zorunlu.

Checkout'ta bu kontrol yok.
Normal/uygunsuz item, slot boşsa doğrudan SWITCHBOT window'una yerleştirilebilir ve manager'a register edilir.

#### Additional Equipment
Checkout'ta:
- CanEquipNow
- FindEquipCell
- EquipTo
- page eligibility
kontrolleri yok.

Doğrudan `AddToCharacter(ADDITIONAL_EQUIPMENT_1,...)` mümkün.

#### Checkin yönü
`SafeboxCheckin` source window için de allowlist uygulamıyor.
SWITCHBOT source item normal MoveItem active guard'ını bypass ederek storage'a alınabilir.

Client binding'in 3-arg formu explicit window_type kabul ettiği için bu yalnız wire-format teorisi değildir; değiştirilmiş Python/client tarafından üretilebilir.

Normal resmi UI davranışı ayrıca test edilmeli; server bug sınıflandırması client'ın dürüst olmasına bağlı olmamalı.

### BUG-SWITCHBOT-005 — UnregisterItem running event'i sonlandırmıyor
- Statik durum: **doğrulandı**
- BUG-SWITCHBOT-003 logout/event leak için kesin mekanizmalardan biri

`CSwitchbot::UnregisterItem` slotun:
- item ID
- active
- finished
- alternatives
state'ini temizler.

Fakat `CSwitchbotManager::UnregisterItem`:
`!HasActiveSlots() && IsSwitching()`
durumunda Stop çağırmıyor.

Son active item MoveItem dışı bir yolla kaldırılırsa running event kalır.

`switchbot_event`:
→ `SwitchItems()`
→ active slot yoksa iş yapmadan döner
→ yine `PASSES_PER_SEC(0.2f)` döndürerek schedule olur.

Normal logout sırasında CHARACTER destructor → `ClearItem()` → SWITCHBOT item `RemoveFromCharacter` → UnregisterItem zinciri bulunduğundan bu kusur custom packet'e bağımlı değildir.

### OBS-SWITCHBOT-002 — Client slot upper-bound off-by-one
`PythonSwitchbot.cpp` Start/Stop bindingleri slot guard olarak:
`if (bSlot > SWITCHBOT_SLOT_COUNT)`
kullanıyor.

Bu nedenle `slot == SWITCHBOT_SLOT_COUNT` client binding katmanından geçebilir.
Server tarafındaki `ValidPosition(slot) -> slot < SWITCHBOT_SLOT_COUNT` bunu reddettiği için mevcut server state korunuyor.

Doğru client-local sınır semantiği `>=` olmalı; şimdilik server tarafından absorbe edilen boundary observation.


### BUG-ITEM-007 — Special Inventory extend bWindow out-of-bounds
- Statik durum: **doğrulandı**
- Sınıf: client-controlled index / bounds validation eksikliği
- Build: `ENABLE_EXTEND_INVEN_ITEM_UPGRADE_SPECIAL_INV`

Client packet: `TPacketCGSendExtendInvenRequest/Upgrade.bWindow` = `uint8_t`.

Server `CInputMain::ExtendInvenRequest/Upgrade` -> `ExtendSpecialInvenRequest/Upgrade(packet->bWindow)` öncesinde `bWindow < 3` kontrolü yapmıyor.

Character helperları:
- `GetExtendSpecialInvenStage(bPage) -> bSpecialInventoryStage[bPage]`
- `SetExtendSpecialInvenStage(..., bPage) -> bSpecialInventoryStage[bPage] = ...`
- `GetExtendSpecialInvenMax(bPage)` ayrıca local 3-element base array kullanıyor.

Bu nedenle 3..255 special window:
- request/upgrade yolunda OOB read,
- sonraki index calculations'da undefined behavior,
- upgrade başarılı path'e ulaşırsa OOB write
riski oluşturuyor.

Official UI'nin 0..2 üretmesi server trust boundary için yeterli koruma değildir; client Python binding de kendi 0..2 allowlist'ini uygulamıyor.

### BUG-ITEM-008 — Locked Special Inventory slot DB restore
- Statik durum: **doğrulandı**
- Sınıf: persistence / unlock-state invariant bypass
- Doğrudan normal-client trigger: bulunmadı

Normal `MoveItem` locked special target'ı `IsEmptySpecialItemGrid -> GetExtendSpecialInvenMax` ile reddeder.

Fakat DB load: `ItemLoad -> persisted window=INVENTORY,pos -> AddToCharacter -> SetItem` yolunda target special position için character'ın unlocked max'ı zorunlu placement guard değildir.

`IsValidItemPosition(INVENTORY)` tüm static `INVENTORY_SLOT_COUNT` aralığını kabul eder. `SetItem` special branch'i target static special range içindeyse locked max üzerindeki hücreyi kesin olarak reject etmez ve item/grid pointerlarını kurabilir.

Sonuç: malformed, legacy veya başka bir bug tarafından üretilmiş DB row açılmamış special slotta item restore edebilir. Bu durum görünmeyen/erişilemeyen item ve persistence tutarsızlığına dönüşebilir.


### BUG-EXCHANGE-001 — CheckSpace/Done Special Inventory divergence
- Statik durum: **doğrulandı**
- Sınıf: preflight/commit mismatch + non-atomic transfer

`CExchange::CheckSpace()` Dragon Soul dışındaki tüm itemları regular inventory `CGrid` sayfalarında simüle eder.

`CExchange::Done()` ise `ENABLE_SPECIAL_INVENTORY` altında non-DS item için `victim->GetEmptyInventory(item)` çağırır; special type itemlar Skillbook/Stone/Material tabına yönlenir.

İki model eşdeğer değildir:
- regular inventory'de yer var ama ilgili special tab dolu/locked olabilir -> CheckSpace true, Done false.
- regular inventory dolu ama special tab boş olabilir -> CheckSpace false, gerçekte placement mümkün.

Daha kritik olan ilk durumdur. `Done()` itemları sırayla `RemoveFromCharacter -> AddToCharacter -> FlushDelayedSave` ile taşır. Sonraki itemda boş special slot bulunamazsa false döner; önce taşınan itemlar rollback edilmez.

Bu nedenle trade kısmi olarak uygulanıp ardından Cancel ile kapanabilir.

### BUG-EXCHANGE-002 — Page-4 CheckSpace grid reservation control-flow bug
- Statik durum: **doğrulandı**
- Build: extend inventory

`CheckSpace()` page 4 branch'inde:
`if (item->GetSize() > 1 && iPos > boundary)`
satırından sonra `return false` yoktur.

Preprocessor bloğu bittikten sonraki `s_grid4.Put(...)` bu `if` statement'ının gövdesi olur.

Sonuç:
- size=1 itemlarda condition false -> page4 grid slotu reserve edilmez.
- size>1 ve düzgün sığan itemlarda condition false -> yine reserve edilmez.
- sonraki incoming item aynı blank position'ı yeniden bulabilir.

Preflight bu nedenle gerçek kapasiteyi olduğundan fazla görebilir. `Done()` gerçek inventory placement'ında sonraki item için yer bulamazsa BUG-EXCHANGE-001 ile aynı non-atomic partial-transfer sınıfına girer.

### BUG-EXCHANGE-003 — Exchange item source window allowlist eksikliği
- Statik durum: **doğrulandı**
- Trigger: modified/crafted client packet gerekir

Client `netSendExchangeItemAddPacket(window_type, cell, display)` doğrudan `TItemPos(window_type,cell)` gönderir ve window allowlist uygulamaz.

Server `CExchange::AddItem`:
1. `item_pos.IsValidItemPosition()`
2. `item_pos.IsEquipPosition()` reject
3. `GetItem(item_pos)`
kontrollerini yapar.

`IsValidItemPosition()` SWITCHBOT ve ADDITIONAL_EQUIPMENT_1 windowlarını geçerli sayar. `IsEquipPosition()` yalnız EQUIPMENT windowundaki normal/DragonSoul equip alanlarını kapsar; ADDITIONAL_EQUIPMENT_1'i equip olarak sınıflandırmaz.

`CHARACTER::GetItem` her iki özel windowdan da item döndürebilir. Bu nedenle normal inventory/DS trade source semantiği server'da zorunlu tutulmuyor.

Switchbot Start itemı `SetLocked` ile kilitlemiyor; active state manager tablosunda tutuluyor. Böyle bir item exchange commit ile RemoveFromCharacter olduğunda Switchbot unregister/event lifecycle buglarıyla birleşebilir.

Additional Equipment itemı `m_bEquipped=true` olabilir; exchange source validator bunu equipment position olarak görmez. Commit RemoveFromCharacter üzerinden unequip davranışına gidebilir ve normal trade UI/CanUnequipNow sınırlarını bypass eden bir yol oluşur.


### BUG-EXCHANGE-001 — Special Inventory CheckSpace/Done domain mismatch → partial transfer
- Statik durum: **doğrulandı**
- Sınıf: transaction preflight/commit mismatch + rollback eksikliği
- Build: `ENABLE_SPECIAL_INVENTORY`

`CExchange::CheckSpace()` Dragon Soul dışındaki incoming itemları yalnız normal inventory page gridlerinde simüle eder.

`CExchange::Done()` ise special item için `victim->GetEmptyInventory(item)` kullanır; bu helper item type'a göre Skillbook/Stone/Material special inventory range'ine gider.

Bu nedenle normal inventory'de alan varken ilgili special inventory dolu olabilir:
- `CheckSpace()` true
- `Done()` special itema geldiğinde `empty_pos < 0`.

`Done()` daha önceki itemları çoktan `RemoveFromCharacter -> AddToCharacter(victim)` ile taşıdıysa rollback yapmaz. Offer array sırası nedeniyle önce normal item, sonra başarısız special item olduğunda one-sided/partial item transfer statik olarak mümkündür.

Ek olarak normal inventory dolu ama special inventory boş olduğunda false-negative trade rejection oluşur.

### BUG-EXCHANGE-002 — Extended inventory page 4 reservation hatası → CheckSpace false-positive
- Statik durum: **doğrulandı**
- Build: `ENABLE_EXTEND_INVEN_ITEM_UPGRADE`
- Sınıf: space-simulation control-flow bug + transaction atomicity

`CExchange::CheckSpace()` page4 branch'inde `s_grid4.Put(iPos,...)` şu multi-size boundary condition'ın statement'ı haline gelmiş:
`if (item->GetSize() > 1 && iPos > wSlotPos - (...))`

Sonuç:
- size=1 item için condition false → `Put` çalışmaz,
- normal şekilde sığan size>1 item için de çoğu durumda `Put` çalışmaz,
- boundary'yi aşan bazı size>1 durumlarda tersine `Put` çalışır.

En basit etkisi: page4'te tek boş slot varken birden fazla size=1 incoming item aynı slotu preflight'ta tekrar kullanabilir. `CheckSpace()` true döndükten sonra gerçek `Done()` ilk itemı yerleştirir, sonraki item boşluk bulamayabilir. Önce taşınan item rollback edilmez.

### BUG-EXCHANGE-003 — Exchange source-window allowlist eksikliği
- Statik durum: **doğrulandı**
- Sınıf: client-controlled TItemPos / semantic guard bypass

Official UI ITEM_ADD source olarak Inventory veya Dragon Soul kullanır. Fakat Python binding explicit `window_type` kabul eder.

Server `CExchange::AddItem` yalnız:
- generic `IsValidItemPosition`
- `!IsEquipPosition`
- item existence / anti-give / lock / exchange state
kontrollerini uygular.

Bu, `SWITCHBOT` ve `ADDITIONAL_EQUIPMENT_1` gibi generic olarak valid fakat exchange için beklenmeyen source windowları geçirir.

Etkiler:
- active SWITCHBOT item normal `MoveItem` içindeki `CSwitchbotManager::IsActive` source guardını bypass ederek exchange offer'a girebilir,
- Additional Equipment item normal move yolundaki `CanUnequipNow` semantiğini bypass edebilir.

Runtime sonuçları izole testte doğrulanmalı; server-side allowlist eksikliği statik olarak kesindir.

### OBS-EXCHANGE-001 — TPacketCGExchange zero-init eksikliği
Client `SendExchange*` fonksiyonları `TPacketCGExchange packet;` oluşturup yalnız subheader'a gerekli alanları dolduruyor.

Server `CInputMain::Exchange` switch öncesinde her packet için `pinfo->arg1` character lookup yapıyor. START dışındaki paketlerde `arg1` initialize edilmemiş olabilir. Bu nondeterministic early-return ve client stack data'nın gereksiz wire'a çıkması açısından gözlem olarak tutuluyor.

### OBS-EXCHANGE-002 — AddGold cheque-build boolean kontrolü
`ENABLE_CHEQUE_SYSTEM` altında offer-stage balance kontrolü gold ve cheque yetersizliğini `&&` ile birleştiriyor. Tek currency yetersizliği AddGold aşamasından geçebilir. Final `CExchange::Check()` gold ve cheque'yi ayrı kontrol ettiği için şu an doğrudan currency exploit olarak sınıflandırılmadı; UX/state inconsistency ve future-regression riski.


### BUG-EXCHANGE-004 — Currency overflow TOCTOU can lose offered currency
- Statik durum: **doğrulandı**
- Sınıf: time-of-check/time-of-use + unchecked void mutation

`CInputMain::Exchange(ELK_ADD)` recipient için teklif anında:
- `recipient gold + offer < GOLD_MAX`
- `recipient cheque + offer < CHEQUE_MAX`
kontrolü yapar.

Fakat bu yalnız offer creation anındaki snapshot'tır. Final `Accept()` içinde recipient currency overflow tekrar doğrulanmaz.

`CExchange::Done()` gold transferinde:
1. sender `PointChange(POINT_GOLD, -m_lGold)`
2. recipient `PointChange(POINT_GOLD, +m_lGold)`
çalıştırılır.

`PointChange(POINT_GOLD)` overflow olduğunda void olarak erken `return` eder. `Done()` bunu göremez ve transferi başarılı kabul etmeye devam eder. Böylece sender'dan gold düşüp recipient'a ekleme yapılmaması mümkündür.

Reachability doğrulaması: exchange açıkken `CanHandleItem()` genel exchange guard içermez; ayrıca ground `PickupItem()` exchange state kontrol etmeden ITEM_ELK için `GiveGold()` çalıştırır. Recipient offer oluşturulduktan sonra kendi gold bakiyesini artırabilir.

Cheque tarafında `Done()` transferden hemen önce overflow'u tekrar kontrol eder ve false döner; ancak bu kontrol item ve gold mutationlarından **sonra** olduğu için cheque overflow da trade'i orta-commit'te durdurup önceki mutationları rollback etmez.

### OBS-EXCHANGE-001 — AddGold cheque koşullarında conjunction kullanımı
`CExchange::AddGold()` cheque build altında:
- insufficient funds check'i `gold insufficient && cheque insufficient`
- existing offer check'i `m_lGold > 0 && m_lCheque > 0`
şeklinde yapıyor.

Bu, tek currency yetersizken AddGold'un geçici olarak offer state yazmasına veya yalnız bir currency varken offer'ın overwrite edilmesine izin verir. Final `Check()` sender funds'ı iki currency için ayrı ayrı kontrol ettiği için tek başına completed transfer exploit'i statik olarak gösterilmedi. Şimdilik observation.


### BUG-EXCHANGE-004 — Final distance recheck yok / client-side range enforcement
- Statik durum: **doğrulandı**
- Sınıf: server trust-boundary / state invariant
- Etki: modified client ile başlangıçtan sonra uzaklaşıp remote trade completion.

ExchangeStart mesafeyi < EXCHANGE_MAX_DISTANCE (1000) kontrol eder. Normal movement exchange'i server-side cancel etmez ve final CExchange::Accept current distance'ı yeniden kontrol etmez.

Official uiexchange.py::OnUpdate 1000 mesafe aşılırsa client tarafından CANCEL gönderir. Bu nedenle koruma server invariant değil, client davranışıdır.

### BUG-EXCHANGE-005 — Gold recipient cap TOCTOU → sender debit without receiver credit
- Statik durum: **doğrulandı**
- Sınıf: currency transaction / TOCTOU / asymmetric commit
- Etki: gold loss / griefing-risk; gain exploit olarak sınıflandırılmadı.

ELK_ADD offer anında recipient gold cap kontrol edilir. Fakat exchange açıkken server ITEM_PICKUP kabul eder ve PickupItem exchange state kontrolü olmadan ITEM_ELK için GiveGold çalıştırır. Party pickup distribution da recipient gold'unu değiştirebilir.

Final Check() recipient gold cap'i tekrar doğrulamaz. Done() önce sender debit, sonra receiver credit uygular. Receiver addition GOLD_MAX overflow nedeniyle PointChange içinde early-return edebilir. Fonksiyon void olduğundan Done() bunu algılayamaz. Sender debit zaten uygulanmıştır ve rollback yoktur.

### BUG-EXCHANGE-001/002 persistence severity update
Her başarılı moved item sonrasında ITEM_MANAGER::FlushDelayedSave(item) -> SaveSingleItem -> HEADER_GD_ITEM_SAVE gönderilir. Daha sonraki item fail olduğunda Cancel() bu item ownership değişimini geri almaz. Bu nedenle partial transfer DB cache'e persist edilebilir.

### Exchange lifecycle — güvenli kapatılan alanlar
- character teardown/disconnect -> Cancel
- death -> Cancel
- active ENABLE_CHECK_WINDOW_RENEWAL + SetExchange(W_EXCHANGE) nedeniyle standard CanWarp() active exchange sırasında false.

Doğrudan WarpSet() kendi içinde exchange cancel/check yapmadığından özel callerlar ayrıca taranabilir, ancak standart warp için bug olarak sınıflandırılmadı.


### BUG-EXCHANGE-003 — concrete impact update
Source-window allowlist eksikliğinin iki somut etkisi statik olarak kapatıldı:

**Active Switchbot:**
- Exchange AddItem active slotu reddetmez.
- Switchbot event item_id ile itemı bulur ve item IsExchanging olsa da ChangeAttribute çalıştırabilir.
- Trade UI'ye gönderilmiş item attribute snapshot'ı accept öncesinde değişebilir.
- Transferde source SWITCHBOT slot unregister edilir; sorun özellikle offer→accept integrity penceresidir.

**Additional Equipment:**
- ADDITIONAL_EQUIPMENT_1 generic valid window'dur fakat IsEquipPosition değildir.
- Normal MoveItem CanUnequipNow uygular; Exchange AddItem uygulamaz.
- CanUnequipNow ITEM_FLAG_IRREMOVABLE dahil unequip policy uygular.
- Done -> RemoveFromCharacter -> Unequip bu policy'i yeniden uygulamaz.

Bu nedenle modified client, normal item-move kurallarınca çıkarılması engellenecek Additional Equipment itemını exchange transfer yoluna sokabilir.

### Exchange static completion note
BUG-EXCHANGE-001..005 ve OBS-EXCHANGE-001/002 ile ana statik risk seti çıkarıldı. Bundan sonraki Exchange işi öncelikle EX-T01..EX-T09 runtime doğrulamasıdır.


### BUG-SHOP-001 — Premium shop sale commits before DB stash acknowledgement
- Statik durum: **cross-process atomicity gap doğrulandı**
- Etki koşulu: DB/cache bağlantı veya packet işleme başarısızlığı

Premium `CShop::Buy` sırası:
1. buyer gold/cheque yeterlilik kontrolü
2. buyer destination hesaplama
3. buyer currency debit
4. seller-shop item `RemoveFromCharacter`
5. buyer `AddToCharacter`
6. item `FlushDelayedSave`
7. local shop slot clear/broadcast
8. GAME -> DB `HEADER_GD_SHOP / SHOP_SUBHEADER_GD_BUY(pid,pos)`

Seller proceeds doğrudan GAME'de verilmez. DB `CClientManager::ShopSaleResult` ilgili DB-side shop itemını bulur, stored price/cheque değerini stash'e ekler, itemı DB shop tablosundan kaldırır ve cache'e yazar.

`CShop::Buy` commit öncesinde `db_clientdesc->GetSocket()` kontrolü yapmıyor ve DB tarafında sale result için synchronous acknowledgement beklemiyor. GAME tarafında debit/item transferini geri alan rollback yolu da yok.

Sonuç: DB sale notification işlenmezse item + buyer debit kalıcılaşabilirken seller stash credit'i gerçekleşmeyebilir. Bu durum normal gameplay exploitinden çok availability/persistence bütünlüğü bugıdır.

### BUG-SHOP-002 — Empty Private Shop Search result uses vector[0]
- Statik durum: **doğrulandı**
- Build: `ENABLE_PRIVATESHOP_SEARCH_SYSTEM`

Search response sonunda:
`ch->GetDesc()->Packet(&vecPrivateShopSearchItem[0], sizeof(TPrivateShopSearchItem) * vecPrivateShopSearchItem.size());`

çağrısı vektör boşken de çalışıyor.

`std::vector::operator[](0)` empty vector için undefined behavior'dır; packet length 0 olsa bile pointer ifadesi güvenli değildir.

Beklenen güvenli şekil:
- empty ise yalnız header gönder,
- veya C++11+ `vec.data()` kullan ve transport'ın zero-length semantics'ini açık tut.

Runtime/ASan testi boş search result ile yapılmalı.


### BUG-SHOP-001 — Premium Private Shop stash cap silently clips sale proceeds
- Statik durum: **doğrulandı**
- Build: ENABLE_PREMIUM_PRIVATE_SHOP
- Sınıf: currency cap / asymmetric sale commit

DB ShopSaleResult credits sold.price/sold.cheque through AlterGoldStash/AlterChequeStash. Those functions add then clamp to GOLD_MAX/CHEQUE_MAX.

There is no sale precheck ensuring stash + sale <= cap.

Buyer has already been charged and item ownership already transferred before DB stash credit. Therefore seller can receive only part of the proceeds while sale still completes.

### BUG-SHOP-002 — personal_shop tax not applied to premium private shop stash
- Statik durum: **doğrulandı**
- Build: ENABLE_PREMIUM_PRIVATE_SHOP
- Sınıf: cross-layer accounting mismatch

CShop::Buy computes tax and reduces local dwPrice after buyer was charged. In premium branch seller is not credited from this local dwPrice.

Game sends DB only seller pid + display pos. DB ShopSaleResult reloads cached sold entry and executes AlterGoldStash(sold.price,true), so full listed price enters stash. Net price/tax is absent from packet.

Effect: game-side personal_shop tax calculation does not reduce premium seller stash proceeds.

### OBS-SHOP-001 — item/currency/shop-cache persistence is non-atomic
Premium sale ordering spans three save domains: item FlushDelayedSave, DB shop sale/cache mutation, buyer character delayed Save. A process/connection failure between stages can produce divergent persisted state. Runtime fault-injection is required before classifying a concrete crash-recovery outcome.


## Canonical Shop bug index — 2026-09-26 second pass

> Bu bölüm Shop/Private Shop için canonical numaralandırmadır ve yukarıdaki provisional/çakışan BUG-SHOP numaralarını supersede eder.

### BUG-SHOP-001 — Premium sale cross-process atomicity gap
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP`
- Sınıf: GAME item/currency commit -> DB stash commit arasında transaction/ack eksikliği

Premium `CShop::Buy` buyer debit + item ownership transfer + `FlushDelayedSave` yaptıktan sonra DB'ye yalnız `pid + display_pos` sale bildirimi yollar.
DB `ShopSaleResult` seller stash credit + shop table removal yapar.
GAME tarafı synchronous acknowledgement/rollback beklemez.

DB packet kaybı/crash/peer failure aralığında buyer debit ve item transferi persist olurken seller stash credit'i eksik kalabilir.

### BUG-SHOP-002 — Empty Private Shop Search result vector[0] UB
- Statik durum: **doğrulandı**
- Build: `ENABLE_PRIVATESHOP_SEARCH_SYSTEM`

Search sonucu boşken `&vecPrivateShopSearchItem[0]` ifadesi oluşturuluyor.
Packet size 0 olsa bile empty vector üzerinde `operator[](0)` undefined behavior'dır.
ASan/runtime empty-result testi gerekir.

### BUG-SHOP-003 — Premium personal_shop tax accounting mismatch
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP`

GAME `CShop::Buy` buyer'dan full listed price düşer, ardından `personal_shop` tax hesaplayıp local `dwPrice` değerini net'e indirir.
Premium seller credit bu local net değeri kullanmaz.
DB sale packet yalnız `pid + display_pos` taşır; DB cached `sold.price` üzerinden **full listed price** seller stash'e ekler.

Sonuç: premium private shop satışında GAME'de hesaplanan personal_shop tax seller proceeds'ten düşülmez.

### BUG-SHOP-004 — TransferItemAway off-by-one -> vector OOB
- Statik durum: **doğrulandı**
- Trigger: crafted owner remove-item packet
- Sınıf: bounds / memory safety

`CShop::TransferItemAway(ch, pos, ...)` kontrolü:
`if (pos > m_itemVector.size()) return false;`

Doğru sınır `pos >= size` olmalı.
`pos == m_itemVector.size()` geçer ve hemen ardından:
`SHOP_ITEM& r_item = m_itemVector[pos];`
ile out-of-bounds erişim oluşur.

`TPacketMyShopRemoveItem.slot` int'tir ve caller bunu `uint8_t` olarak geçirir; aktif shop vector size 90 iken slot=90 erişilebilir crafted input'tur.

### BUG-SHOP-005 — Initial MyShop bCount > host max -> transfer-before-validation orphan state
- Statik durum: **doğrulandı**
- Build: `ENABLE_PREMIUM_PRIVATE_SHOP + ENABLE_MYSHOP_DECO`
- Aktif limitler: shop grid = 10x9 = 90 hücre; `SHOP_HOST_ITEM_MAX = 80`

`TPacketCGMyShop.bCount` uint8_t ve `CInputMain::MyShop` / `CHARACTER::OpenMyShop` tarafında `bCount <= SHOP_HOST_ITEM_MAX` explicit guard yok.

`CShopManager::CreatePCShop` sırası:
1. `TransferItems(owner,pTable,bItemCount)`
2. `SetShopItems(pTable,bItemCount)`

`SetShopItems` ancak **transferden sonra** `bItemCount > SHOP_HOST_ITEM_MAX` deyip return eder.

Crafted `bCount=81` ile source itemlar PREMIUM_PRIVATE_SHOP window'una taşınıp item save'i flush edilebilir; ardından shop listing vector/table kurulmaz.
Bu, runtime item ile persisted shop metadata arasında orphan/inaccessible item state oluşturabilir.

### BUG-SHOP-006 — Duplicate display_pos during initial shop creation can overwrite/orphan runtime item
- Statik durum: **doğrulandı**
- Trigger: crafted initial MyShop item table

`CHARACTER::OpenMyShop` duplicate **source TItemPos** kontrol eder fakat duplicate `display_pos` kontrol etmez.

Initial `TransferItems` sırasında `m_pGrid->IsEmpty(display_pos,...)` çağrılır fakat bu fonksiyon içinde grid'e `Put` yapılmaz.
Aynı display_pos'a iki source item bu aşamadan geçebilir.

İkinci item `AddToCharacter(PREMIUM_PRIVATE_SHOP, same_cell)` yaptığında `CHARACTER::SetItem` mevcut `pShopItems[cell]` pointer'ını reddetmeden yeni pointer ile overwrite eder.
İlk item owner/window/cell state'ini koruyup runtime lookup'tan kopabilir.

Ardından `SetShopItems` ilk listing sırasında aynı hücrede son overwrite edilen itemı okuyabilir ve ikinci listing grid collision nedeniyle reddedilebilir.
Sonuç: item ownership/listing/shopItems metadata tutarsızlığı ve orphan riski.

### BUG-SHOP-007 — Withdraw stash TOCTOU -> DB stash debit without player credit
- Statik durum: **doğrulandı**
- Build: premium private shop
- Sınıf: async request/response + unchecked void currency mutation

`CInputMain::WithdrawShopStash` request anında:
- requested <= local shop stash
- player gold + requested < GOLD_MAX
- cheque + requested < CHEQUE_MAX
kontrollerini yapar.

DB `WithdrawShopGold` stash'i **önce azaltır** ve success result yollar.

GAME `CInputDB::WithdrawGoldResult` success geldiğinde cap'i tekrar doğrulamaz:
1. local shop stash azaltılır
2. `PointChange(POINT_GOLD,+amount)`
3. cheque için aynı

Request ile response arasında player gold/cheque başka bir yoldan yükselirse `PointChange` overflow'da void early-return eder.
DB stash debit zaten uygulanmış olduğundan ve bu branch rollback yollamadığından gelir kalıcı kaybolabilir.

### OBS-SHOP-001 — stash clamp tek başına normal-flow exploit değil
DB `AlterGoldStash/AlterChequeStash` add sonrası clamp uygular.
Ancak normal open/add akışları `current stash + tüm listed values < cap` invariantını korur.
Dolayısıyla önceki “her satışta cap clipping doğrudan reachable” provisional sınıflandırması **downgrade** edilmiştir.

Clipping hâlâ:
- state desync,
- crafted count/display corruption,
- arithmetic/config edge,
- crash/recovery divergence
sonrası ikincil etki olabilir.

### OBS-SHOP-002 — cross-empire 3x uint32 overflow, default configte dormant
Normal shop buy path'inde başka imparatorluk için `uint32_t dwPrice *= 3` overflow guard olmadan uygulanır.

Server default:
`g_bEmpireShopPriceTripleDisable = true`
yani 3x fiyat varsayılan olarak kapalıdır.

Runtime config ile `SHOP_PRICE_3X_TAX` açılırsa yüksek listed price ×3 32-bit wrap yapabilir.
Premium DB seller credit'i cached original listed price üzerinden yaptığı için bu config kombinasyonu ayrıca ekonomik test gerektirir.

### Shop static second-pass güvenli kapatılanlar
- Initial source window: OpenMyShop yalnız INVENTORY / DRAGON_SOUL_INVENTORY kabul ediyor.
- AddMyShopItem source window: explicit INVENTORY / DRAGON_SOUL_INVENTORY allowlist var.
- PrivateShopSearchBuy: shop existence + editing/closed + visibility + same-map + VIEW_RANGE checks mevcut.
- NPC Sell: active viewed shop must be NPC shop, CanHandleItem/distance/item-lock/seal/anti-sell/gold-cap kontrolleri mevcut.
- Last-item manual remove: TransferItemAway -> CloseMyShop -> Save() ile full closed shop table DB'ye flush ediliyor.


## Shop canonical third-pass update — boot/reload + size-domain audit

### BUG-SHOP-004 reachability refinement
`TransferItemAway` içindeki `if (pos > m_itemVector.size())` off-by-one statik olarak gerçektir; ancak temiz/resmî client state'inde shop display slotları 0..79 olduğu için `pos == 80` bağımsız normal-flow trigger değildir.

Aktif build'de `m_itemVector.size()==80`, fakat PREMIUM_PRIVATE_SHOP item window/grid 90 hücre kabul ettiği için bu bug özellikle BUG-SHOP-008 ile üretilmiş/corrupt 80..89 slot state'inde reachable olur. Bu nedenle BUG-SHOP-004 **dependent memory-safety hardening bug** olarak tutulur.

### BUG-SHOP-008 — 90-slot server grid vs 80-entry shop vector → crafted display_pos OOB
- Statik durum: **doğrulandı**
- Trigger: modified/crafted client
- Sınıf: server-side bounds / memory safety

Aktif sabitler:
- `SHOP_GRID_WIDTH=10`
- `SHOP_GRID_HEIGHT=9`
- `SHOP_INVENTORY_MAX_NUM=90`
- `SHOP_HOST_ITEM_MAX=80`

`CShop::SetShopItems` premium MYSHOP_DECO build'inde:
`m_itemVector.resize(SHOP_HOST_ITEM_MAX)` → 80 entry.

Fakat `CShop::SetShopItem`:
1. `iPos = pTable->display_pos`
2. `m_pGrid->IsEmpty(iPos,...)`
3. `m_pGrid->Put(iPos,...)`
4. `SHOP_ITEM& item = m_itemVector[iPos]`

şeklinde ilerliyor ve `iPos < m_itemVector.size()` kontrolü yok.

CGrid 10x9 olduğu için display_pos 80..89 grid kontrolünden geçebilir; ardından 80-entry vector OOB erişimi oluşur.

**Reachability:**
- `CInputMain::AddMyShopItem` source window için allowlist uygular, fakat `p->targetPos` için 0..79 server bound uygulamaz.
- Packet target `int`, TShopItemTable `display_pos` uint8.
- `TransferItems` 80..89'u 90-slot PREMIUM_PRIVATE_SHOP window'una taşıyabilir.
- hemen sonraki `SetShopItem` vector OOB'a gider.
- Initial MyShop table için de aynı display_pos sınıfı geçerlidir.

Resmî client UI 5x8 = 40 slot/page ve 2 page kullanır; yani normal UI 0..79 üretir. Python/C++ send binding ise arbitrary int target alır ve server limiti enforce etmez.

### BUG-SHOP-009 — ClosePlayerShop Special Inventory preflight/commit mismatch → partial close
- Statik durum: **doğrulandı**
- Build: `ENABLE_SPECIAL_INVENTORY`
- Sınıf: preflight/commit mismatch + non-atomic item recovery

`CInputMain::ClosePlayerShop` tüm non-DragonSoul shop itemlarını önce yalnız dört regular inventory `CGrid` üzerinde simüle eder.

Gerçek transfer loop'unda ise:
`ch->GetEmptyInventory(item)`
kullanılır; Special Inventory itemları Skillbook/Stone/Material domainlerine yönlenebilir.

Sonuçlar:
- regular inventory'de yer var ama ilgili special inventory dolu/locked → preflight true, gerçek transfer daha sonra fail.
- regular inventory dolu ama special inventory boş → false-negative close rejection.

Daha kritik ilk durumda transferler sırayla `TransferItemAway` ile uygulanır ve her item save edilir / DB remove packetleri gönderilir. Sonraki item fail olursa önce taşınan itemlar rollback edilmez; shop kısmen kapanmış/boşalmış state'te kalabilir.

### BUG-SHOP-010 — Shop cache persistence DELETE + INSERT non-transactional crash window
- Statik durum: **doğrulandı**
- Sınıf: DB persistence atomicity / crash recovery

`CShopCache::OnFlush` shop item metadata için ayrı AsyncQuery'ler çalıştırır:
1. `DELETE FROM private_shop_items WHERE pid=...`
2. ayrı `INSERT INTO private_shop_items (...)`

SQL transaction yoktur.

DB process/core crash veya connection failure DELETE uygulandıktan sonra INSERT uygulanmadan gerçekleşirse:
- actual item rows `item.window=PREMIUM_PRIVATE_SHOP` olarak kalabilir,
- fakat fiyat/display metadata `private_shop_items` kaybolabilir.

Boot loader `private_shop_items INNER JOIN item` kullandığı için bu itemlar shop cache reconstruction'a girmeyebilir. Character item load tarafında PREMIUM_PRIVATE_SHOP item row'ları mevcut kalabildiğinden inaccessible/orphan shop-item state oluşabilir.

### OBS-SHOP-003 — MyShopInfoLoad position-index robustness
DB `SendMyShopInfo` active build'de 80-entry price-info domain kullanır ve display position check'i `item.display_pos > SHOP_HOST_ITEM_MAX` şeklindedir; equality 80 reddedilmez.

GAME `MyShopInfoLoad`:
`std::array<TMyShopPriceInfo, SHOP_HOST_ITEM_MAX> info;`
oluşturup bounds check olmadan:
`info[p->pos] = *p`
ve daha sonra
`info[item.pos]`
okur.

Normal official state 0..79 ile çalışır. BUG-SHOP-008 / DB corruption / stale persisted slot 80+ sonrası OOB read/write mümkündür.

Ayrıca `info` value-initialize edilmediği için metadata'sı eksik bir position okunursa POD alanları uninitialized olabilir. Bu gözlem özellikle BUG-SHOP-010 crash-recovery state'i ile birlikte runtime/ASan test edilmelidir.

### OBS-SHOP-004 — Official client Won withdraw width mismatch
Client packet `TPacketCGShopWithdraw.chequeAmount` uint32_t olmasına rağmen:
`CPythonNetworkStream::WithdrawMyShopMoney(uint32_t goldAmount, uint8_t chequeAmount)`
olarak tanımlı.

Python binding `int chequeAmount` okur, sonra bu uint8_t parametreye daralır ve packet'e uint32 olarak yazılır.

Sonuç: resmî client üzerinden 255 üstü Won/cheque withdraw miktarı modulo-256/truncate davranışı gösterebilir. Server crafted packet'te uint32 kabul eder; bu server integrity exploiti değil, client functional bugıdır.

### Shop boot/expiry/static lifecycle closure
- DB boot: `private_shop_items` ile `item(window=PREMIUM_PRIVATE_SHOP)` pid+pos üzerinden JOIN edilir.
- GAME boot: fake CHARACTER oluşturulur, persisted itemlar fake shop window'una bağlanır, sonra `SpawnShop -> CreatePCShop` üzerinden detached shop char'a transfer edilir.
- Expiry/delete: `ITEM_MANAGER::RemoveItem`, PREMIUM_PRIVATE_SHOP itemında `shop->RemoveItemByID` çağırır; runtime vector + owner shopItems + DB remove/closed save zinciri mevcut.
- Last item expiry/removal `CloseMyShop -> Save()` ile full closed table flush yapar.
- Client add/remove bindings target/slot değerlerini server limitine göre sanitize etmez; server validation bu nedenle security boundary olmalıdır.

### Shop static completion note
Premium Private Shop / NPC Shop ana statik haritası bu turla **static completion** seviyesine alınmıştır.

Canonical high-confidence set:
- BUG-SHOP-001..003
- BUG-SHOP-005..010
- BUG-SHOP-004 dependent hardening/reachability
- OBS-SHOP-001..004

Bundan sonraki Shop işi öncelikle runtime/ASan/fault-injection test matrisidir.


## Safebox / Mall — canonical first pass (2026-09-26)

### BUG-SAFEBOX-001 — Mall close overwrites persisted Safebox gold with zero
- Statik durum: **doğrulandı**
- Build: `ENABLE_SAFEBOX_MONEY`
- Reachability: **normal official flow**
- Etki: persisted personal Safebox gold kaybı

`CHARACTER::LoadMall` Mall nesnesini `CSafebox(this, 3 * SAFEBOX_PAGE_SIZE, 0)` ile oluşturur. `CHARACTER::CloseMall` ise `m_pkMall->Save()` çağırır. `CSafebox::Save()` SAFEBOX/MALL ayrımı yapmadan account ID + `m_lGold` ile `HEADER_GD_SAFEBOX_SAVE` yollar. DB `QUERY_SAFEBOX_SAVE`, `safebox.gold` alanını bu değerle günceller.

Sonuç: Mall açılıp normal kapatıldığında Mall instance'ın 0 gold değeri personal Safebox'ın persisted gold alanının üzerine yazılabilir.

### BUG-SAFEBOX-002 — Safebox gold withdraw signed-overflow + debit-before-credit loss
- Statik durum: **doğrulandı**
- Build: `ENABLE_SAFEBOX_MONEY`

Withdraw precheck `ch->GetGold() + static_cast<int>(p->dwMoney) >= GOLD_MAX` ifadesinde signed-int toplama kullanır. `GOLD_MAX=2,000,000,000`; ayrı ayrı valid iki değer toplamda INT_MAX'i aşabilir.

Commit sırası önce `SetSafeboxMoney(current-amount)`, sonra `PointChange(POINT_GOLD,+amount)`. PointChange int64 toplamla overflow'u reddedebilir fakat void döndüğü için önceki safebox debit rollback edilmez.

### BUG-SAFEBOX-003 — Safebox stack merge removes source on partial/zero transfer
- Statik durum: **doğrulandı**
- Reachability: crafted/modifiye client

`CSafebox::MoveItem` stack branch count'u destination kapasitesine göre küçülttükten sonra `if (item->GetCount() >= count) Remove(bCell);` yapar. Bu koşul partial transferde de ve destination full olup `count=0` olduğunda da true olabilir.

`Remove` source'u slot/grid'den çıkarıp owner'ı null / window RESERVED yapar. Ardından kalan count ownerless itemda kalır; delayed persistence owner-null item için ITEM_DESTROY gönderebilir.

Official UI safebox->safebox MoveItem packetini yalnız empty-slot eventinde gönderir; occupied stack hedefi normal UI'den üretilmez.

### BUG-SAFEBOX-004 — Safebox/Mall semantic TItemPos allowlist eksikliği
- Statik durum: **doğrulandı**
- Cross-reference: BUG-ITEM-006

Checkin source `TItemPos` için semantic allowlist yoktur. Checkout explicit destination'da `IsEmptyItemGrid` kullanılır; bu helper SWITCHBOT ve ADDITIONAL_EQUIPMENT_1'i de kabul eder. Non-DS path INVENTORY/BELT-only allowlist uygulamaz.

Modified client ile normal MoveItem guardlarını bypass eden storage <-> SWITCHBOT/Additional Equipment yolları mümkündür.

### OBS-SAFEBOX-001 — CSafebox::Add grid Put sonucu kontrol edilmiyor
`CSafebox::Add` top-left position validity sonrası `m_pkGrid->Put(...)` dönüşünü kontrol etmeden `m_pkItems[dwPos]=item` yapar. Normal checkin `IsEmpty` ile korunur; malformed/overlapping persisted DB rowlarında grid ile item array state'i ayrışabilir.

### Safebox first-pass güvenli sonuçlar
- `CreateItemTableFromRes` stale static-vector leakage üretmiyor: no-row'da clear, normal durumda resize.
- `CSafebox::ChangeSize` shrink yapmıyor; yalnız büyütüyor.
- SAFEBOX/MALL item persistence owner=account_id üzerinden ilerliyor.


## Safebox / Mall — canonical second pass / active-build correction (2026-09-26)

### Build correction
Aktif server build:
- `ENABLE_SAFEBOX_IMPROVING` = ON
- `ENABLE_SPECIAL_INVENTORY` = ON
- `ENABLE_SWITCHBOT` = ON
- `ENABLE_ADDITIONAL_EQUIPMENT_PAGE` = ON
- `ENABLE_SAFEBOX_MONEY` = **OFF**

Bu nedenle daha önce kaydedilen:
- BUG-SAFEBOX-001 (Mall close -> safebox.gold=0)
- BUG-SAFEBOX-002 (gold withdraw overflow/debit-before-credit)

kod seviyesinde gerçek kusurlar olmakla birlikte **mevcut build'de derlenmiyor / dormant** durumdadır. SAFEBOX_MONEY ileride açılırsa yeniden kritik hale gelirler.

### BUG-SAFEBOX-003 — Safebox stack merge source-loss
- Statik durum: **doğrulandı / aktif build**
- Reachability: modified/crafted client
- Etki: item remainder kaybı / DB destroy

`CSafebox::MoveItem` occupied stack branch:
1. requested count destination kapasitesine clamp edilir.
2. ardından `if (item->GetCount() >= count) Remove(bCell);` çalışır.

Clamp sonrası `count <= source count` olduğu için bu koşul partial merge'de de true olur. Destination full ise count=0 olur ve source yine Remove edilir.

`Remove()` source itemı SAFEBOX runtime slot/gridinden çıkarır, `RemoveFromCharacter()` owner'ı null ve window'u RESERVED yapar. Ardından `SetCount(remainder)` ownerless itemı delayed-save kuyruğuna sokabilir. `ITEM_MANAGER::SaveSingleItem` owner yoksa ITEM_DESTROY yollar; ayrıca eventual DestroyItem da non-skip item için destroy packet üretir.

Official `uisafebox.py` occupied-target safebox move packet'i üretmediği için normal UI yolu yoktur; server crafted input'a karşı korunmasızdır.

### BUG-SAFEBOX-004 — Safebox/Mall semantic TItemPos bypass
- Statik durum: **doğrulandı / aktif build**
- Cross-reference: BUG-ITEM-006
- Reachability: modified client

Checkout non-DS yolu:
- caller-controlled `p->ItemPos`
- `IsEmptyItemGrid(dest,...)`
- Special Inventory type equality
- sonra doğrudan `AddToCharacter(dest)`.

Destination için INVENTORY/BELT allowlist yoktur. `IsEmptyItemGrid` aktif build'de SWITCHBOT ve ADDITIONAL_EQUIPMENT_1 windowlarını da valid olarak kabul eder.

Bu nedenle Safebox/Mall itemı normal MoveItem semantiğini atlayarak:
- SWITCHBOT slotuna,
- ADDITIONAL_EQUIPMENT_1 slotuna
yerleştirilebilir.

Checkin tarafı da explicit source-window allowlist kullanmaz. SWITCHBOT itemı `GetItem(TItemPos)` ile alınabilir; RemoveFromCharacter -> SetItem(SWITCHBOT,null) UnregisterItem yapar. Additional Equipment gerçek equipped item ise `IsEquipped()` guardı nedeniyle checkin'de reddedilir.

### BUG-SAFEBOX-005 — persisted invalid/overlapping Safebox row -> grid desync / OOB write on removal
- Statik durum: **doğrulandı / aktif build**
- Reachability: malformed/legacy/corrupt DB row
- Sınıf: persistence validation + memory safety

Load:
`CHARACTER::LoadSafebox/LoadMall`
→ yalnız `m_pkSafebox->IsValidPosition(pItems->pos)` ile top-left slotu kontrol eder
→ `CSafebox::Add(pos,item)`.

`CSafebox::Add`:
1. top-left `IsValidPosition`
2. item window/cell set + save
3. `m_pkGrid->Put(pos,1,item->GetSize())`
4. **Put sonucunu kontrol etmez**
5. `m_pkItems[pos]=item`.

`CGrid::Put` item yüksekliği bottom boundary'yi aşarsa veya grid hücresi overlap ise false döner. Buna rağmen item pointerı safebox array'ine bağlanır.

Etkiler:
- overlap row: grid itemı reserve etmez ama slot pointerı vardır; sonraki Remove başka itemın occupied grid hücrelerini temizleyebilir.
- duplicate top-left row: sonraki `m_pkItems[pos]` önceki runtime pointerı overwrite eder.
- bottom-boundary multi-size row: sonraki `CSafebox::Remove -> CGrid::Get(pos,1,size)` yalnız top-left'i bounds-check eder ve yüksekliğe göre ilerler; row+h için ikinci bounds check yoktur. Bu durumda grid buffer dışına write oluşabilir.

Normal checkin `IsEmpty(pos,size)` ile korunduğundan ana trigger persisted invalid state'tir.

### Safebox lifecycle closure
- Runtime item expiry/delete: ITEM_MANAGER::RemoveItem SAFEBOX/MALL windowunda ilgili CSafebox::Remove(cell) çağırır; sonra M2_DESTROY_ITEM DB destroy packetini üretir.
- Logout/character teardown: delayed item saves flush edilir, ardından CloseSafebox/CloseMall çalışır. CSafebox destructor runtime itemları SkipSave ile unload eder; normal DB item rowları korunur.
- ItemAward Safebox/Mall routing: personal Safebox mall-awardları, Mall non-mall awardları filtreler; bu iki domain arasında cross-routing görülmedi.
- Packet slot widths: CG Safebox/Mall source slots uint8_t, current UI/storage domainiyle uyumlu; yeni truncation adayı bulunmadı.

### Current active-build canonical set
- BUG-SAFEBOX-003 — crafted stack partial/zero merge item loss
- BUG-SAFEBOX-004 — semantic destination/source-window bypass
- BUG-SAFEBOX-005 — persisted malformed row grid desync / potential OOB write

Build-dependent dormant:
- BUG-SAFEBOX-001
- BUG-SAFEBOX-002


## Safebox / Mall — final static pass (2026-09-26)

### OBS-SAFEBOX-002 — DB load failure can leave personal Safebox request stuck
`ReqSafeboxLoad` request öncesi `m_bOpeningSafebox=true` yapar.

Flag şu normal yollarda temizlenir:
- wrong password -> `SafeboxWrongPassword -> CancelSafeboxLoad`
- DB response geldiğinde conflicting window tespit edilirse -> `CancelSafeboxLoad`
- başarılı açılıştan sonra kullanıcı `CloseSafebox` yaparsa -> false.

DB `RESULT_SAFEBOX_LOAD` ikinci item sorgusunda SQL result yoksa DB yalnız log + local request context cleanup yapıp GAME'e success/failure packet göndermeden return eder. GAME tarafında flag'i sıfırlayacak response oluşmaz.

Sonuç: geçici DB/query failure sonrası aynı character session'ında yeni safebox load isteği `m_bOpeningSafebox` nedeniyle sürekli reddedilebilir; relog/character teardown recovery gerektirebilir.

Sınıf: availability/reliability, security exploit değil.

### OBS-SAFEBOX-003 — Mall open trust boundary Safebox'tan daha gevşek
`do_mall_password`:
- password length
- existing Mall instance
- 10-second request throttle
kontrollerini yapar.

Fakat personal Safebox `ReqSafeboxLoad` yolundaki NPC/open-position distance ve pending-opening state modelini kullanmaz.
`CInputDB::MallLoad` da Exchange/Shop/Cube/open-window conflict kontrolü yapmadan `LoadMall` çağırır.

Mall item checkout yine `SafeboxCheckout(..., bMall=1)` üzerinden ve item-placement kontrolleriyle ilerler. Password DB tarafında doğrulanmaya devam eder.

Bu nedenle bunu doğrudan ownership exploit olarak değil, **server-side access-policy / remote Mall access observation** olarak sınıflandırıyoruz. Dungeon/warp/other-window gameplay policy runtime'da ayrıca denenebilir.

### OBS-SAFEBOX-004 — personal Safebox cross-session lock yok, global login invariantına bağımlı
Personal Safebox state/lock `m_bOpeningSafebox` ve `m_pkSafebox` ile CHARACTER-local tutulur.
DB tarafında account_id bazlı SAFEBOX rows için Guild Storage'daki gibi explicit storage-open lock bulunmaz.

Normal mimaride aynı account'ın iki aktif character session'ına izin verilmemesi beklenen üst seviye invarianttır. Bu invariant herhangi reconnect/multi-core edge'de kırılırsa iki game process aynı account SAFEBOX item state'ini bağımsız load edebilir.

Mevcut statik taramada normal login yolundan bu invariantı kıran trigger doğrulanmadığı için bug değil, architecture dependency olarak tutulur.

### Safebox/Mall STATIC COMPLETE
Kapatılan ana alanlar:
- UI + Python/C++ send bindings
- password/load/close
- SAFEBOX/MALL DB load
- checkin/checkout
- in-storage move/stack
- Special Inventory routing
- Switchbot / Additional Equipment trust boundary
- item save/flush
- malformed DB reconstruction
- item award domain routing
- expiry/delete
- disconnect/logout
- packet slot width
- money feature compile-state
- Mall access-policy differences.

Aktif build canonical bugs:
- BUG-SAFEBOX-003
- BUG-SAFEBOX-004
- BUG-SAFEBOX-005

Dormant feature bugs:
- BUG-SAFEBOX-001
- BUG-SAFEBOX-002

Observations:
- OBS-SAFEBOX-002..004


## Mailbox — canonical first pass (2026-09-26)

Active build: `ENABLE_MAILBOX` ON.

### BUG-MAIL-001 — negative Yang/Won in write packet can mint sender currency
- Statik durum: **doğrulandı**
- Reachability: modified/crafted client
- Sınıf: server-side signed input validation / economy integrity

`TPacketCGMailboxWrite` carries:
- `int iYang`
- `int iWon`

`CInputMain::MailboxWrite` forwards them directly to `CMailBox::Write`.

`Write` checks only:
- `TotalYang = iYang + MAILBOX_PRICE_YANG`
- `TotalYang > Owner->GetGold()`
- `iWon > Owner->GetCheque()`

There is no `iYang >= 0` / `iWon >= 0` guard.

Then:
- `PointChange(POINT_GOLD, -TotalYang)`
- `PointChange(POINT_CHEQUE, -iWon)`

Negative attachment values therefore turn the debit into a positive credit. DB `QUERY_MAILBOX_WRITE` performs no secondary amount validation and stores the packet as-is in `m_map_mailbox`.

`bIsItemExist` uses `iYang > 0 || iWon > 0`, so negative values are not even marked as attachments.

### BUG-MAIL-002 — write-confirm protocol is not an authorization boundary
- Statik durum: **doğrulandı**
- Reachability: modified/crafted client

Intended flow:
`MAILBOX_WRITE_CONFIRM -> DB CHECK_NAME -> CheckPlayerResult`
validates player existence and current mail count against `MAILBOX_MAX_MAIL`.

But `HEADER_CG_MAILBOX_WRITE` can be sent directly.
`CMailBox::Write` does not revalidate:
- recipient exists,
- recipient mailbox count < max,
- any prior successful confirm state.

DB `QUERY_MAILBOX_WRITE` simply:
`m_map_mailbox[p->szName].emplace_back(*p)`.

Consequences:
- mail can be written to nonexistent arbitrary names,
- target max-mail limit can be bypassed,
- DB mailbox map can be grown beyond intended 90-mail domain,
- BUG-MAIL-001 does not require a valid recipient.

### BUG-MAIL-003 — sender attachment/currency commits before mailbox DB acknowledgement
- Statik durum: **doğrulandı**
- Sınıf: cross-process atomicity / persistence loss

`CMailBox::Write` order:
1. optional source item `RemoveFromCharacter`
2. `ITEM_MANAGER::DestroyItem` -> DB item destroy packet
3. sender Yang/Won debit
4. `HEADER_GD_MAILBOX_WRITE`
5. immediate client `POST_WRITE_OK`.

No DB acknowledgement or rollback exists.

DB `QUERY_MAILBOX_WRITE` only mutates in-memory `m_map_mailbox`; it does not immediately persist SQL.

Game/DB peer failure or DB-process crash can therefore preserve sender-side debit/item destruction while the mail is absent.

### BUG-MAIL-004 — mailbox DB persistence is delayed RAM-only until periodic backup
- Statik durum: **doğrulandı**
- Default backup interval: 3600 seconds

New writes/deletes/gets modify only `m_map_mailbox`.
SQL persistence occurs in `MAILBOX_BACKUP()`.

A DB process crash before the next backup can lose mailbox mutations that the game already reported as successful.

This amplifies BUG-MAIL-003 and receiver-side get/delete consistency issues.

### BUG-MAIL-005 — index identity drift between GAME snapshot and DB vector can clear the wrong mail
- Statik durum: **doğrulandı**
- Sınıf: mutable-vector index used as persistent transaction identity

GAME `CMailBox` holds a snapshot `vecMailBox`.
All mutations sent to DB identify a mail only by:
`name + uint8 Index`.

DB holds its independent `m_map_mailbox[name]` vector.

`MAILBOX_BACKUP()`:
- erases deleted/expired entries,
- sorts the DB vector by SendTime descending.

New mail can also be appended while the recipient keeps an existing mailbox window open.

Therefore the same numeric index can stop referring to the same logical mail.

Example effect class:
- GAME grants attachment from local index N,
- sends MAILBOX_GET(name,N),
- DB index N now points to a different mail after erase/sort,
- DB clears different attachment,
- locally granted mail can remain claimable after reload while another mail is lost/cleared.

This supports both duplication and attachment-loss scenarios.

### BUG-MAIL-006 — high-value receive tax/cap arithmetic uses signed int and can overflow
- Statik durum: **doğrulandı**
- Reachability: can occur with otherwise positive/legitimate large Yang mail

`GetItem` computes:
`const int TotalYang = mail.AddData.iYang - mail.AddData.iYang * MAILBOX_TAX / 100;`

With 5% tax, `iYang * 5` can overflow signed 32-bit above roughly 429M Yang.

Cap precheck also uses:
`TotalYang + Owner->GetGold() >= GOLD_MAX`
in signed int arithmetic.

After that `Owner->GiveGold(TotalYang)` is void; its eventual PointChange can refuse the credit at GOLD_MAX, but `GetItem` still clears the mail attachment and sends MAILBOX_GET to DB.

Result: wrong tax amount and/or mail Yang loss on high-value claims.

### BUG-MAIL-007 — mailbox source TItemPos allows Switchbot / Additional Equipment rule bypass
- Statik durum: **doğrulandı**
- Reachability: modified client
- Cross-reference: BUG-ITEM trust-boundary family

`Write` accepts any `TItemPos::IsValidItemPosition()` and only rejects `pos.IsEquipPosition()`.

Active `IsValidItemPosition` includes:
- SWITCHBOT
- ADDITIONAL_EQUIPMENT_1

Additional Equipment is not included by `IsEquipPosition()`.
The code does not call normal `MoveItem` / `CanUnequipNow` semantic guards.

A source item is fetched directly and destroyed through `RemoveFromCharacter`, allowing storage/system-window semantics to be bypassed. Switchbot removal unregisters the slot; Additional Equipment can bypass normal unequip policy checks.

### Mailbox first-pass persistence observation
DB mailbox state has no immutable mail ID in the GAME<->DB mutation packets. `TMailBox` contains only recipient name + uint8 Index. This is the root cause behind index-drift sensitivity.


## Mailbox — final static pass (2026-09-26)

### BUG-MAIL-008 — boot loader early-return prevents persisted mailbox reload
- Statik durum: **doğrulandı / aktif build**
- Etki: DB restart sonrası mailbox durability failure

`CClientManager::InitializeTables()` startup sırasında `InitializeMailBoxTable()` çağırır.

Fonksiyonun başı:
`if (m_map_mailbox.empty()) return true;`

Fresh DB process'te `m_map_mailbox` default olarak boşdur. Bu nedenle SQL `SELECT ... FROM mailbox%s` satırına ulaşılmaz.

Sonuç:
- shutdown backup ile SQL'e yazılmış postalar restart sonrası RAM map'e geri yüklenmez,
- kullanıcı mailbox load'ları boş map görür,
- sonraki `MAILBOX_BACKUP()` boş map ile tabloyu TRUNCATE ederek persisted kayıtları kalıcı silebilir.

Bu koşul büyük olasılıkla ters yazılmıştır.

### BUG-MAIL-009 — MAILBOX_BACKUP TRUNCATE + individual INSERTs non-transactional
- Statik durum: **doğrulandı**
- Sınıf: destructive full-table rewrite / crash atomicity

Backup önce:
`TRUNCATE TABLE player.mailbox`
çalıştırır.

Daha sonra her mail için ayrı `DirectQuery(INSERT...)` yürütür.
Transaction / staging table / atomic rename yoktur.

Crash, SQL error veya process kill TRUNCATE sonrası herhangi bir noktada olursa persistent mailbox table boş veya kısmi kalabilir.

Ayrıca TRUNCATE hard-coded `player.mailbox` kullanırken INSERT `mailbox%s` + `GetTablePostfix()` kullanır. Non-empty table postfix konfigürasyonunda farklı tabloların truncate/insert edilmesi mümkündür.

### BUG-MAIL-010 — user-controlled mailbox strings are written to SQL without escaping
- Statik durum: **doğrulandı**
- Sınıf: SQL query corruption / injection surface

`MAILBOX_BACKUP` raw:
- recipient name,
- sender name,
- title,
- message

değerlerini:
`VALUES('%s','%s','%s','%s',...)`
ile query içine gömer.

`mysql_real_escape_string` / DB escape helper kullanılmaz.
Mailbox banword kontrolü SQL escaping değildir.

Normal bir apostrof bile INSERT syntax'ını bozabilir; crafted text SQL syntax manipulation surface oluşturur.
Bu durum BUG-MAIL-009'un full-table rewrite modeliyle birleştiğinde backup sırasında mail persistence kaybını büyütebilir.

### BUG-MAIL-011 — fixed packet strings are not server-NUL-terminated before strlen/%s
- Statik durum: **doğrulandı**
- Reachability: crafted client
- Sınıf: server memory-safety / OOB read

`TPacketCGMailboxWrite` fixed char arrays taşır:
- szName
- szTitle
- szMessage

Server input packet üzerinde terminator force etmez.

`CMailBox::Write` doğrudan:
- `sys_err("%s"...)`
- `strlen(szName/title/message)`
kullanır.

Non-NUL-terminated crafted array packet buffer sınırından öteye okunabilir; crash / undefined read riski vardır.

Benzer şekilde confirm-name DB path'inde `p->szName` `%s` ile SQL query'ye sokulur.

Official client `strcpy` ile normalde NUL üretir fakat server güvenlik sınırı client davranışına bağımlı olmamalıdır.

### BUG-MAIL-012 — receiver grants attachment before DB GET acknowledgement
- Statik durum: **doğrulandı**
- Sınıf: cross-process atomicity / duplication-loss window

`CMailBox::GetItem` sırası:
1. item Create + AutoGiveItem
2. GiveGold
3. GiveCheque
4. local snapshot attachment fields = 0
5. `HEADER_GD_MAILBOX_GET(name,index)`.

DB acknowledgement yoktur.

GAME tarafındaki grant persist olurken DB GET packet'i kaybolur / DB peer fail olursa DB map attachment'ı koruyabilir ve reopen sonrası tekrar claim edilebilir.
Tersi crash sıralarında local grant kaybolup DB attachment temizlenebilir.

BUG-MAIL-005 index drift ile birleşirse yanlış DB mailinin temizlenmesi daha da kolaylaşır.

### OBS-MAIL-001 — W_MAILBOX is set but omitted from CanWarp opened-window mask
`CHARACTER::SetMailBox` aktif mailbox için `W_MAILBOX` set eder.

`CHARACTER::CanWarp` son mailbox işleminden sonraki portal cooldown'u kontrol eder fakat opened-window bitmask içinde `W_MAILBOX` yoktur.

Cooldown geçince mailbox object açıkken warp mümkün olabilir.
Cross-core character teardown mailbox'ı kapatır; same-process warp/access policy runtime'da doğrulanmalıdır.

Sınıf: gameplay/access-policy observation, doğrudan duplication olarak sınıflandırılmadı.

### OBS-MAIL-002 — mailbox block result enums exist but enforcement absent
`POST_WRITE_TARGET_BLOCKED` ve `POST_WRITE_BLOCKED_ME` result enumları mevcut.
Current MailBox.cpp write/check-name flow Messenger/block relationship sorgulamıyor.

Block sisteminin mailbox'ı da kapsaması ürün beklentisiyse enforcement eksiktir; mevcut koddan tek başına intended policy kesinleştirilmediği için observation olarak tutulur.

### OBS-MAIL-003 — client write bindings also trust Python lengths
Official client `SendPostWrite` ve `SendPostWriteConfirm` fixed packet arrays için `std::strcpy` kullanır.
Normal UI length-limit uygularsa sorun görünmez; arbitrary Python caller uzun string ile client stack overwrite/self-crash yüzeyi oluşturabilir.

Server BUG-MAIL-011 bağımsız olarak geçerlidir.

### Mailbox STATIC COMPLETE
Kapatılan alanlar:
- open/load/create/close
- confirm/check-name
- send item/Yang/Won
- receive item/Yang/Won
- get-all/delete/delete-all/add-data
- DB runtime map
- index identity
- periodic persistence
- boot reload
- expiry filtering
- SQL serialization
- string trust
- item-source windows
- warp/logout lifecycle
- client write bindings.

Canonical active bugs:
BUG-MAIL-001 .. BUG-MAIL-012.

Observations:
OBS-MAIL-001 .. OBS-MAIL-003.

## Ticket System — recovered canonical bugs

### BUG-TICKET-001 — PAGE_REPLY does not enforce ticket ownership
- Statik durum: **doğrulandı**
- Sınıf: authorization / information disclosure

`CTicketSystem::Open(PAGE_REPLY)` has the owner/deny check commented out and only verifies `GetExistID(ticked_id)`.
A modified client that knows/guesses another valid ticket ID can request its reply log through `SendTicketLogs(LOGS_FROM_REPLY,...)`.
`CTicketSystem::Reply` itself does call `GetOwner`; this bug is specifically the reply-view/open path.

### BUG-TICKET-002 — ticket text is interpolated into SQL without escaping
- Statik durum: **doğrulandı**
- Sınıf: SQL injection / query corruption

Create inserts title/content, Reply inserts reply text, staff ban inserts reason, and multiple ticket-ID/name queries use raw `%s`.
`IsDenied` is a blacklist and explicitly does not block single quote/backslash; it is not SQL escaping.
Server must escape/bind SQL independently of UI limits.

### BUG-TICKET-003 — ticket ID collision loop never refreshes collision query
- Statik durum: **doğrulandı**
- Sınıf: infinite loop / resource growth

Create generates strID, executes one SELECT, then:
`while (dwExtract->Get()->uiNumRows > 0)` reseeds and appends more random chars.
The query result is never refreshed and strID is never reset.
If the first generated ID collides, condition remains true forever while the string grows.

### BUG-TICKET-004 — admin sort accepts invalid mode and may query uninitialized buffer
- Statik durum: **doğrulandı**
- Sınıf: undefined behavior / DB query correctness

`Open(PAGE_SORT_ADMIN)` accepts any `mode > 0`.
`SendTicketLogs(LOGS_ADMIN,...,iSortMode)` initializes `szQuery` only for modes 1..4 and then always calls DirectQuery(szQuery).
Additionally SQL `LIMIT offset,count` receives `iEndIdx` as count; for page >1 that value is cumulative rather than MAX_LOGS_PER_PAGE.

### BUG-TICKET-005 — client ticket Request boundary check allows id == size
- Statik durum: **doğrulandı**
- Sınıf: client OOB read / crash

Both `CPythonTicketLogs::Request` and Reply variant use:
`if (m_vecData.size() < id)`.
For `id == size()` the guard is false and `m_vecData[id]` is out of bounds.

## Dungeon Info — recovered canonical bugs

### BUG-DUNGEON-001 — server Warp/Ranking index is unchecked
- Statik durum: **doğrulandı**
- Sınıf: server OOB / modified-client crash surface

Both `Warp` and `Ranking` access `s_vecDungeonProto[byIndex]` before validating `byIndex < size()`.
CG packet exposes uint8 index directly.

### BUG-DUNGEON-002 — client dungeon array has 255 slots but uint8 index can be 255
- Statik durum: **doğrulandı**
- Sınıf: client OOB

`m_vecDungeonInfoDataMap[255]` valid indices are 0..254.
`AddDungeon(uint8_t byIndex,...)` and many getters index it directly; 255 is representable by the network/Python boundary.

### BUG-DUNGEON-003 — CPythonDungeonInfo::Clear clears only first dungeon vector
- Statik durum: **doğrulandı**
- Sınıf: stale state / reload corruption

`m_vecDungeonInfoDataMap->clear()` is equivalent to clearing element 0 only.
Slots 1..254 retain old packet data while count/load flags reset.

### BUG-DUNGEON-004 — Warp couples level-limit count to entry-position vector
- Statik durum: **doğrulandı**
- Sınıf: server OOB / malformed-config crash

Warp loops `iPos < vecLevelLimit.size()` and indexes `vecEntryPosition[iPos]`.
No invariant check guarantees both config vectors have equal sizes.

### BUG-DUNGEON-005 — variable config item vectors copied into fixed packet arrays without cap
- Statik durum: **doğrulandı**
- Sınıf: stack/packet memory overwrite from malformed config

`SendInfo` loops full `vecRequiredItem` and `vecBossDropItem` and writes fixed `sRequiredItem[]` / `sBossDropItem[]` arrays with no size cap.

### BUG-DUNGEON-006 — bonus bounds check is off by one
- Statik durum: **doğrulandı**
- Sınıf: packet stack OOB

The loop breaks only when `iAffect > POINT_MAX_NUM`.
Index `POINT_MAX_NUM` is already outside arrays sized `[POINT_MAX_NUM]`; guard must stop before equality.

## Battle Pass — recovered canonical bugs

### BUG-BPASS-001 — mission update packet sends uninitialized bMissionType
- Statik durum: **doğrulandı**
- Sınıf: protocol correctness / uninitialized-data leak

`TPacketGCExtBattlePassMissionUpdate` contains `bMissionType`.
Character update/set code creates non-zero-initialized packet and assigns header/passType/missionIndex/newProgress but not missionType.
Client reads missionType and uses it in `HaveMission(...)`/UI selection.

### BUG-BPASS-002 — SetExtBattlePassMissionProgress can re-award an already completed mission
- Statik durum: **doğrulandı; caller reachability audit açık**
- Sınıf: reward duplication

Existing matched mission is forcibly changed to `bCompleted = 0`, then value is overwritten.
If new value is at/above threshold, code marks completed and calls `BattlePassRewardMission` again.
Any legitimate/replayable caller that sets a completed mission can duplicate mission reward.

### BUG-BPASS-003 — BattlePassRequestOpen uses dangling season_name pointer
- Statik durum: **doğrulandı**
- Sınıf: use-after-lifetime / undefined behavior

Inside each pass block:
a local `std::string BattlePassName` is created, `season_name = BattlePassName.c_str()`, then the string is destroyed at block end.
The pointer is used afterward to fill the packet.

### BUG-BPASS-004 — unbounded strcpy into season-name packet
- Statik durum: **doğrulandı**
- Sınıf: stack overwrite / config-trust

`strcpy(packet.szSeasonName, season_name)` has no destination-size enforcement.
Battle-pass name comes from config loader.

### BUG-BPASS-005 — final reward path dereferences MYSQL_ROW without zero-row check
- Statik durum: **doğrulandı**
- Sınıf: server crash on inconsistent persistence state

After SELECT from `player.battlepass_playerindex`, code checks only SQL errno.
It calls `mysql_fetch_row` then immediately reads `row[0]`.
Missing registration row can therefore null-dereference.

### BUG-BPASS-006 — Event Manager cache arrays are not initialized
- Statik durum: **doğrulandı**
- Sınıf: uninitialized state / season selection

`CBattlePassManager` constructor initializes scalar active IDs/times but not:
- `m_dwActiveBattlePassID[3]`
- `m_dwBattlePassStartTime[3]`
- `m_dwBattlePassEndTime[3]`

The manager is an automatic object in main, and `InitializeBattlePass()` calls `CheckBattlePassTimes()`, which reads these arrays under ENABLE_EVENT_MANAGER.

### BUG-BPASS-007 — Event Manager stores boolean state as battle-pass ID
- Statik durum: **doğrulandı for configured IDs != 1**
- Sınıf: season lifecycle / wrong identity

`BattlePassData(const TEventTable*, uint8_t bType, bool bState)` calls:
`SetBattlePassID(bState, bType)`.
The setter stores that uint32 directly as active ID, so start state becomes ID 1 and stop becomes 0.
A configured season whose real battle-pass ID is not 1 cannot be represented through this path.

## Battle Pass — persistence/lifecycle completion

### BUG-BPASS-008 — mission reward durability is split from mission progress durability
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / repeat reward

Mission completion immediately calls `BattlePassRewardMission`, which grants reward items using `AutoGiveItem`.
`CItem::AddToCharacter` calls `Save()`, placing the item into ITEM_MANAGER delayed-save state, and `ITEM_MANAGER::Update()` can persist it while the session is still running.

Battle Pass mission state is different: dirty `TPlayerExtBattlePassMission` records are sent to DB only from CHARACTER disconnect/logout and saved by DB as `REPLACE INTO battlepass_missions`.

Therefore this ordering exists:
1. mission becomes completed in GAME RAM;
2. reward item is granted;
3. item can become durable in DB;
4. battlepass_missions completion is still only RAM;
5. GAME crashes before clean logout;
6. next login reloads old mission progress and the mission can complete/reward again.

This needs fault injection for reproduction timing, but the persistence split itself is statically confirmed.

### BUG-BPASS-009 — final pass completion commits before final reward durability
- Statik durum: **doğrulandı**
- Sınıf: non-atomic claim / reward loss

`BattlePassRequestReward`:
1. verifies all missions;
2. SELECTs `battlepass_completed`;
3. executes synchronous UPDATE setting `battlepass_completed=1`;
4. only afterward calls `BattlePassReward`;
5. reward items use `AutoGiveItem` / delayed item save.

A crash or grant/save failure after step 3 and before reward items become durable leaves the DB claiming the final reward was consumed, so retry is rejected and the player can permanently lose the final reward.

### BUG-BPASS-010 — mission heap objects are never released
- Statik durum: **doğrulandı**
- Sınıf: memory leak

`LoadExtBattlePass`, normal Update and manual Set allocate `new TPlayerExtBattlePassMission` and store raw pointers in `m_listExtBattlePass`.
The CHARACTER initialization path only calls `m_listExtBattlePass.clear()`; disconnect iterates/saves the pointers but does not delete them.
No ownership cleanup for these allocations exists in the mapped source.
Character churn therefore leaks memory proportional to loaded/created mission rows.

### BUG-BPASS-011 — Battle Pass ranking cooldown timestamp is uninitialized
- Statik durum: **doğrulandı**
- Sınıf: uninitialized state / incorrect rate limiting

CHARACTER initialization sets `m_dwLastReciveExtBattlePassInfoTime = 0` but never initializes `m_dwLastExtBattlePassOpenRankingTime`.
The first ranking request reads it before the first setter call:
`if (get_dword_time() < GetLastReciveExtBattlePassOpenRanking())`.
An indeterminate value can incorrectly block the first ranking request or produce nonsensical remaining-time output.

### BUG-BPASS-006 impact refinement
Boot ordering confirms Event Manager table initialization occurs before `CBattlePassManager::InitializeBattlePass()`, but Event Manager initialization only builds event queues; it does not populate Battle Pass cache arrays.
`InitializeBattlePass()` then calls `CheckBattlePassTimes()`, so the uninitialized cache arrays remain a real first-boot read path under ENABLE_EVENT_MANAGER.

### BUG-BPASS-007 impact refinement
Current `Project_Game/share/locale/europe/battlepass/{normal,premium,event}.txt` all use `BattlePassID 1`.
Therefore bool `bState` accidentally equals the current ID while active.
This masks the defect today; any future season configured with ID 2+ will still be represented as active ID 1.

## Achievement System — initial canonical bugs

### BUG-ACH-001 — completion reward is durable before achievement completion state
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / repeat reward

`FinishAchievement` mutates only the in-memory player achievement map, sends client packets, then immediately calls `RewardPlayer`.
Rewards can be items (`AutoGiveItem`), gold, titles or achievement points.

The complete achievement/progress/points/title map is sent to DB only from `CAchievementSystem::OnLogout`.
DB then stores it in `CAchievementCache` for later flush.

Item rewards can independently become durable through normal ITEM_MANAGER delayed saves while the achievement completion remains only GAME RAM.
A hard GAME crash before clean logout can therefore reload the old unfinished state and allow the same achievement to finish/reward again.

### BUG-ACH-002 — achievement cache flush is non-transactional destructive rebuild
- Statik durum: **doğrulandı**
- Sınıf: persistence atomicity / data loss

`CAchievementCache::OnFlush` performs:
1. DELETE all `achievement_tasks` rows for pid and DELETE all `achievements` rows for pid;
2. REPLACE `achievement_data`;
3. INSERT every achievement row;
4. INSERT every unfinished task row.

These are separate DirectQuery calls with no transaction.
DB/process failure after the DELETE and before complete reconstruction can permanently leave a player with missing or partially rebuilt achievement/task state.

### BUG-ACH-003 — stale task ID can crash GetAchievementProgress for max_value achievements
- Statik durum: **doğrulandı**
- Sınıf: config-evolution / server crash

DB load preserves stored task IDs.
The login merge adds missing current tasks but does not remove obsolete task IDs from an existing achievement.

In `GetAchievementProgress`, when the current achievement has `max_value > 0`, code does:
`cTask = target_achievement->tasks.find(task.first)`
and immediately reads `cTask->second.type` without verifying `cTask != end()`.

If an XML update removes/renumbers a task while the DB still contains that old task ID, progress evaluation can dereference end() and crash the GAME core.
