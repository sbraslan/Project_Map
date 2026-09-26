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

## Dungeon Info — recovered canonical bugs
