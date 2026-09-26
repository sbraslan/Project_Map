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
