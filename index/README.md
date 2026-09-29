# Mapping Acceleration Index

**Status:** READY
**Generated:** 2026-09-29
**Source repositories:** READ-ONLY

## Purpose
This directory is the machine-readable acceleration layer for Metin2 static mapping. It avoids depending on GitHub Code Search indexing and keeps Project_Map as the only writable mapping repository.

## Files
- `repositories.json` — source repository tree SHA, counts, extension coverage.
- `files.json` — mapping-relevant code/text inventory for ClientSrc, ServerSRC, Binary, Game and DumpProto.
- `features.json` — canonical subsystem/system/bug/test routing.
- `symbols.json` — symbol seed index extracted from verified Project_Map state.
- `packets.json` — packet/header seed index extracted from verified mapping state.
- `callgraph.json` — static flow/call-edge seed graph from mapped flows.

## Lookup order
1. `features.json`
2. `symbols.json` / `packets.json`
3. `callgraph.json`
4. `files.json`
5. Fetch only the exact source files required for verification.

## Refresh rule
Compare the current source repository recursive-tree SHA with `repositories.json`. Rebuild the affected inventory/index only if the SHA changed.

## External documentation rule
Context7 is supplementary only. Use it to verify external library/API semantics (C++, Python, Boost, CEF, build tooling, etc.). It is not the authoritative locator for Metin2 source code.

## Safety / source policy
ClientSrc, ServerSRC, Binary, Game and DumpProto remain read-only during mapping. Only Project_Map may be updated unless the user explicitly changes this rule.
