# CURRENT — Canonical Active Checkpoint

**Active subsystem:** Ticket System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Last updated:** 2026-09-26

## Startup read set
For a normal "ilerleyelim" turn read only:
1. `STATE.json`
2. `CURRENT.md`
3. `systems/ticket.md`

Conditional:
- `bugs/ticket.md` only when validating/recording a bug.
- `tests/ticket.md` only for runtime/fault-injection work.
- source repos: search exact symbols first, then fetch exact files/ranges.

Do **not** reconstruct state from old chats. GitHub state is canonical.

## Just closed
Hunting System -> **STATIC COMPLETE**.
Verified Hunting bugs: `BUG-HUNT-001..005`.
All 62 configured Hunting reward VNUMs exist in the readable DumpProto item index; no missing configured VNUM was found.

## Current Ticket state
Already mapped:
- `game/src/ticket.cpp`
- `input_main.cpp::CInputMain::TicketSystem`
- CG/GC Ticket headers
- `UserInterface/PythonTicket.cpp`
- `root/uiticket.py`

Verified Ticket bugs: `BUG-TICKET-001..005`.

## Exact next work
1. Validate CG Ticket subpacket length/fixed-char handling; close TICKET-T06.
2. Validate Ticket packet-info size/sequence and client receive framing.
3. Audit PAGE/ACTION/admin authorization matrix.
4. Audit DB result null/empty handling and ticket/reply lifecycle.
5. Decide Ticket STATIC COMPLETE and update tests/checkpoint.

## End-of-turn write rule
When meaningful progress is made:
1. update `systems/ticket.md`;
2. update `bugs/ticket.md` / `tests/ticket.md` only if affected;
3. overwrite this file;
4. update `STATE.json`.

Keep this file short and overwrite-only.
