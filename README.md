# flow-trace-audit

**End-to-end reachability audit for any codebase.** Finds every feature that exists in code but is not wired end to end: dead endpoints, broken calls, buttons that do nothing, responses nobody handles.

> Tracing the flow is the source of truth.

Built and signed by **AVAA DEV** — L'agent IA gouverné. **NO CODE NO DONE.**

## The problem

AI-assisted development generates capable modules fast — and quietly leaves them unwired. The code compiles, the tests pass, the button renders. And nothing happens when you click it. Backend commands sit unreachable; frontend calls target handlers that were renamed three sessions ago; error branches swallow failures silently.

No linter catches this. Type checkers don't cross the UI/backend boundary. It is the single most common failure mode of AI-generated applications.

## What this plugin does

One skill, one method, any stack:

1. **Phase 0 — Discovery**: identifies the project's *real* communication mechanisms from the code (Tauri invoke, WebSocket, HTTP routes, IPC, gRPC, custom bridges…). No assumed architecture.
2. **Phases 1–3 — Three exhaustive inventories**: backend entry points, frontend emitted calls, UI trigger chains — each item proven with `file:line` evidence.
3. **Phase 4 — Reachability matrix**: cross-references the inventories and classifies every gap into nine statuses: `BROKEN-CALL`, `INERT-TRIGGER`, `SWALLOWED-ERROR`, `DEAD-ENDPOINT`, `ORPHAN-RESPONSE`, `DEFINED-NOT-REGISTERED`, `PIPELINE-BREAK`, `CONFIG-DRIFT`, `COGNITION-BREAK`.
4. **Phase 5 — Règle #30**: every command output is read with the five-pass protocol before any conclusion.
5. **Phase 6 — Report with integrity check**: the audit is valid only when every inventoried item is accounted for. Counts must balance.

Flux G traces the cognition of AI agents themselves — perception, real assembled context, recall, traced decision, action, feedback, consolidation — so an agent that ignores instructions, forgets, loops or claims work it did not do is diagnosed like any other broken flow.

The audit modifies zero project files. It produces a diagnostic you can trust, then you decide what to wire.

## Install

```shell
/plugin marketplace add charlesphilippebelhadj/avaa-dev-flow-trace
/plugin install flow-trace-audit@avaa-dev
```

## Use

Explicitly:

```shell
/flow-trace-audit:audit
/flow-trace-audit:audit src/features/settings
```

Or just ask naturally — the skill triggers on requests like *"audit the wiring"*, *"is everything connected end to end?"*, *"why does this button do nothing?"*, *"find dead endpoints"*.

## Method provenance

The canonical methodology — **Debug Pilleur Master, Universal Debugging and Tracing Methodology** (universal version 2026-05-24, flux G 2026-10-26) — is bundled verbatim in `skills/flow-trace-audit/references/DEBUG_PILLEUR_MASTER.md` (canonical, French) with a full English translation in `DEBUG_PILLEUR_MASTER_EN.md`: 7 universal flux types (including agent cognition) with step-by-step verification chains, checklists and schemas, reference commands, 25 documented failure patterns, the Règle #30 five-pass output-reading protocol, the forbidden anti-patterns of AI-assisted debugging, and how to apply the method to development tasks, not only diagnostics.

It was forged on real projects at AVAA DEV, where a governed multi-agent delivery system enforces one doctrine: a feature is not done until its complete flow — trigger → transport → handler → effect → response — is traced and proven. Assumptions are forbidden; evidence is mandatory.

## License

MIT — see [LICENSE](LICENSE).
