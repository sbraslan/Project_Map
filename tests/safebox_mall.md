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


## Canonical Safebox / Mall runtime matrix — 2026-09-28

Legacy `SAFEBOX-Txx` and `SB-Txx` notes above are retained for history. From this point forward, **SFB-Txx** identifiers are canonical.

### SFB-T01 — partial occupied-stack merge
Isolated modified-client test of an occupied Safebox stack merge where only part of the source should move.
Covers BUG-SAFEBOX-003.

### SFB-T02 — full destination / zero transferable count
Isolated modified-client test with a full compatible destination stack so transferable count becomes zero; verify source remains owned and persisted.
Covers BUG-SAFEBOX-003.

### SFB-T03 — checkout to SWITCHBOT
Isolated modified-client checkout to a SWITCHBOT destination and verify server-side semantic rejection.
Covers BUG-SAFEBOX-004.

### SFB-T04 — checkout to Additional Equipment
Isolated modified-client checkout to ADDITIONAL_EQUIPMENT_1 and verify server-side semantic rejection.
Covers BUG-SAFEBOX-004.

### SFB-T05 — SWITCHBOT source checkin
Isolated modified-client checkin from SWITCHBOT and verify the storage path cannot bypass normal source semantics.
Covers BUG-SAFEBOX-004.

### SFB-T06 — malformed bottom-boundary persisted item
Disposable DB row with a multi-size item whose persisted Safebox top-left is valid but whose footprint crosses the grid boundary. Open/remove under ASan/debug.
Covers BUG-SAFEBOX-005.

### SFB-T07 — duplicate/overlapping persisted positions
Disposable DB with duplicate/overlapping Safebox item positions; open, inspect grid/item ownership, then close/reload.
Covers BUG-SAFEBOX-005.

### SFB-T08 — DB load failure opening-flag recovery
Fault-inject the Safebox item-query/load failure path and verify a later request in the same character session is not permanently blocked.
Validates OBS-SAFEBOX-002 only.

### SFB-T09 — Mall access-policy observation
Controlled normal account/password flow from remote or conflicting gameplay/window states; compare Mall opening policy with personal Safebox policy.
Validates OBS-SAFEBOX-003 only.

### SFB-T10 — same-account concurrent-session architecture dependency
Only if an isolated login-layer test can intentionally permit duplicate active account sessions, attempt concurrent personal Safebox access from separate cores/sessions.
Validates OBS-SAFEBOX-004 only; no normal-login exploit is assumed.

### SFB-T11 — dormant Mall-close gold overwrite
Conditional only if ENABLE_SAFEBOX_MONEY is re-enabled. Open/close Mall with known persisted personal Safebox gold and verify it is not overwritten.
Covers dormant BUG-SAFEBOX-001.

### SFB-T12 — dormant Safebox withdraw overflow/rollback
Conditional only if ENABLE_SAFEBOX_MONEY is re-enabled. Use controlled near-cap player gold and a large Safebox withdrawal; verify debit/credit is overflow-safe and atomic.
Covers dormant BUG-SAFEBOX-002.

### Readiness consolidation — 2026-09-28
- No Safebox/Mall runtime test was executed.
- Active-build verified bugs:
  - SFB-T01/SFB-T02 -> BUG-SAFEBOX-003
  - SFB-T03/SFB-T04/SFB-T05 -> BUG-SAFEBOX-004
  - SFB-T06/SFB-T07 -> BUG-SAFEBOX-005
- Observation-only:
  - SFB-T08 -> OBS-SAFEBOX-002
  - SFB-T09 -> OBS-SAFEBOX-003
  - SFB-T10 -> OBS-SAFEBOX-004
- Dormant feature coverage:
  - SFB-T11 -> BUG-SAFEBOX-001
  - SFB-T12 -> BUG-SAFEBOX-002
- Historical OBS-SAFEBOX-001 is superseded by the stronger canonical BUG-SAFEBOX-005 evidence and is not treated as a separate current observation.
- Current active build has ENABLE_SAFEBOX_MONEY off, so SFB-T11/SFB-T12 must remain disabled unless the feature is explicitly enabled in an isolated build.
- There is no clean normal-player verified-bug candidate in the active Safebox set; the closest ordinary-flow policy observation is SFB-T09.
- Overall first live runtime gate remains DUNGEON-T10.
