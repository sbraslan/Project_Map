# Metin2 Project — Recovery Pointer / Win32 Client Baseline

**Created:** 2026-10-22  
**Purpose:** durable recovery pointer before continuing M64 ABI/x64 migration work.  
**Canonical-state rule:** this file is a recovery/checkpoint note only. `MAP_STATE.json` remains the sole canonical authority for lifecycle/cursor state.

## 1. Project_Map pointer

Project_Map HEAD immediately before this pointer:
- `1bb454e1293e041aff0bf7d1604d2ac1c4c948ce`
- commit: `docs: add role-based agent workflow for Metin2 mapping`

Canonical mapping state read before creating this pointer:
- mode: `MODERNIZATION_X64_AUDIT`
- execution lock: `LOCKED`
- closed subsystems: `47`
- open subsystems: `0`
- active cursor: `null`

The earlier static gameplay/system mapping is therefore complete on the pinned source snapshot. Runtime/fault-injection testing remains locked.

## 2. Pinned source snapshots

These are the source SHAs recorded by `MAP_STATE.json`:

- Project_ClientSrc: `19c1e08141eb8ff2bf22fd8b93df3e4f3a516d81`
- Project_ServerSRC: `321712c849aa01d3635b9fe6a9cde79cc0adcb64`
- Project_Binary: `c87aea062f8c50e70a854d4e1dc0c75773960e27`
- Project_Game: `4f26f48e0b20b76825a5a6158c992b8c5670f44e`
- Project_DumpProto: `1066f4620af451c5f728260eb5f8a086672e6384`

Repository policy at checkpoint time:
- writable: `sbraslan/Project_Map`
- read-only: ClientSrc, ServerSRC, Binary, Game, DumpProto

## 3. Confirmed Client Win32 baseline

### User-verified local evidence

On 2026-10-22 the user confirmed that Visual Studio successfully builds the current client with no build error in the active Win32 configuration.

This is the current **working Win32/x86 client baseline** and should be preserved before any ABI/x64 source change.

Important evidence distinction:
- Visual Studio build success is **user-verified local evidence**.
- Local external-library binaries are not stored in the audited Project_ClientSrc GitHub source snapshot.
- This pointer records their working state; it does not copy the local binary files into Project_Map.

## 4. External-library blocker history and resolved state

During preparation of the reproducible Win32 baseline, the build was previously blocked by missing external libraries including:
- `SpeedTreeRT.lib`
- `cryptlib-Release.lib`
- `snappy.lib`

By the time of this checkpoint, Visual Studio successfully completes the client build, so these previously observed missing-library blockers are no longer active for the current local Win32 build environment.

### Crypto++ 8.2

Crypto++ 8.2 was built successfully:
- build result observed: `35 warnings / 0 errors`
- output: `cryptlib-Release.lib`
- recorded local size: `34,763,784 bytes`

Symbol verification observed:
- `AlgorithmProvider` -> found
- `OptimalDataAlignment` -> found

### Recorded local external-library inventory

The last captured local inventory before successful Visual Studio build was:

| Library | Recorded size |
|---|---:|
| `DevIL.lib` | 31,326 bytes |
| `ILU.lib` | 12,006 bytes |
| `mss32.lib` | 263,786 bytes |
| `granny2.11.8.0.lib` | 257,226 bytes |
| `SpeedTreeRT.lib` | 50,622 bytes |
| `CrashRpt1500.lib` | 6,840 bytes |
| `cryptlib-Release.lib` | 34,763,784 bytes |

`snappy.lib` was a prior linker blocker. Its explicit size was not captured in the final inventory above, but the later successful Visual Studio build is the current practical baseline evidence that the complete active link dependency set is usable.

## 5. Symbol-check interpretation

A diagnostic `dumpbin /symbols` pass produced:
- `mss32.lib -> AIL_startup`: OK
- `cryptlib-Release.lib -> AlgorithmProvider`: OK
- `cryptlib-Release.lib -> OptimalDataAlignment`: OK
- `DevIL.lib -> ilBindImage`: not found by that diagnostic
- `granny2.11.8.0.lib -> GrannySetLogCallback`: not found by that diagnostic
- `CrashRpt1500.lib -> crInstallA`: not found by that diagnostic

These three negative diagnostic matches are **not classified as active build failures** at this checkpoint because the actual Visual Studio client build subsequently succeeds. The old `/symbols` check should therefore not be used by itself as a blocker decision for these import/static libraries.

Source-side dependency proof in `Project_ClientSrc/UserInterface/UserInterface.cpp` includes:
- `#pragma comment(lib, "devil.lib")`
- Granny library selection including `granny2.11.8.0.lib`
- `#pragma comment(lib, "mss32.lib")`
- `#pragma comment(lib, "SpeedTreeRT.lib")`
- CrashRpt include/library under `ENABLE_CRASHRPT`
- runtime call `GrannySetLogCallback(&Callback)`
- runtime call `crInstall(&info)`

## 6. x64 migration position

The architecture plan remains:

1. preserve the now-working Win32 baseline;
2. capture x86 packet/struct size baselines;
3. normalize ABI-sensitive client/server wire fields;
4. prove no x86 packet size changed;
5. build the full ServerSRC graph for x64;
6. only after server ABI parity, proceed with the client x64 profile and external dependency audit.

Do **not** mix the ABI/x64 migration with DX11 work.

## 7. Immediate next task after recovery

Resume at the packet-size/ABI baseline gate.

First packet family:
- `TPacketCGMove`
- `TPacketCGSyncPositionElement`
- `TPacketGCMove`
- `TPacketGCPointChange`
- `TPacketGCWarp`
- `TPacketGCItemSet`
- `TPacketGCAffectElement`
- `TPacketGCTime`

The next operation should establish the current x86 sizes/field widths on both sides before changing any protocol type.

Relevant source surfaces:
- `Project_ServerSRC/game/src/packet.h`
- `Project_ServerSRC/common/tables.h`
- `Project_ClientSrc/UserInterface/Packet.h`

The existing `X64_MIGRATION_AUDIT.md` remains the migration evidence document. This pointer exists only so the work can be safely resumed from the current proven baseline without reconstructing the dependency/build history from chat.

## 8. Recovery rule

If later work breaks the client build:

1. return conceptually to this checkpoint;
2. verify the source snapshots above;
3. verify the working Win32 external-library set;
4. restore a successful Win32 build first;
5. only then re-apply ABI/x64 changes in small waves.

Do not treat a later x64 build problem as justification to alter gameplay behavior or start DX11 migration.
