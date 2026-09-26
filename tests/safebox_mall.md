# safebox mall — Runtime Tests

> Recovered from legacy `10_UNKNOWN_AREAS.md`; normal continuation reads this only when this subsystem is active.

## Safebox / Mall — first pass sonrası açık alanlar

### Statik olarak çözülen
- checkin/checkout primary flow
- account-ID item persistence
- DB safebox/mall load
- money save/withdraw primary flow
- Mall close money overwrite
- safebox internal stack branch
- explicit TItemPos trust boundary

### Runtime testleri
- SAFEBOX-T01: Mall aç/kapat öncesi ve sonrası persisted safebox.gold.
- SAFEBOX-T02: player gold ~1.5B + withdraw >=0.7B cap/rollback.
- SAFEBOX-T03: crafted occupied-stack partial merge.
- SAFEBOX-T04: full destination stack + count=0 crafted move.
- SAFEBOX-T05: checkout -> SWITCHBOT / ADDITIONAL_EQUIPMENT.
- SAFEBOX-T06: unsupported source window -> checkin.

### Kalan kısa statik tur
1. malformed/overlapping DB row -> CSafebox::Add grid behavior
2. SAFEBOX/MALL item expiration/removal
3. ItemAward full/duplicate placement and isolation
4. packet slot/count narrowing
5. disconnect/logout/CloseSafebox/CloseMall persistence ordering

## Safebox / Mall — second pass sonrası kalanlar

### Aktif runtime testleri
- SAFEBOX-T03: crafted occupied-stack partial merge; source remainder DB sonucu.
- SAFEBOX-T04: full destination stack + count=0 crafted move.
- SAFEBOX-T05: checkout destination SWITCHBOT.
- SAFEBOX-T06: checkout destination ADDITIONAL_EQUIPMENT_1.
- SAFEBOX-T07: SWITCHBOT source -> safebox checkin.
- SAFEBOX-T08: malformed persisted multi-size item bottom-boundary; open + remove under ASan.
- SAFEBOX-T09: overlapping / duplicate persisted positions; open/close/reload consistency.

### Build-dependent, şimdilik çalıştırma
`ENABLE_SAFEBOX_MONEY` kapalı olduğu için:
- eski SAFEBOX-T01 Mall gold overwrite
- eski SAFEBOX-T02 gold withdraw overflow
mevcut build'de applicable değildir. Feature açılırsa tekrar aktive edilmeli.

### Kalan statik
1. password/load request pending-state lifecycle
2. aynı account safebox'ının iki karakter/core üzerinden concurrent open ihtimali
3. warp/channel-change ve open safebox cleanup
4. static completion checkpoint.

## Safebox / Mall — static completion checkpoint

Safebox/Mall statik keşfi kapatıldı.

### Runtime-only kalan
- SB-T01 crafted partial stack merge.
- SB-T02 full destination stack + count=0.
- SB-T03 checkout -> SWITCHBOT.
- SB-T04 checkout -> ADDITIONAL_EQUIPMENT_1.
- SB-T05 SWITCHBOT source -> checkin.
- SB-T06 malformed bottom-boundary persisted item under ASan.
- SB-T07 duplicate/overlap persisted positions.
- SB-T08 force DB safebox item-query failure -> opening flag recovery.
- SB-T09 Mall remote/open-during-other-window gameplay policy.
- SB-T10 same-account reconnect race only if global login layer permits duplicate active session.

Money tests yalnız ENABLE_SAFEBOX_MONEY yeniden açılırsa uygulanmalı.
