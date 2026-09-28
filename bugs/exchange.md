# exchange — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

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


## Registry identifier normalization note — 2026-09-28
- `BUG-EXCHANGE-001..003` are stable verified identifiers.
- `BUG-EXCHANGE-004` is currently reused by two distinct verified findings in this historical registry:
  1. currency overflow/transaction atomicity;
  2. missing final server-side distance recheck.
- `BUG-EXCHANGE-005` is the gold-recipient-cap TOCTOU specialization.
- Observation numbering around packet initialization versus AddGold/Cheque boolean logic is also historically duplicated/inconsistent.
- Detection-only policy: **do not renumber or rewrite historical bug IDs now**. Runtime documentation must disambiguate by descriptive finding name and canonical `EXC-Txx` test ID.
