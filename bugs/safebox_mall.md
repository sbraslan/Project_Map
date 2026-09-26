# safebox mall — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

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
