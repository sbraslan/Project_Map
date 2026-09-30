# Metin2 Modernization Roadmap

## Confirmed current architecture

### Client
- Win32/x86 only.
- Visual Studio projects target `Win32` and `MachineX86`.
- `LargeAddressAware=true`, but executable remains 32-bit.
- C++20 toolchain is already configured.
- Python 2 libraries are linked from an x86-oriented external dependency tree.

### Server
- FreeBSD game and DB both compile explicitly with `-m32`.
- Current ABI/data structures contain many legacy `long` fields.
- On 64-bit FreeBSD (LP64), `long` changes from 32-bit to 64-bit, so removing `-m32` without an ABI normalization pass can change packet/database/cache structure sizes.

### Graphics
- Renderer is native Direct3D 9.
- Core globals use `LPDIRECT3D9`, `LPDIRECT3DDEVICE9`, `D3DPRESENT_PARAMETERS`, `D3DCAPS9`.
- Renderer uses legacy D3DX matrices, FVF vertex formats, fixed-function transforms and D3D9 state management.

## Strategic goal
Modernize foundation before gameplay-system expansion:
1. stability / deterministic frame pacing;
2. 64-bit readiness;
3. modern rendering abstraction;
4. DirectX 11 backend;
5. higher-quality visual features;
6. gameplay fixes and new systems on top of the modernized base.

## Phase 0 — Baseline / measurement
Before architecture changes:
- reproducible Debug/Release/Distribute builds;
- baseline FPS/frametime in town, crowded scene, dungeon and empty map;
- RAM usage;
- draw-call count where possible;
- loading time;
- server tick CPU usage;
- crash/log baseline.

## Phase 1 — 64-bit readiness without changing architecture

### Shared protocol/data cleanup
Replace ABI-sensitive wire/persistence types:
- `long`, `unsigned long`, raw pointer-sized assumptions;
- ambiguous `int` where serialized;
- packet structures whose size depends on platform ABI.

Use fixed-width:
- `int8_t/uint8_t`
- `int16_t/uint16_t`
- `int32_t/uint32_t`
- `int64_t/uint64_t`

Add compile-time checks:
- `static_assert(sizeof(PacketType) == expected)`;
- serialization-size tests;
- explicit packing only where protocol requires it.

### Server x64
After protocol normalization:
- remove `-m32` in an isolated x64 build profile;
- rebuild every static/external library as x64;
- resolve pointer truncation / printf format / casts;
- compare packet and DB structure sizes against x86 baseline;
- run login/item/shop/guild/storage persistence smoke tests.

### Client x64
Create `x64` Visual Studio configurations rather than replacing Win32 immediately.
Required dependency audit:
- Python 2 runtime/libs;
- CEF;
- Granny/model runtime;
- DevIL/image stack;
- Crypto/OpenSSL and other static libs;
- any anti-cheat/protect library.

Both x86 and x64 clients should coexist until parity is proven.

## Phase 2 — DX9 renderer cleanup / abstraction
Do not replace D3D9 calls ad hoc.

Introduce a render abstraction layer around:
- device creation/present/reset;
- vertex/index buffers;
- textures;
- render states;
- transforms/constants;
- shaders;
- render targets;
- sampler states;
- draw calls.

Keep D3D9 as Backend A first. The game must still render exactly as before through the abstraction.

Benefits:
- isolates legacy rendering;
- makes DX11 migration incremental;
- allows profiling/batching/frame-pacing improvements before DX11.

## Phase 3 — DirectX 11 backend
Implement Backend B using:
- `ID3D11Device`
- `ID3D11DeviceContext`
- `IDXGISwapChain`
- vertex/index/constant buffers;
- HLSL vertex/pixel shaders;
- input layouts replacing FVF;
- sampler/rasterizer/blend/depth-stencil state objects.

Fixed-function D3D9 operations must become shaders:
- world/view/projection transforms;
- lighting/material calculations;
- texture-stage operations;
- fog;
- alpha/blending behavior;
- terrain/object/effect rendering.

D3DX9 dependencies should be replaced or isolated.

## Phase 4 — Visual/quality improvements
After DX11 parity:
- stable high-resolution rendering;
- anisotropic filtering;
- better shadow maps;
- improved terrain/object draw distance;
- higher-resolution texture pipeline;
- improved water;
- gamma/brightness controls;
- optional MSAA/FXAA/SMAA/TAA depending on engine constraints;
- modern post-processing;
- better particle/effect batching;
- optional PBR-like materials only where assets support them.

## Phase 5 — Performance modernization
Independent of visual quality:
- frame-time limiter and stable frame pacing;
- remove busy waits;
- CPU/GPU profiling;
- draw-call batching;
- visibility/culling improvements;
- resource streaming/loading;
- reduce per-frame allocations;
- optimize Python/C++ bridge hot paths;
- background-safe asset decoding/loading where engine lifetime rules permit it.

## Difficulty assessment

### Server x64
Difficulty: medium-high.
Main challenge is ABI/protocol cleanup, not compiler support.
Do this before changing wire protocols or adding new major systems.

### Client x64
Difficulty: high.
Main challenge is external x86 dependencies and pointer-size assumptions.
Renderer itself does not need DX11 to become x64.

### DX9 -> DX11
Difficulty: very high if attempted as a direct replacement.
Manageable as a staged renderer-backend migration.
The current engine is deeply coupled to D3D9 fixed-function/state-manager semantics.

## Recommended execution order
1. baseline/profiling;
2. fixed-width protocol/data cleanup;
3. server x64;
4. client x64 dependency audit + dual x86/x64 build;
5. renderer abstraction while retaining DX9;
6. DX11 backend to visual parity;
7. visual upgrades;
8. deeper performance work;
9. gameplay-system refactors/new features.

## Core rule
Do not combine x64 migration and DX11 rewrite in one change set. Each must have its own compatibility gate and rollback path.
