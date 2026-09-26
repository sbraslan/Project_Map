# safebox mall

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Safebox / Mall static audit başladı

**Tarih:** 2026-09-26

Shop static completion sonrasında personal Safebox / Item Mall subsystemine geçildi.

### Bu tur haritalanan
- official Safebox UI slot events
- Python binding / C++ send boundary
- SAFEBOX_CHECKIN / CHECKOUT / ITEM_MOVE
- MALL_CHECKOUT
- Safebox money deposit/withdraw
- CSafebox Add/Remove/MoveItem/Save/ChangeSize
- DB safebox load/save
- SAFEBOX/MALL account-ID item persistence

### Yeni doğrulanmış buglar
- BUG-SAFEBOX-001: Mall close, Mall gold=0 değerini generic Safebox Save ile persisted safebox.gold üzerine yazıyor.
- BUG-SAFEBOX-002: withdraw signed-int overflow + safebox debit-before-player-credit rollback eksikliği.
- BUG-SAFEBOX-003: crafted occupied-target stack move partial/zero transferde source itemı yanlış kaldırıyor.
- BUG-SAFEBOX-004 / BUG-ITEM-006: semantic source/destination window allowlist eksikliği.

### Official client ayrımı
Official UI safebox->safebox MoveItem packetini boş hedefe count=0 ile yollar. Occupied target stack packetini üretmediği için BUG-SAFEBOX-003 normal UI değil modified-client/server-validation sınıfıdır.

### Sıradaki
1. malformed/overlap DB load + grid reconstruction
2. SAFEBOX/MALL item expiration/removal
3. ItemAward placement/account-window isolation
4. packet width/slot boundaries
5. logout/close persistence ordering

## Checkpoint — Safebox / Mall second pass + active build classification

**Tarih:** 2026-09-26

### Build durumu
`ENABLE_SAFEBOX_IMPROVING` açık; `ENABLE_SAFEBOX_MONEY` kapalı.

Bu nedenle ilk turdaki money bugları mevcut binary/server build için runtime önceliğinden çıkarıldı:
- BUG-SAFEBOX-001 dormant
- BUG-SAFEBOX-002 dormant

### Bu tur kapatılan
- malformed/overlap DB load + CGrid semantics
- SAFEBOX/MALL item expiry/remove
- ItemAward Safebox-vs-Mall routing
- packet source-slot width
- logout/disconnect CloseSafebox/CloseMall ordering
- stack-loss persistence etkisi
- Switchbot / Additional Equipment semantic checkout reachability

### Yeni aktif bug
**BUG-SAFEBOX-005**:
Persisted invalid/overlap item rowunda CSafebox::Add CGrid::Put(false) sonucunu yok sayıyor. Runtime pointer grid reservation olmadan kuruluyor. Bottom-boundary multi-size state sonradan Remove edildiğinde CGrid::Get height bounds kontrolü yapmadan grid buffer dışına yazabilir.

### Safebox/Mall statik durum
Ana client -> game -> DB -> reload -> close/expiry akışı **static completion'a yakın**.

Kalan kısa statik tur:
1. safebox password/load-request lifecycle + pending-load state
2. account-level same-safebox multi-character/concurrent-open davranışı
3. channel/warp/open-window cleanup
4. ardından Safebox/Mall STATIC COMPLETE checkpoint.

## Checkpoint — Safebox / Mall STATIC COMPLETE

**Tarih:** 2026-09-26

Safebox/Mall final statik tur tamamlandı.

### Son kapanan alanlar
- password/pending-load lifecycle
- DB failure response behavior
- Mall request policy
- open-window / CanWarp integration
- disconnect close ordering
- account-level concurrency dependency

### Son sınıflandırma
Aktif build:
- BUG-SAFEBOX-003 — crafted partial/full stack merge source loss
- BUG-SAFEBOX-004 — semantic TItemPos storage bypass
- BUG-SAFEBOX-005 — malformed persisted row -> grid desync / possible OOB write

`ENABLE_SAFEBOX_MONEY` kapalı:
- BUG-SAFEBOX-001/002 dormant.

### Durum
**Safebox / Mall: STATIC COMPLETE**

Yeni Safebox statik taraması yalnız runtime test sonucu yeni caller/edge çıkarırsa açılacak.
Bir sonraki yeni transaction subsystemine geçilebilir.

## Related
- Bugs: `../bugs/safebox_mall.md`
- Runtime tests: `../tests/safebox_mall.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
