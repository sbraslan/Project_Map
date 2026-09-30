# x64 Migration Audit — Phase 1

**Goal:** prepare Client + Server for a clean 64-bit migration without changing gameplay behavior.

**Source policy:** source repositories remain read-only during this audit. Only Project_Map tracks migration state.

## Confirmed architecture

### Client
- All 16 Visual Studio projects currently define only `Win32`.
- UserInterface explicitly targets `MachineX86`.
- No x64 build configuration exists yet.
- Renderer remains D3D9; this is not changed during the x64 phase.

### Server
- `game` and `db` compile with `-m32`.
- Core static libraries also compile with `-m32`:
  - libthecore
  - libgame
  - libsql
  - libpoly
  - libachievement
- Lua 5.0 config also carries `-m32`.

Therefore the server migration must move the complete build graph to x64, not only game/db.

## ABI / wire-format blockers

### Server shared tables
`common/tables.h` is under packed layout and currently contains:
- 57 occurrences of `long` / `long long` in the audited file.
- 13 serialized/state `time_t` fields plus one local `size_t` loop usage.

Examples of ABI-sensitive fields:
- coordinates / map indices;
- socket arrays;
- seal dates;
- alignment;
- guild/war values;
- affect values/durations;
- event timestamps;
- Growth Pet timestamps;
- mailbox timestamps.

On FreeBSD x64 (LP64), `long` becomes 64-bit. These fields cannot be allowed to change wire/persistence width accidentally.

### Server game packets
`game/src/packet.h` contains:
- 99 `long` / `long long` occurrences in the audited packed packet region;
- 3 `time_t` packet fields.

This is the highest-priority x64 blocker because packet size drift changes stream framing.

### Client packet mirror
`UserInterface/Packet.h` contains:
- 74 `long` / `long long` occurrences;
- 4 `time_t` fields.

Windows x64 keeps `long` at 32-bit (LLP64), so many client packet fields keep their present width while FreeBSD server `long` would expand. If server x64 is compiled without normalization, client/server packet layouts can diverge immediately.

## Required classification

Every ABI-sensitive field will be classified into one of three groups.

### A — MUST NORMALIZE BEFORE x64
Anything that crosses a binary boundary:
- client <-> game packets;
- game <-> DB packets;
- game <-> P2P packets;
- persisted raw structures;
- raw `memcpy` / buffer-encoded structs;
- binary file formats.

Replace architecture-dependent width with explicit-width types:
- `int32_t` / `uint32_t`
- `int64_t` / `uint64_t`

For timestamps on wire/persistence, select and document an explicit width instead of raw `time_t`.

### B — REVIEW
- IDs / coordinates / timestamps used internally and later serialized;
- pointer-to-integer casts;
- `sizeof` assumptions;
- format strings;
- arithmetic that depends on 32-bit overflow;
- indexing/count types shared with packets.

### C — MAY REMAIN
- local/internal `long` values that never cross an ABI boundary and do not depend on 32-bit overflow semantics.

Do not mechanically replace all `long` occurrences.

## Client x64 build blockers

All 16 client projects are currently Win32-only:
- CWebBrowser
- EffectLib
- EterBase
- EterGrnLib
- EterImageLib
- EterLib
- EterLocale
- EterPack
- EterPythonLib
- GameLib
- MilesLib
- PRTerrainLib
- ScriptLib
- SpeedTreeLib
- SphereLib
- UserInterface

An x64 configuration must be added alongside Win32 first; Win32 is not removed during migration.

Known external/runtime dependency families that require x64 verification:
- Python 2 runtime/libs;
- Granny;
- CEF/WebBrowser stack;
- Miles audio;
- SpeedTree;
- DevIL/image stack;
- cryptography / OpenSSL-related libraries;
- any protect/anti-cheat/static external library.

The ClientSrc repository references external library directories, but those external binary libraries are not part of the audited ClientSrc source tree. Their architecture must therefore be verified separately before a successful x64 link is possible.

## Server x64 build blockers

Before removing `-m32`:
1. normalize Group A packet/table fields;
2. rebuild all project static libraries as x64;
3. verify external static libraries are x64;
4. verify MySQL/OpenSSL/system libraries match the x64 target;
5. add structure-size guards;
6. compare x86 and x64 protocol sizes before any live connection.

## Mandatory safety guards

For shared packets/tables, introduce compile-time ABI checks during implementation:

```cpp
static_assert(sizeof(TPacketExample) == EXPECTED_SIZE);
```

Where possible, also assert critical field widths:

```cpp
static_assert(sizeof(decltype(TPacketExample::x)) == 4);
```

Packet-size baselines must be captured from the current working x86 build before changing types.

## Migration execution order

### M64-01 — ABI inventory
Status: ACTIVE
- enumerate packed/shared structs containing `long`, `time_t`, pointer-sized or ABI-sensitive fields;
- map each to its producer/consumer;
- classify A/B/C.

### M64-02 — Packet-size baseline
Status: PENDING
- record current x86 `sizeof` values for shared client/server and game/DB/P2P structures;
- add a small compile-time/runtime dump harness later when source-write phase is authorized.

### M64-03 — Fixed-width protocol normalization
Status: PENDING
- convert Group A fields only;
- keep protocol values semantically unchanged;
- add `static_assert` guards.

### M64-04 — Server x64 build profile
Status: PENDING
- create x64 build path without deleting existing x86 path;
- rebuild complete library graph;
- resolve compiler/linker issues.

### M64-05 — Server x64 smoke parity
Status: PENDING
- boot game/db;
- login;
- player load/save;
- item load/save;
- inventory/equipment;
- shop/safebox;
- guild/P2P;
- warp/map state.

### M64-06 — Client x64 build profile
Status: PENDING
- add x64 configurations to all 16 Visual Studio projects;
- resolve external x64 dependencies;
- fix pointer-width assumptions.

### M64-07 — Client x64 smoke parity
Status: PENDING
- boot/login/character select;
- world load;
- inventory/equipment;
- combat/effects;
- shops/UI;
- warp/reconnect;
- long-session memory stability.

### M64-08 — x86 retirement decision
Status: PENDING
Only after x64 parity and stability are proven.

## Current first action

Continue M64-01 by producing a producer/consumer matrix for:
1. `common/tables.h`
2. `game/src/packet.h`
3. `UserInterface/Packet.h`

Start with structures containing `long` or `time_t` that are directly sent using raw `sizeof(struct)` semantics.

## Non-goals during this phase
- no DX11 work;
- no visual changes;
- no gameplay-system additions;
- no broad bug-fix campaign;
- no speculative performance rewrites.

The runtime bug-validation queue remains preserved but paused behind infrastructure modernization.
