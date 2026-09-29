# TICKET-T07 — Runtime Gate Handoff

**Prepared:** 2026-09-29  
**Execution status:** LOCKED / NOT RUN  
**Canonical bug:** `BUG-TICKET-007`  
**Canonical test:** `TICKET-T07`  
**Order:** First Ticket normal-path test after the Dungeon Info normal-path cluster.

> Handoff only. It does not authorize runtime execution.

## Static basis

Current server:
- `MAX_LOGS_GENERAL = 40`;
- normal-user query uses `LIMIT 40`;
- GC payload contains `logs[40]`.

Current C++ client:
- `TICKET_MAX_LOGS_GENERAL = 10`;
- `CPythonTicketLogs::AddLogDetails` copies only indices 0..9 from the 40-row packet.

Current Python UI:
- `TICKET_LOGS_PER_PAGE = 20`;
- `TICKET_MAX_PAGE_LOGS = 10`;
- page 1 tries to display positions 0..19;
- later pages address logical positions 20..199;
- non-admin page changes do not send a new server page request.

Therefore the normal user contract is inconsistent:
server sends up to 40 -> C++ retains 10 -> UI expects 20 rows/page across 10 pages.

## Boundary with BUG-TICKET-005

Fresh preflight closed an important ambiguity:
- `AppendLogs()` calls `ticket.GetLogByID(row)` only for `row in xrange(TICKET_MAX_LOGS_GENERAL)`, i.e. 0..9;
- page rendering for 10..19 reads the Python cache dictionaries with `.get()`;
- therefore ordinary TICKET-T07 does **not** directly call C++ `Request(10)` and does not normally trigger BUG-TICKET-005.

BUG-TICKET-005 remains an isolated direct/client-boundary test under TICKET-T05.

## Preconditions

1. Normal non-staff account.
2. Running build should correspond to mapped source or drift must be recorded.
3. Ticket system enabled and DB tables operational.
4. Account/character has enough historical tickets to make loss visible; >=25 is preferred.
5. Do not alter client constants or server LIMIT before evidence capture.

## Future live action

When runtime is explicitly unlocked and earlier gates are resolved:

1. Log in normally with a non-staff character.
2. Open the Ticket UI through ordinary UI flow.
3. Capture page 1.
4. Navigate to page 2 and at least one later page.
5. Compare visible ticket rows against known DB/server-side ticket count if safely observable.
6. Capture client/server logs.
7. Stop; do not patch in the same evidence step.

## Expected result

With more than 10 real tickets:
- only the first 10 delivered rows are retained by C++;
- page 1 positions 10..19 cannot represent rows 10..19 from the server packet;
- page 2+ cannot obtain additional normal-user rows from the server.

## Result classification

### REPRODUCED
Normal user has >10 tickets, UI opens, and rows beyond the retained first 10 are missing/blank/unavailable despite existing server-side records.

### NOT REPRODUCED
Mapped-equivalent build displays the full expected pagination correctly. Before retracting, compare deployment source/constants for an existing fix.

### INCONCLUSIVE
Use when there are <=10 tickets, DB data cannot be established, Ticket feature is unavailable, or deployment differs materially.

## Evidence template

- Test: `TICKET-T07`
- Bug: `BUG-TICKET-007`
- Result:
- Date/time:
- Client/server build:
- Known ticket count:
- Page 1 visible rows:
- Page 2 visible rows:
- Later-page behavior:
- Client errors:
- Server errors:
- Deployment drift:
- Evidence reference:
- Notes:

## Current state

**READY FOR FUTURE EXECUTION, BUT LOCKED.**
