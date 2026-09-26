# guild — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-GUILD-001 — Offline member removal + ENABLE_PULSE_MANAGER null pointer
`CGuild::RemoveMember`:
`LPCHARACTER ch = FindByPID(pid)`
sonrasında `if (ch)` kontrolünden **önce**
`ch->GetPlayerID()`
kullanıyor.

`ENABLE_PULSE_MANAGER` aktif build'de offline member remove işlemi null dereference riski taşıyor.
Guild Storage dışı genel guild bug'ı olarak ayrıca kaydedildi.
