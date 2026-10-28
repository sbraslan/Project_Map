# 2026-10-28 — x64 + Python 3.15 Gameplay Stable Checkpoint

## Status

**STATE:** STABLE GAMEPLAY CHECKPOINT  
**Date:** 2026-10-28  
**Checkpoint tag:** `checkpoint/x64-py315-gameplay-stable-2026-10-28`

This checkpoint records the first confirmed gameplay-stable baseline after the server/client x64 migration and Python 3.15 runtime port.

## Source snapshot

| Repository | Branch | Commit |
|---|---|---|
| `sbraslan/Project_ClientSrc` | `migration/client-x64` | `2f788743b9818329da4d741ff6912faa0affed74` |
| `sbraslan/Project_ServerSRC` | `modernization/x64-wiretime-compat` | `1742dbdc073324bb7154128218954b9555d6551f` |
| `sbraslan/Project_Binary` | `main` | `5039c5d65141eef5fc2bf5c9f8895ebc96abcc42` |
| `sbraslan/Project_Game` | `main` | `24bfe96db414abd070f77f215bece36aaa52e1d0` |
| `sbraslan/Project_DumpProto` | `main` | `1066f4620af451c5f728260eb5f8a086672e6384` |

ClientSrc and Binary carry the same annotated checkpoint tag.

## Runtime binary

- Configuration: **Release x64**
- Executable: `metin2release.exe`
- SHA256: `F5AA00C4A9A499D61310BBF4435EBD9385CD7A7FA570CA5E26BFBE668638E60C`
- Size: **12,150,272 bytes**
- Build timestamp observed: **2026-10-28 00:02:05**
- Source/runtime executable hashes were verified equal before gameplay validation.

## Current execution stack

### Server / database host

| Component | Current value | Verification |
|---|---|---|
| OS | **FreeBSD 14.3 amd64** | Confirmed in migration/build work |
| Server architecture | **64-bit / amd64** | game + db compiled |
| Compiler | **GCC/G++ 14** | Confirmed |
| Database | **MariaDB 10.11** | Confirmed |
| Boost | **1.89** | Confirmed installed package |
| Crypto++ | **8.9** | Confirmed installed package |
| ccache | Installed | Exact version not pinned in checkpoint evidence |
| DevIL | Installed | Exact version not pinned in checkpoint evidence |
| Server locale config | `LOCALE = latin1` | Runtime config observed |
| DB connection | MariaDB running; prior socket issue resolved | Confirmed during runtime bring-up |

### Windows client build/runtime

| Component | Current value | Verification |
|---|---|---|
| Target | **Windows x64 / Release** | Confirmed |
| Toolchain | **Visual Studio 2022 / v143** | Confirmed |
| Python | **3.15.1 x64** | Confirmed runtime log |
| Granny | **2.11.10.0 x64** | Confirmed migration dependency |
| CEF | **CEF 90 x64** | Confirmed migration dependency |
| libjpeg | Static library | Confirmed dependency |
| LZ4 | Integrated | Version not independently pinned |
| Crypto++ / cryptlib | x64 client dependency | Exact client library version not independently pinned here |
| Granny runtime DLL | `granny2_x64.dll` | Runtime dependency |
| CEF import library | `libcef.lib` | Build dependency |
| Miles / MSS | `mss32` dependency present | Exact version not independently pinned |
| Renderer | Existing legacy renderer retained | DX11 migration remains future work |

## Python 3.15 root synchronization

The tested runtime `Project_Client/pack/root` was synchronized into `Project_Binary/root`.

- Runtime Python files checked: **140**
- Python 3.15 AST failures: **0**
- Binary synchronization commit: `5039c5d65141eef5fc2bf5c9f8895ebc96abcc42`
- 111 tracked root files changed in that commit.
- Repository-only files were preserved; synchronization did not delete files merely because they were absent from runtime.

Critical runtime/root files were hash-matched after sync, including:

- `system.py`
- `introloading.py`
- `ui.py`
- `uicuberenewal.py`
- `uicharacter.py`
- `localeinfo.py`
- `uiquest.py`
- `uichat.py`
- `constinfo.py`
- `uimessenger.py`
- `uitooltip.py`
- `playersettingmodule.py`
- `uitaskbar.py`
- `uiskillcolor.py`
- `uinewinventory.py`

