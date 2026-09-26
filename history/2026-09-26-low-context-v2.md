# 2026-09-26 — Low-context state v2

Project_Map already used subsystem-scoped maps and an overwrite-only CURRENT.md. This migration completes the design by adding a machine-readable `STATE.json` and making GitHub—not chat history—the canonical continuation source.

Key rules:
- startup reads are limited to STATE.json, CURRENT.md, and the active subsystem;
- archive and completed subsystems are excluded from normal continuation;
- source repos remain read-only;
- source HEAD snapshots are persisted for targeted invalidation;
- every meaningful turn updates the active map plus CURRENT.md and STATE.json;
- Git commit history replaces ever-growing prose checkpoint logs.

Active subsystem at migration: Hunting System.
Verified active bugs: BUG-HUNT-001 through BUG-HUNT-004.
