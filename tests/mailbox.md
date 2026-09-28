# mailbox — Runtime Tests

> Recovered from legacy `10_UNKNOWN_AREAS.md`; normal continuation reads this only when this subsystem is active.

## Mailbox — first pass sonrası açık alanlar

### Runtime kritik
- MAIL-T01: negative iYang packet; sender gold delta.
- MAIL-T02: negative iWon packet; sender cheque delta.
- MAIL-T03: direct WRITE without prior CHECK_NAME.
- MAIL-T04: > MAILBOX_MAX_MAIL direct write flood.
- MAIL-T05: keep mailbox open across DB backup sort/erase, then claim by old index.
- MAIL-T06: large Yang attachment around 430M+ tax arithmetic.
- MAIL-T07: source SWITCHBOT / ADDITIONAL_EQUIPMENT_1 send.
- MAIL-T08: DB process crash before periodic backup after successful send.

### Kalan statik
- backup TRUNCATE/INSERT transaction safety
- SQL string escaping for title/message/from/name
- packet fixed-char NUL termination
- expiry/delete/confirm index behavior
- boot loader
- messenger/block enforcement
- warp/logout/system-close lifecycle.

## Mailbox — static completion sonrası runtime test matrisi

- MAIL-T01 negative iYang -> sender gold delta.
- MAIL-T02 negative iWon -> sender cheque delta.
- MAIL-T03 direct WRITE without confirm / nonexistent target.
- MAIL-T04 mailbox >90 flood and client load behavior.
- MAIL-T05 open snapshot + incoming mail + periodic backup sort -> old index GET.
- MAIL-T06 local delete/expired erase + backup -> index shift.
- MAIL-T07 430M+ / 1B+ Yang receive tax/cap arithmetic.
- MAIL-T08 source SWITCHBOT.
- MAIL-T09 source ADDITIONAL_EQUIPMENT_1 / irremovable item.
- MAIL-T10 GAME write then DB process crash before backup.
- MAIL-T11 DB restart with pre-populated mailbox SQL table; verify InitializeMailBoxTable early return.
- MAIL-T12 kill DB during TRUNCATE->INSERT backup.
- MAIL-T13 apostrophe/quote in title/message during backup.
- MAIL-T14 crafted non-NUL packet string under ASan.
- MAIL-T15 receiver grant with DB GET fault injection.
- MAIL-T16 same-process warp while mailbox remains open after cooldown.
- MAIL-T17 block-list policy verification if mailbox blocking is intended.

Statik Mailbox keşfi kapalı; yalnız test sonucu yeni edge çıkarsa tekrar açılmalı.


- MAIL-T18 client fixed-string boundary: isolated/debug client only; call mailbox write/confirm bindings with boundary and overlength Python strings and observe client-side packet-buffer behavior under ASan/debug. Validates OBS-MAIL-003 only; server-side BUG-MAIL-011 remains independently covered by MAIL-T14.

## Mailbox readiness consolidation — 2026-09-28
- No Mailbox runtime test was executed.
- MAIL-T01/MAIL-T02 -> BUG-MAIL-001.
- MAIL-T03/MAIL-T04 -> BUG-MAIL-002.
- MAIL-T10 -> BUG-MAIL-003 and BUG-MAIL-004 across sender-commit / delayed-DB-persistence boundaries.
- MAIL-T05/MAIL-T06 -> BUG-MAIL-005.
- MAIL-T07 -> BUG-MAIL-006.
- MAIL-T08/MAIL-T09 -> BUG-MAIL-007.
- MAIL-T11 -> BUG-MAIL-008.
- MAIL-T12 -> BUG-MAIL-009.
- MAIL-T13 -> BUG-MAIL-010.
- MAIL-T14 -> BUG-MAIL-011.
- MAIL-T15 -> BUG-MAIL-012.
- MAIL-T16 -> OBS-MAIL-001.
- MAIL-T17 -> OBS-MAIL-002.
- MAIL-T18 added for OBS-MAIL-003 without promoting the observation.
- Primary legitimate/ordinary-flow candidates: MAIL-T07 (large legitimate Yang receive arithmetic) and MAIL-T11 (restart reload, disposable environment).
- Modified-client/adversarial: MAIL-T01..T04, MAIL-T08, MAIL-T09, MAIL-T14.
- Crash/persistence/fault-injection: MAIL-T10, MAIL-T12, MAIL-T15.
- Overall first live runtime gate remains DUNGEON-T10.
