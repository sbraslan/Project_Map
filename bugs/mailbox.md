# mailbox — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

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