## Major resolved runtime blockers

1. **x64 Granny mesh crash**
   - Replaced the old hard-coded `/ 32` Granny type-size assumption with `sizeof(granny_data_type_definition)`.
   - Character/game entry no longer crashes in `CGrannyModel::LoadMeshs`.

2. **Python 3.15 compatibility**
   - Comparator sort conversion.
   - Integer division fixes where Python 2 semantics were required.
   - `time.clock()` migration.
   - old `apply()` removal.
   - Python 3 collection/iteration compatibility fixes.
   - inventory range float issue fixed.
   - Skill Color ConfigParser / file I/O compatibility fixed.

3. **x64 pointer truncation**
   - Emotion icon pointer return converted to x64-safe pointer helper.

4. **Legacy locale byte -> Python Unicode bridge**
   - Added `PythonLocaleString.h`.
   - Selected locale codepage is used instead of globally forcing Turkish.
   - Applied to Event, Skill, Character, Cube Renewal, Application, MiniMap, Quest, IME and WindowManager text paths.

5. **Cube Renewal**
   - Legacy locale byte decoding fixed.
   - Synthetic recipe slot `-1` no longer reaches seal-date lookup as unsigned `65535`.

6. **Guild Building**
   - Legacy locale line loading fixed through locale-safe Python Unicode bridge.

7. **Shaman assets**
   - Official GF Shaman folders merged missing-only.
   - Required dependency scan reached zero missing assets for the inspected Shaman set.
   - Grade registration corrected to avoid nonexistent bare target-motion files.

8. **TaskBar EXP**
   - Python 2 integer division semantics restored for EXP quarter calculations.

9. **Skill Color**
   - `wndMgr.GetText()` moved to locale-safe Unicode bridge.
   - Config file read/write migrated to Python 3-compatible `_io.open`.
   - `UserData/skill_color` directory is created automatically.

## Gameplay validation result

Latest smoke-test syserr reached normal gameplay and normal shutdown.

Observed final scan:

- `Traceback = 0`
- `TypeError = 0`
- `RuntimeError = 0`
- `SystemError = 0`
- `UnicodeDecodeError = 0`
- `FileNotFoundError = 0`
- `NameError = 0`
- `AttributeError = 0`
- `Abort!!!! = 0`
- `ACCESS_VIOLATION = 0`
- `FATAL CLIENT CRASH = 0`

Normal shutdown evidence ended with `PYTHON STARTUP: RunMainScript returned true`.

This is a **general gameplay smoke checkpoint**, not proof that every statically mapped subsystem defect in `RUNTIME_VALIDATION_QUEUE.md` has been tested. The defect-specific runtime queue remains pending.

## Remaining known issue

### Turkish quest / LC mojibake

Quest debug output still shows corrupted Turkish text examples such as:

- `Biyolo­un iste­i`
- `Ejderha ta■² parþas²n²n geli■i`

This does **not** currently cause a traceback or native crash. Next work should determine whether corruption is logger-only or visible in the actual quest UI, then fix the LC/quest conversion path without globally forcing CP1254.

## Next execution order

1. Quest Turkish LC / mojibake investigation and visual verification.
2. Remaining visible gameplay/UI regressions.
3. Ninth-skill chain validation/activation for VNUM **176–183** across client, server, DB/proto and assets.
4. Remove temporary debug noise only after stability remains proven:
   - `DH2 DEBUG`
   - `CIPHER DEBUG`
   - `NET RECV DEBUG`
   - `BOOTTRACE`
5. DX11 remains a later modernization target; do not mix it into current gameplay-stability work.

## Rollback

For source rollback/reference, use:

`checkpoint/x64-py315-gameplay-stable-2026-10-28`

on both `Project_ClientSrc` and `Project_Binary`.

Do not rewrite or move this tag.


## Execution Package Matrix

Aşağıdaki tablo, bu checkpoint için kullanılan server/client yürütme ve build paketlerinin kanonik özetidir.

