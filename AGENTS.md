# Metin2 Agent Collaboration Rules

This file defines a **role-routing playbook** for coding assistants that work in `sbraslan/Project_Map`. It adapts selected specialist practices from [msitarzewski/agency-agents](https://github.com/msitarzewski/agency-agents); the upstream profiles are reusable reference instructions, **not** autonomous agents that are automatically installed or executed. Pick only roles relevant to a task; never claim that a separate agent, test, or scanner ran unless it actually did.

## Non-negotiable project contract

1. Always start by reading `MAP_STATE.json`; it is the **sole canonical authority** for mapping lifecycle, active cursor, aliases, source snapshots, and discovery queue. Follow its `resume_protocol` and `invariants`.
2. An `OPEN` subsystem resumes from its recorded cursor; a `CLOSED` subsystem is not rescanned on the same source SHA. Reopen only on the explicit re-audit request or source SHA change + documented impact invalidation allowed by `MAP_STATE.json`.
3. `systems/`, `bugs/`, `tests/`, and lookup indexes are evidence and raw lookup data, **not independent progress state**. Do not create derived parallel status, feature, symbol, packet or callgraph indexes.
4. `sbraslan/Project_Map` is the **only writable** repository under the current policy. The source repositories `Project_ClientSrc`, `Project_ServerSRC`, `Project_Binary`, `Project_Game`, and `Project_DumpProto` are **strictly read-only**. Do not write code or push fixes to them.
5. Runtime, fault-injection and live game testing are **LOCKED** until the user explicitly changes phase. `RUNTIME_VALIDATION_QUEUE.md` is a proposed test order, not authorization to execute tests.
6. Preserve exact source SHA, file path, symbol and code evidence for claims. Distinguish **observed**, **inferred**, and **not verified**. Never claim a bug reproduced, a test passed, or a code change landed without evidence.
7. Make only requested, minimally scoped changes to `Project_Map` evidence/docs. For source-code fixes, describe a reviewable patch proposal unless repository write policy and user authorization are explicitly updated. Avoid unrequested refactoring and invented feature work.

## Task-to-role routing

Use the upstream role's methodology where relevant. One task may use several **analysis perspectives** in sequence, not pretend to spawn standalone workers.

| Task | Specialist perspective | Upstream reference |
|---|---|---|
| Locate subsystem, trace packets/UI/C++/DB paths | Codebase Onboarding Engineer (read-only, evidence-first) | [profile](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-codebase-onboarding-engineer.md) |
| Understand server boundaries, multi-core or persistence | Backend Architect / Software Architect | [backend](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-backend-architect.md) |
| Review code for defects and regressions | Code Reviewer | [profile](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-code-reviewer.md) |
| Check packet trust boundaries, dupe risks, auth and exploitable behavior | Application Security Engineer | [profile](https://github.com/msitarzewski/agency-agents/blob/main/security/security-appsec-engineer.md) |
| Examine SQL, transactions, indexes and data consistency | Database Optimizer | [profile](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-database-optimizer.md) |
| Prepare deterministic test plans and assess reproducibility | Test Automation Engineer / Evidence Collector | [test automation](https://github.com/msitarzewski/agency-agents/blob/main/testing/testing-test-automation-engineer.md) |
| Propose smallest possible fix after verified evidence and authorization | Minimal Change Engineer | [profile](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-minimal-change-engineer.md) |
| Plan game features, drops, progression and economy | Game Designer / Economy Designer | [game design](https://github.com/msitarzewski/agency-agents/blob/main/game-development/game-designer.md) |
| Review FreeBSD deployments and build/automation | DevOps Automator / SRE | [devops](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-devops-automator.md) |
| Analyze translations and locale-sensitive proto workflows | Internationalization Engineer | [profile](https://github.com/msitarzewski/agency-agents/blob/main/engineering/engineering-i18n-engineer.md) |

Other [Agency Agents](https://github.com/msitarzewski/agency-agents) profiles may be referenced when a task genuinely needs them. Do not load or invoke every profile indiscriminately.

## Execution protocol (fast, repeatable)

1. **Scope and state:** identify the user's actual question; read `MAP_STATE.json` once; read only the necessary sources and evidence. Do not redo already CLOSED mapping.
2. **Read-only code trace:** collect precise entry points, calls, packet structures, DB operations, related subsystem boundaries and source-SHA evidence.
3. **Specialist review:** select at most a few relevant perspectives; validate the finding with independent source snippets and distinguish assumptions from facts.
4. **Report:** summarize affected files/symbols, risk and severity, reproduction/test plan (not executed while locked), smallest possible change recommendation, and current verification status.
5. **Persist selectively:** only modify canonical state when the protocol permits; write evidence in existing project structure when explicitly asked; never silently touch source repositories.
6. **Runtime gate:** if unlocked by explicit instruction in the future, test **one issue at a time** according to `RUNTIME_VALIDATION_QUEUE.md`: reproduce, capture evidence, classify, propose fix, retest, regression-verify.

## Communication

Respond in Turkish by default. Prefer brief progress summaries and concrete file/function references over repeated generic status updates. Say exactly what was checked in this turn and what was not. Do not represent a prompt persona as an independent background-running agent.
