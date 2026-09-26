# ticket — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-TICKET-001 — PAGE_REPLY does not enforce ticket ownership
- Statik durum: **doğrulandı**
- Sınıf: authorization / information disclosure

`CTicketSystem::Open(PAGE_REPLY)` has the owner/deny check commented out and only verifies `GetExistID(ticked_id)`.
A modified client that knows/guesses another valid ticket ID can request its reply log through `SendTicketLogs(LOGS_FROM_REPLY,...)`.
`CTicketSystem::Reply` itself does call `GetOwner`; this bug is specifically the reply-view/open path.

### BUG-TICKET-002 — ticket text is interpolated into SQL without escaping
- Statik durum: **doğrulandı**
- Sınıf: SQL injection / query corruption

Create inserts title/content, Reply inserts reply text, staff ban inserts reason, and multiple ticket-ID/name queries use raw `%s`.
`IsDenied` is a blacklist and explicitly does not block single quote/backslash; it is not SQL escaping.
Server must escape/bind SQL independently of UI limits.

### BUG-TICKET-003 — ticket ID collision loop never refreshes collision query
- Statik durum: **doğrulandı**
- Sınıf: infinite loop / resource growth

Create generates strID, executes one SELECT, then:
`while (dwExtract->Get()->uiNumRows > 0)` reseeds and appends more random chars.
The query result is never refreshed and strID is never reset.
If the first generated ID collides, condition remains true forever while the string grows.

### BUG-TICKET-004 — admin sort accepts invalid mode and may query uninitialized buffer
- Statik durum: **doğrulandı**
- Sınıf: undefined behavior / DB query correctness

`Open(PAGE_SORT_ADMIN)` accepts any `mode > 0`.
`SendTicketLogs(LOGS_ADMIN,...,iSortMode)` initializes `szQuery` only for modes 1..4 and then always calls DirectQuery(szQuery).
Additionally SQL `LIMIT offset,count` receives `iEndIdx` as count; for page >1 that value is cumulative rather than MAX_LOGS_PER_PAGE.

### BUG-TICKET-005 — client ticket Request boundary check allows id == size
- Statik durum: **doğrulandı**
- Sınıf: client OOB read / crash

Both `CPythonTicketLogs::Request` and Reply variant use:
`if (m_vecData.size() < id)`.
For `id == size()` the guard is false and `m_vecData[id]` is out of bounds.

### BUG-TICKET-006 — non-NUL fixed-char Ticket subpackets can cause server out-of-bounds reads
- Statik durum: **doğrulandı**
- Sınıf: packet parsing / memory safety / client trust

Ticket CG subpackets carry fixed char arrays such as:
- `ticked_id[11]`
- `title[33]`
- `content[513]`
- `reply[513]`
- `char_name[13]`
- `reason[33]`.

`CInputMain::TicketSystem` validates only that the expected fixed subpacket byte count is present. It does not verify that each character field contains a terminating NUL inside its own array.

The raw fields are then passed directly to `CTicketSystem::{Open,Create,Reply,Action}`, which call `strlen`, `strcmp`, build `std::string` values, and pass the fields to `%s` SQL/query formatting.

A modified client can therefore fill a fixed array completely with nonzero bytes and make server string functions continue reading beyond the field boundary into adjacent packet/stack memory. This is an out-of-bounds read and can produce a crash or corrupted query/input interpretation.

### BUG-TICKET-007 — normal-user ticket pagination contract is internally inconsistent
- Statik durum: **doğrulandı**
- Sınıf: client/server contract / pagination / data loss

The three layers disagree on the number of normal-user ticket rows:

Server:
- `MAX_LOGS_GENERAL = 40`
- `SendTicketLogs(LOGS_GENERAL)` queries up to 40 rows and sends `TSubPacketTicketLogsData.logs[40]`.

C++ client:
- `TICKET_MAX_LOGS_GENERAL = 10`
- `CPythonTicketLogs::AddLogDetails` copies only `p.logs[0..9]` into the client vector.

Python UI:
- `TICKET_LOGS_PER_PAGE = 20`
- `TICKET_MAX_PAGE_LOGS = 10`
- page 1 reads indices 0..19 and later pages request local indices up to 199.

There is no normal-user CG page request equivalent to the admin page-change packet. Therefore rows 10..39 already delivered by the server are discarded by the C++ cache, page 1 itself expects more rows than are cached, and pages 2..10 cannot be backed by server data.

This is a deterministic static pagination/data-loss defect, not merely a cosmetic page-count mismatch.

## Dungeon Info — recovered canonical bugs