| Katman | Paket / Sürüm | Durum |
|---|---|---|
| Server OS | **FreeBSD 14.3 amd64** | Aktif |
| Server mimari | **64-bit amd64** | `game` + `db` derlenmiş |
| Server compiler | **GCC / G++ 14** | Aktif |
| Database | **MariaDB 10.11** | Çalışıyor |
| Boost | **1.89** | Kurulu |
| Crypto++ | **8.9** | Kurulu |
| ccache | Kurulu | Sürümü henüz pinlenmedi |
| DevIL | Kurulu | Sürümü henüz pinlenmedi |
| Server locale | `LOCALE = latin1` | Mevcut runtime config |
| Client hedef | **Windows x64 / Release** | Aktif |
| Client compiler | **Visual Studio 2022 / v143** | Aktif |
| Python | **Python 3.15.1 x64** | Aktif, oyun içi doğrulandı |
| Granny | **Granny 2.11.10.0 x64** | Aktif |
| Granny DLL | `granny2_x64.dll` | Runtime dependency |
| CEF | **CEF 90 x64** | Aktif |
| CEF lib | `libcef.lib` | Build dependency |
| libjpeg | Static | Aktif |
| LZ4 | Integrated | Exact sürüm henüz pinlenmedi |
| Crypto++ client | `cryptlib` x64 | Aktif, exact client library version ayrıca doğrulanmalı |
| Miles Sound | `mss32` | Dependency mevcut |
| Renderer | Mevcut legacy renderer | DX11 daha sonraki faz |
| Binary repo Python | **140 .py dosyası** | Python 3.15 AST: **0 hata** |

### Checkpoint repository snapshot

| Repository | Branch | Commit |
|---|---|---|
| `Project_ClientSrc` | `migration/client-x64` | `2f788743b9818329da4d741ff6912faa0affed74` |
| `Project_ServerSRC` | `modernization/x64-wiretime-compat` | `1742dbdc073324bb7154128218954b9555d6551f` |
| `Project_Binary` | `main` | `5039c5d65141eef5fc2bf5c9f8895ebc96abcc42` |
| `Project_Game` | `main` | `24bfe96db414abd070f77f215bece36aaa52e1d0` |
| `Project_DumpProto` | `main` | `1066f4620af451c5f728260eb5f8a086672e6384` |

### Runtime executable

- Executable: `metin2release.exe`
- Configuration: **Release x64**
- SHA256: `F5AA00C4A9A499D61310BBF4435EBD9385CD7A7FA570CA5E26BFBE668638E60C`
- Size: **12,150,272 bytes**
- Stable tag: `checkpoint/x64-py315-gameplay-stable-2026-10-28`


## Post-checkpoint full x64 build stabilization

**Date:** 2026-10-28  
**Project_ClientSrc commit:** `75ad11e2`  
**Full SHA:** `75ad11e2` (branch head at time of validation)  
**Branch:** `migration/client-x64`

### Root cause

A full x64 solution build failed only in `GameLib Release|x64` with:

- `C3859: PCH için sanal bellek oluşturulamadı`
- Windows system error `1455`: pagefile / virtual-memory commit too small
- follow-on `C1076`
- downstream `LNK1104 GameLib.lib` in UserInterface because GameLib never produced its Release x64 library

Other x64 configurations built successfully, proving this was not a source-code or dependency-link correctness failure.

### Minimal fix

Only this project setting changed in `Src/GameLib/gamelib.vcxproj` for `Release|x64`:

```diff
-      <MultiProcessorCompilation>true</MultiProcessorCompilation>
+      <MultiProcessorCompilation>false</MultiProcessorCompilation>
```

The XML serializer formatting noise from the first edit attempt was discarded. Final committed diff is exactly:

- **1 file changed**
- **1 insertion**
- **1 deletion**

### Verification

- `GameLib Release x64` rebuild: **PASS**
- `GameLib.lib` generated successfully
- full x64 solution build subsequently reported all projects successful
- no gameplay/runtime source behavior was changed by this build-only setting

### Git

Commit:

`75ad11e2 build: stabilize GameLib Release x64 memory usage`

The existing stable gameplay tag remains unchanged:

`checkpoint/x64-py315-gameplay-stable-2026-10-28`

Do **not** move or rewrite that tag. This build-system fix is a post-checkpoint stabilization commit layered on top of the gameplay-stable baseline.
