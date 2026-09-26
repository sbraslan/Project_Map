# mailbox

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Mailbox static audit başladı

**Tarih:** 2026-09-26

Safebox/Mall STATIC COMPLETE sonrasında Mailbox subsystemine geçildi.

### Haritalanan
- mailbox open/load snapshot
- write-confirm/check-name
- direct write
- item/Yang/Won sender commit
- receiver GetItem/GetAllItems
- DB in-memory mailbox map
- GET/DELETE/CONFIRM index protocol
- periodic MAILBOX_BACKUP
- client Python/C++ write binding

### İlk kritik bulgular
- BUG-MAIL-001: negative signed Yang/Won -> sender currency mint.
- BUG-MAIL-002: direct WRITE confirm/name/full-limit kontrolünü bypass ediyor.
- BUG-MAIL-003: sender item/currency DB ack öncesi commit.
- BUG-MAIL-004: mailbox writes RAM-only, SQL backup periyodik.
- BUG-MAIL-005: GAME snapshot index vs DB sorted/erased vector index drift.
- BUG-MAIL-006: large Yang receive tax/cap signed overflow.
- BUG-MAIL-007: SWITCHBOT / Additional Equipment source semantic bypass.

### Sıradaki Mailbox turu
1. MAILBOX_BACKUP SQL atomicity + escaping
2. boot/reload table reconstruction
3. string termination / packet trust
4. delete/expiry semantics
5. block/messenger policy
6. open/close/warp lifecycle
7. Mailbox static completion.

## Checkpoint — Mailbox STATIC COMPLETE

**Tarih:** 2026-09-26

Mailbox ikinci/final statik tur tamamlandı.

### Final yeni kritikler
- BUG-MAIL-008: boot loader `m_map_mailbox.empty()` iken erken return ediyor; persisted SQL mail reload edilmiyor.
- BUG-MAIL-009: full table TRUNCATE + per-mail INSERT transaction değil.
- BUG-MAIL-010: backup SQL string escaping yok.
- BUG-MAIL-011: packet fixed strings için server NUL termination yok.
- BUG-MAIL-012: receiver attachment grant DB GET ack öncesi commit.

### Policy gözlemleri
- W_MAILBOX set ediliyor fakat CanWarp opened-window maskesinde yok.
- mailbox block-result enumları var ama Messenger/block enforcement yok.
- client Python binding uzun stringleri strcpy ile packet array'lerine yazıyor.

### Durum
**Mailbox: STATIC COMPLETE**

Mailbox için sıradaki iş runtime/fault-injection test matrisidir.
Yeni subsystem'e geçmeye hazır.

## Related
- Bugs: `../bugs/mailbox.md`
- Runtime tests: `../tests/mailbox.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
