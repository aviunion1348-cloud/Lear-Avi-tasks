# Lear Repository Audit Superprompt

You are auditing the Lear / Prash repository at `/home/user/Lear-Avi-tasks` on the fixed branch `arena/01a0e95c-lear-avi-tasks`. The audit date is 2026-09-28 UTC. Do not change application behavior while auditing. Do not install dependencies, contact external providers, mutate cloud resources, send notifications, or create credentials. Work from the repository as the source of truth.

## Mission

Build a source-grounded, end-to-end model of the entire application before proposing or implementing any change. Understand not only what the code is intended to do, but what it actually does at runtime, where state lives, what crosses trust boundaries, which paths are dynamic versus demo/hardcoded, and where documentation, tests, and implementation disagree.

## Required audit method

1. Inventory every tracked file and group it into product code, desktop code, Tauri shell, connectors, actions, AI/brain, watcher, persistence/configuration, tests, fixtures, evaluation harness, scripts, infrastructure, CI, and documentation.
2. Perform a structural pass over all Python and TypeScript/Rust source: module imports, classes, functions, route decorators, action registrations, connector registry entries, persistence paths, environment variables, subprocess/network calls, and test coverage targets.
3. Read deeply enough to trace every important execution path rather than stopping at filenames or docstrings. For each claimed behavior, identify its implementation source and caller/callee chain.
4. Trace the main user journeys end to end:
   - first launch and onboarding;
   - connector connect, validate, status, disconnect, and credential persistence;
   - project/environment/service creation and health polling;
   - dashboard metrics, generated widgets, activity, notifications;
   - watch start, restore, poll, pause/resume, stop, WebSocket broadcast, and notifications;
   - chat intent resolution, telemetry injection, streaming, approval, command execution, and audit;
   - CLI investigate/fix/run/watch/repl/TUI flows;
   - diagnosis, log acquisition, schema validation, memory, correlation, multi-failure diagnosis, remediation, verification, and reconciliation;
   - incidents and approval through dashboard, war room, email, and Slack;
   - demo/storefront/admin surfaces.
5. Map every connector through the shared contract: auth inputs, real provider APIs or local subprocesses, locate/poll/log/stats/watch behavior, write capabilities, normalization, error handling, and action mappings.
6. Map every write action through `plan -> circuit breaker -> permission decision -> approval/secret handling -> execute -> verify -> breaker record -> audit`. Record the action ID, risk tier, reversibility, target, and external side effects.
7. Map every state store and data lifetime. Distinguish process memory, localStorage, YAML, dotenv, JSON, append-only logs, generated HTML, model-provider state, and external-provider state. Record exact paths, schemas, load/save timing, retention limits, and concurrency/atomicity behavior.
8. Audit trust and safety boundaries: secret masking, credential persistence, raw request bodies, shell commands, subprocess use, SMTP/IMAP/webhooks, CORS, WebSocket auth, URL construction, HTML escaping, approval routes, audit completeness, circuit-breaker scope, and demo code.
9. Compare implementation against README, handoff, roadmap, task specs, changelog, CI, and tests. Mark each discrepancy as implemented, partially implemented, planned, stale, demo-only, or unverified.
10. Run only safe local validation available in the environment. At minimum run Python compilation and inspect git status. If tests/builds cannot run because dependencies are absent, record that precisely; do not silently install dependencies or use live credentials.

## Evidence rules

- Prefer exact file paths, symbols, route names, environment keys, and observed code behavior over marketing language.
- Never call a value “live” merely because a UI label says live. Distinguish provider API data, local fixture data, generated fallback data, and hardcoded demo narrative.
- Never assume a database exists. Prove whether SQL/ORM/database files are present and describe the actual persistence layer.
- Treat secrets as toxic data. Do not print, copy, or persist credential values in the audit artifact. Report only key names, masking behavior, and exposure risks.
- Treat an action that can mutate infrastructure as potentially dangerous even if its UI label says “safe.” Record whether approval is enforced at every ingress path.
- Call out stale comments and documentation when they materially misdescribe the current implementation.
- Call out duplicate state, race-prone globals, endpoint contract mismatches, hardcoded provider assumptions, missing authentication, and unavailable test coverage.

## Deliverable format

Produce:

1. A concise executive architecture map.
2. A complete file inventory with categories and counts.
3. A detailed runtime and data-flow map for backend, desktop, CLI, connectors, actions, brain, watcher, incidents, notifications, and persistence.
4. A connector and action capability matrix.
5. A storage/configuration/database statement with exact paths and schemas.
6. A UI/surface map describing what a user sees and how navigation/state changes.
7. A test/build/verification report with commands and exact limitations.
8. An evidence-based discrepancy, risk, and unfinished-work register. Separate confirmed defects from roadmap work and unverified hypotheses.
9. A reusable list of questions for the owner: what change is wanted, desired scope, safety constraints, acceptance criteria, and what to do next.

Do not implement a product change until the owner answers the final questions. Keep the audit itself non-invasive and leave the working tree clean except for explicitly requested audit artifacts.
