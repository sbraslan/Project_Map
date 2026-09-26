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
