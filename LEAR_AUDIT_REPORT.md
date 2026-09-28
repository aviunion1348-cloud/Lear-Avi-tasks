# Lear / Prash Application Audit Report

**Audit date:** 2026-09-28 UTC
**Repository:** `/home/user/Lear-Avi-tasks`
**Branch:** `arena/01a0e95c-lear-avi-tasks`
**Base commit:** `a3a4df3b8c67a5912a3f267b678cef0330c7946c`
**Scope:** full repository inventory, runtime architecture, backend/API, React/Tauri desktop, CLI/TUI/REPL, connectors, actions, diagnosis brain, watcher, persistence/configuration, incidents, external notification paths, infrastructure, tests, docs, and known discrepancies.

The reusable audit operating instructions are in [`LEAR_AUDIT_SUPERPROMPT.md`](./LEAR_AUDIT_SUPERPROMPT.md). No product behavior was changed during this audit.

---

## 1. Executive summary

Lear is a local-first AI DevOps/SRE application with two principal faces:

1. **Desktop console:** React 19 + TypeScript + Vite, intended to run inside a Tauri 2 shell but also usable as a browser/Vite page. It talks to a local FastAPI bridge with relative `/api/...` URLs for HTTP. It provides onboarding, integrations, projects/stacks, service telemetry widgets, watch controls, dashboard activity, notifications, settings, chat, and incident-related chat surfaces.
2. **Python agent/server surfaces:** the `prash` package provides a CLI/TUI/REPL, a FastAPI desktop bridge, 13 provider connectors, a permissioned action dispatcher, diagnosis and remediation flows, a watcher, local memory, incidents, email/Slack integration, and several FastAPI-served HTML/demo pages.

The core trust model is local credentials plus local file persistence. There is **no SQL database, ORM, migration layer, or hosted persistence implementation in this repository**. Configuration and runtime state are split across `.env`, `prash.yaml`, `.prash/*.json`, `.prash/audit.log`, generated email HTML, process memory, browser localStorage, and external provider state.

The main operational pipeline is:

```text
Desktop / CLI / chat / incident ingress
        ↓
connector registry or intent resolver
        ↓
provider read surface or action dispatcher
        ↓
plan → circuit breaker → permission decision → approval/secret → execute
        ↓
verify → audit → local activity/notification → UI/WebSocket
```

The repository has a substantial implementation, but the current code and roadmap make clear that it is not yet production-hardened. Confirmed high-impact observations include:

- FastAPI CORS is permissive: `allow_origins=["*"]` with credentials enabled.
- `/ws/events` accepts connections without authentication.
- API routes generally have no user/session authentication or CSRF protection; incident approval endpoints accept both GET and POST.
- The desktop and server use multiple independent WebSocket consumers, each opening `ws://127.0.0.1:8000/ws/events`.
- The frontend’s `LearContext` looks for `c.configured`, while the registry serialization returns `status`, so the context-level configured count can remain zero even when integrations are configured.
- The Tauri Rust shell only exposes the template `greet` command; it does not own backend lifecycle, credentials, or native functionality.
- The Vite proxy, frontend WebSocket default, npm root server script, and launchers assume loopback addresses, which is incompatible with a browser-facing sandbox/live-preview deployment without configuration changes.
- Demo/incident/channel code contains fixed infrastructure narrative, localhost URLs, default recipient addresses, and synthetic-looking telemetry text even though the desktop bridge advertises dynamic/no-hardcoded data.
- `server.py` is approximately 3,895 lines and `cli.py` approximately 1,360 lines; the roadmap explicitly marks decomposition and state extraction as not started.
- The environment used for this audit had no `pytest` executable and no `desktop/node_modules`; Python compilation passed, but backend tests and frontend build/tests could not run.

---

## 2. Complete repository inventory

At the audit snapshot:

- **429 tracked files** (`git ls-files`).
- **429 working-tree files** after excluding `.git`, ignored `__pycache__`, and absent `desktop/node_modules`/runtime `.prash` content.
- Approximately **30,018 Python lines** under `prash/` and **13,191 TypeScript/TSX/CSS lines** under `desktop/src/`.
- The exact path-by-path inventory is appended in [§15](#15-path-by-path-inventory).

### 2.1 Product and repository documentation

- `README.md` — setup, local Kubernetes option, CLI examples, status.
- `PRASH_V2.md` — long architecture/specification and implementation history.
- `LEAR_HANDOFF.md` — desktop handoff and completed feature references.
- `CHANGELOG.md` — change history.
- `ROADMAP.md` — production transformation plan; Week 1 security, decomposition, UI, and testing work is listed as not started.
- `CONNECTOR_REWRITE_SPEC.md` — shared connector/watch/event rewrite specification.
- `E2E_TEST_CHECKLIST.md`, `TESTING_CHECKLIST.md`, `TESTING_SETUP.md` — validation and known issue records.
- `agent_guidelines.md` — repository guidance.
- `desktop/README.md` — still mostly the default Tauri/Vite template README.
- `tasks/` — implementation specifications for connectors, actions, brain, CLI, cross-cutting safety, desktop features, watcher, infrastructure, demos, and the Week 1 backlog.
- `tasks/questions.md` — interview/discovery questions for AWS and DevOps practitioners.

### 2.2 Python runtime

- `prash/server.py` — FastAPI bridge, lifecycle, connectors, projects, settings, chat, widgets, watches, notifications, demos, incidents, email, Slack.
- `prash/cli.py` — argparse entry point, dispatcher construction, `run`, `fix`, `investigate`, `stats`, `watch`, `repl`, `tui`, `audit`, `config`, `actions`, circuit controls, setup.
- `prash/dispatch.py` — action orchestration.
- `prash/permissions.py` — permission modes and decision rules.
- `prash/audit.py` — append-only JSON-lines action audit.
- `prash/circuit_breaker.py` — local per-resource action cap.
- `prash/credentials.py` — dependency-free local `.env`/secret loader for CLI/action paths.
- `prash/setup.py` — interactive local credential setup and masking.
- `prash/connector_registry.py` — registry metadata, auth schema, widget templates, dynamic classes, cache, configured-state discovery.
- `prash/watcher.py` — Kubernetes foreground watcher plus connector-driven watch loops and notification fan-out.
- `prash/intent.py` — fast free-text intent resolution plus bounded LLM fallback.
- `prash/fix.py` — diagnosis-to-action seam for Kubernetes, CI, AWS, GCP, Datadog, PagerDuty, and Grafana.
- `prash/repl.py`, `prash/tui.py`, `prash/ui.py` — interactive shells and Rich/Textual presentation.
- `prash/widget_generator.py` — LLM/fallback widget generation and YAML persistence.
- `prash/notifications.py` — Slack, Discord, SMTP email, and Twilio WhatsApp notifier abstraction.
- `prash/incident_manager.py`, `prash/incident_page.py` — persisted incident lifecycle and HTML war room.
- `prash/chat_manager.py` — channel-isolated persistent chat sessions.
- `prash/email_service.py`, `prash/slack_service.py`, `prash/conversation_router.py` — email/Slack rendering, dispatch, inbound routing, IMAP listener, cross-channel chat.
- `prash/customer_store.py`, `prash/demo_page.py`, `prash/admin_panel.py` — storefront/admin/demo HTML and demo actions.
- `prash/__init__.py`, `prash/actions/__init__.py`, `prash/connectors/__init__.py`, `prash/brain/__init__.py` — package markers/exports.

### 2.3 Connectors

- Shared contract: `prash/connectors/base.py`.
- Provider modules: `aws.py`, `azure.py`, `gcp.py`, `kubernetes.py`, `vercel.py`, `github.py`, `gitlab.py`, `datadog.py`, `grafana.py`, `pagerduty.py`, `snyk.py`, `gitleaks.py`, `terraform.py`.
- Test fixtures: `prash/connectors/testdata/{broken-pod,configmap-mismatch,oom-limit,sample-app,silent-crash-pod,wedged-pod}.yaml`.

### 2.4 Actions

The dispatcher registers 30 actions from these modules:

`open_pr.py`, `missing_secret.py`, `restart_pod.py`, `rollback.py`, `scale.py`, `edit_config.py`, `exec_command.py`, `apply_ci_fix.py`, `apply_gitlab_ci_fix.py`, `gcp_alert.py`, `execute_aws.py`, `execute_gcp.py`, `execute_azure.py`, `aws_alert.py`, `pagerduty_incident.py`, `pagerduty_page.py`, `vercel_deploy.py`, `datadog_mute.py`, `datadog_alert.py`, `github_alert.py`, `gitlab_alert.py`, `grafana_silence.py`, `snyk_ignore.py`, `gitleaks_escalate.py`, `terraform_init.py`, and `terraform_apply.py`. Shared definitions are in `contract.py`; `alert_base.py` is a base action module but is not separately registered.

### 2.5 Brain / diagnosis

- `diagnosis_agent.py` — structured model prompt, tool/investigation flow, validation/retry, CI/runtime/monitoring context formatters, error signatures.
- `kimi_client.py` — lazy DeepSeek/Kimi AsyncOpenAI clients, loop-aware recreation, tool calls, retry/fallback logic.
- `schemas.py` — Pydantic diagnosis, exact file edits, options menu, runtime/monitoring actions, validation/coercion.
- `log_fetcher.py`, `gitlab_log_fetcher.py` — GitHub ZIP/GitLab logs, filtering, truncation, failure-job prioritization.
- `multi_diagnosis.py` — independent per-job diagnoses with bounded concurrency and partial results.
- `correlation.py` — pure cross-connector event clustering with a two-minute default window.
- `edit_repair.py` — grounding/repair of model-proposed edits.
- `local_memory.py`, `repo_memory.py` — local JSON episodic memory and prompt formatting; hosted/Supabase memory was intentionally not ported.

### 2.6 Desktop frontend

- Shell/state: `App.tsx`, `main.tsx`, `context/LearContext.tsx`, `index.css`.
- Navigation: `Sidebar.tsx`, `PrashWindow.tsx`, `ErrorBoundary.tsx`, `ModesUI.tsx`.
- Onboarding: `Wizard.tsx`, `StepIntegration.tsx`, `StepCredentials.tsx`, `StepLocation.tsx`, `StepNotifications.tsx`.
- Integrations: `Integrations.tsx`, `ConnectorForm.tsx`.
- Projects: `Projects.tsx`, `ProjectCreate.tsx`, `ProjectDetail.tsx`.
- Dashboard/telemetry: `Dashboard.tsx`, `ServiceWidget.tsx`, `WidgetConfigurator.tsx`, `widgets/{MetricGauge,MetricCard,MetricLineChart,BarChart,StatusGrid,EventTimeline}.tsx`.
- Chat: `Chatbot.tsx`, `ChatWorkspace.tsx`, `ChatMessage.tsx`.
- Operations: `WatcherPanel.tsx`, `ActivityLog.tsx`, `Notifications.tsx`, `NotificationToast.tsx`, `Settings.tsx`.
- Hooks: `useWebSocket.ts`, `useWatcher.ts`, `useNotifications.ts`, `useConnectorStatus.ts`.
- Tests: `src/__tests__/ConnectorForm.test.tsx`, `Integrations.test.tsx`; `src/test/setup.ts`.
- Assets: `src/assets/react.svg`, public/favicon and Lear/Tauri/Vite images.

### 2.7 Tauri shell and build

- `desktop/src-tauri/src/lib.rs`, `main.rs` — default Tauri app plus `greet` command.
- `desktop/src-tauri/tauri.conf.json` — product identity `Prash`, window, Vite URL, bundle targets, `csp: null`.
- `desktop/src-tauri/capabilities/default.json` — core and opener permissions.
- Rust lock/config/build files and icon assets.
- `desktop/package.json`, `package-lock.json`, Vite/Tailwind/PostCSS/TypeScript configs.
- Root `package.json` — concurrent backend/frontend scripts; backend script binds `127.0.0.1`.
- `lear.bat`, `lear.ps1` — Windows setup, venv/npm install, loopback servers, browser opening, restart loop.

### 2.8 Tests, fixtures, evals, scripts, infrastructure

- `tests/` — connector unit tests, action/dispatch tests, brain/schema/memory/correlation tests, CLI/TUI/REPL, watcher, notifications, service connection/API tests, widget generation, and Grafana integration.
- `evals/` — diagnosis cases/results, `run_eval.py`, `score.py`.
- `scripts/demo/` — local/demo cluster manifests, services, failure injection, memory seeding, load generation, reset/deploy scripts.
- `scripts/testing/` — provider breakage fixtures, recovery demo, correlation measurement.
- `infrastructure/` — small Terraform `null_resource` verification fixture, variables, output, required version, lock file.
- `.github/workflows/ci.yml` — three-OS Python test job, kind Kubernetes live tests, Grafana mock integration.
- `.github/workflows/always-broken.yml` — intentionally failing `prash-broken-ci` fixture only.

---

## 3. Application surfaces and startup

### 3.1 Browser/Tauri desktop startup

1. Vite serves the React app at port 1420. The Tauri config points at `http://localhost:1420` in development and `../dist` for packaged frontend output.
2. `main.tsx` renders `App`.
3. `App.tsx` calls `GET /api/config` once during startup.
4. If no configured services are returned, the app renders `Wizard` as an onboarding gate.
5. Otherwise it renders the sidebar and the selected main surface, with the global Copilot/chat drawer and toast layer.
6. URL query parameters can select a tab or open chat. The default context tab is `dashboard`.
7. `LearContext` loads system version, connectors, projects, and active watches, then maintains active project/environment, watcher state, chat state, notification count, and toast state.
8. Project and environment choices are browser-local only under `lear_active_project_id` and `lear_active_environment`.

### 3.2 Python server startup

`prash.server`:

- Loads root `.env` with `dotenv.load_dotenv(..., override=True)` at import time.
- Uses root-relative `ENV_PATH`, `YAML_PATH`, and `.prash/notifications.json` paths.
- Creates a FastAPI app with a lifespan handler.
- On lifespan start, restores watches from `prash.yaml`, loads notifications, starts the WebSocket polling task, starts the connector health-check task, and attempts to start the IMAP listener.
- On shutdown, cancels tasks, stops the IMAP listener, stops watch handles, and clears in-memory watch structures.
- Keeps watches, WebSocket clients, activity, notifications, connection states, and dashboard cache in module-level mutable globals.

### 3.3 CLI startup

`prash.cli:main` loads local credentials, exports relevant cluster environment variables, builds the dispatcher, and routes commands through the permission/audit/circuit path. The CLI can run as one-shot commands, a Textual TUI, or a persistent REPL. The action dispatcher itself is shared conceptually with server chat execution, but the FastAPI chat executor invokes CLI parsing in-process.

---

## 4. Desktop UI: what users see and how it operates

### 4.1 Navigation

The sidebar exposes Dashboard, Lear Chat, Projects, Integrations, Activity Log, Notifications, and Settings. The app also displays connection/watcher indicators, active project and environment controls, the app version, and a global Copilot affordance.

### 4.2 Wizard

The onboarding wizard fetches registry metadata, presents integration selection and credential/location/notification steps, posts configuration, and can auto-import configured services into the default project. It is shown whenever `/api/config` reports no configured services.

### 4.3 Dashboard

`Dashboard.tsx` fetches `/api/dashboard/summary`, `/api/dashboard/activity`, and `/api/incidents`. It presents an aggregate health score, service counts, active watches, recent activity, connected service cards, incident status, quick actions, and provider-specific explanatory content. Service cards route into telemetry or Copilot. The dashboard uses dynamic API results but still contains provider-specific presentation branches and hardcoded wording that should be treated separately from live metrics.

### 4.4 Integrations

`Integrations.tsx`:

- Fetches `/api/connectors`.
- Filters/searches the registry by category and configured/unconfigured state.
- Expands `ConnectorForm` inline.
- Calls `/validate` for a manual health check.
- Calls `/disconnect` after an inline confirmation.
- Displays masked metadata, identity, last verification, docs, and transient success/error messages.

`ConnectorForm.tsx` renders the registry-provided auth field schema, including text/password/file/select/textarea styles, submits to `/api/connectors/{id}/connect`, preserves existing values where supported, and never intentionally displays raw credential values.

### 4.5 Projects and stacks

Projects are YAML-backed objects with:

```text
project = {
  id, name, created_at,
  environments: [
    { name, services: [
      { connector_id, resource_id?, display_name? }
    ] }
  ]
}
```

`Projects.tsx` lists projects, auto-imports configured connectors, opens a four-step create modal, deletes projects, and opens details. `ProjectCreate.tsx` creates project identity, environments, attached services, and posts `/api/projects`. `ProjectDetail.tsx`:

- polls `/api/projects/{id}/status` immediately and every 20 seconds;
- switches environments;
- adds/removes services and environments with `PUT`;
- discovers real resources from `/api/connectors/{id}/resources`;
- routes a service to Dashboard telemetry or service-scoped Copilot.

### 4.6 Telemetry widgets

`ServiceWidget.tsx` loads connector metadata, metrics, status, widgets, and active watches. It can configure/reset widget layouts, display six widget families (gauge, line chart, bar chart, metric card, event timeline, status grid), start/stop watches, and provide connector-specific action affordances. `WidgetConfigurator.tsx` calls `/generate-widgets` and can save results. The server stores custom layouts under `widgets.{connector_id}.{resource_id-or-default}` in `prash.yaml`.

Generation is:

```text
live connector state/stats + registry templates + user prompt
        ↓
LLM JSON generation when an AI key exists
        ↓
JSON extraction/validation/layout normalization
        ↓
deterministic template/metric fallback when unavailable
        ↓
optional YAML persistence
```

### 4.7 Watcher panel and live state

`WatcherPanel.tsx`, `useWatcher.ts`, `LearContext`, `useNotifications`, and service widgets each consume the watch API/WebSocket surfaces. They maintain overlapping local views of active watches/events. Backend events are JSON payloads with `watch_id`, `connector`, `event_type`, `summary`, `raw`, and `timestamp`.

The frontend WebSocket default is hardcoded to `ws://127.0.0.1:8000/ws/events`. Multiple `useWebSocket()` instances open separate sockets rather than sharing a single provider. This is a concrete duplication and deployment concern.

### 4.8 Activity and notifications

`ActivityLog.tsx` queries `/api/activity` and supports text, connector, type, severity, time-range, and pagination views. It combines live in-memory events and disk audit entries. `Notifications.tsx` queries the persistent notification queue, marks individual/all items read, clears the queue, and routes relevant notifications to Dashboard, Settings, or chat. `NotificationToast.tsx` renders transient toasts from the context queue.

### 4.9 Chat and Copilot

There are two desktop chat experiences:

- `Chatbot.tsx` — global/service-scoped drawer, greeting, message stream, incident context, email/Slack controls, and action execution.
- `ChatWorkspace.tsx` — persistent session workspace with origin filters for dashboard, Slack, email, and incidents.

Chat can use direct telemetry-driven SRE analysis, fast intent resolution, async LLM intent fallback, SSE streaming, persistent sessions, incident chat, and action execution. A user click on “Execute Action” is treated by the server as the approval and temporarily replaces CLI’s interactive ask implementation with an always-true `_ChatUiApprovedAsk` for that request.

### 4.10 Settings

`Settings.tsx` fetches `/api/settings`, `/api/config`, and `/api/system/version`. It edits model, permission mode, poll interval, retention days, desktop notifications, degraded alerts, Slack, Discord, and PagerDuty settings. The server persists settings into both `prash.yaml` and selected `.env` keys.

---

## 5. Backend/API map

### 5.1 Connector and resource routes

- `GET /api/connectors`
- `GET /api/connectors/{connector_id}`
- `POST /api/connectors/{connector_id}/connect`
- `POST|DELETE /api/connectors/{connector_id}/disconnect`
- `POST /api/connectors/{connector_id}/check`
- `GET /api/connectors/{connector_id}/validate`
- `GET /api/connectors/{connector_id}/status`
- `GET /api/connectors/{connector_id}/metrics`
- `GET /api/connectors/{connector_id}/resources`

The registry supplies auth fields, missing fields, metadata, widget templates, configured state, and capability flags. Credential connection authenticates the candidate before atomically persisting owned keys into `.env`; unknown fields and missing required fields are rejected. Candidate provider values are temporarily projected into `os.environ` under a lock and restored afterward.

`/status` without a resource reads cached connection lifecycle state; with a resource it calls `poll_state`. `/validate` clears the connector cache to force fresh authentication. The periodic health loop runs approximately every 60 seconds and skips connectors already marked expired/error unless explicitly checked.

### 5.2 Watch routes and WebSocket

- `POST /api/connectors/{id}/watch`
- `DELETE /api/connectors/{id}/watch`
- `POST /api/connectors/{id}/watch/pause`
- `POST /api/connectors/{id}/watch/resume`
- `GET /api/watch/poll`
- `GET /api/watch/active`
- `WS /ws/events`

Watch IDs are normally `{connector}:{target}`. Active handles and metadata are in memory; `active_watches` in `prash.yaml` is used for restart restoration. The background loop ticks every second, honors each watch interval, polls non-paused handles, records events in `_activity_log` up to 1,000 entries, changes status to degraded/error after repeated poll failures, emits recovery transitions, creates notifications capped at 100, persists notifications, and broadcasts JSON to all WebSocket clients.

The WebSocket accepts `ping`, `pause`, `resume`, and `stop` messages. There is no handshake token or authorization check.

### 5.3 Activity/dashboard/system routes

- `GET /api/activity` — merges live events and up to 500 audit records, then filters and paginates.
- `GET /api/dashboard/summary` — 10-second in-memory cache; counts configured connectors, project services, health states, watches, projects.
- `GET /api/dashboard/activity` — recent live activity and watch errors.
- `GET /api/system/version` — reports version `2.4.0`, platform/runtime, total/configured connector counts.
- `GET /api/status` — legacy dynamic status alias.

### 5.4 Project routes

- `GET /api/projects`
- `POST /api/projects`
- `PUT /api/projects/{project_id}`
- `GET /api/projects/{project_id}/status`
- `DELETE /api/projects/{project_id}`
- `POST /api/projects/auto-import`

The project layer validates connector IDs against the registry but does not require each service’s resource ID to exist. Status polls configured connector resources and maps connector states into healthy/warning/error/unknown. Auto-import writes a new default project containing one account/service entry per configured connector.

### 5.5 Config/settings routes

- `GET /api/config` — masked `.env` overview, configured service map, projects.
- `POST /api/config` — non-connector settings only; connector auth keys are rejected and must go through `/connect`.
- `GET /api/settings`
- `POST /api/settings`

`/api/config` masks values using a short prefix/suffix format. `/api/settings` currently returns webhook/routing-key values in its response path rather than a uniform masked representation; this is a confirmed secret-exposure concern.

### 5.6 Chat routes

- `GET /api/chat/greeting`
- `POST /api/chat`
- `POST /api/chat/stream` — SSE
- `POST /api/chat/execute`
- `GET /api/chat/sessions`
- `GET /api/chat/sessions/{session_id}`
- `POST /api/chat/sessions`
- `POST /api/chat/sessions/{session_id}/message`
- `POST /api/chat/upload`

Direct analytical questions go through a telemetry-gathering SRE path. Other prompts go through fast intent, then async model tool resolution. The execution endpoint parses CLI argv in-process, captures stdout/stderr, runs the selected action, records audit/activity, and returns output. It is powerful and should be treated as an action boundary, not merely a chat endpoint.

### 5.7 Notification/widget routes

- `GET /api/notifications`
- `POST /api/notifications/{notification_id}/read`
- `DELETE /api/notifications`
- `POST /api/connectors/{id}/generate-widgets`
- `GET /api/connectors/{id}/widgets`
- `PUT /api/connectors/{id}/widgets`
- `DELETE /api/connectors/{id}/widgets`

### 5.8 Demo/storefront/admin routes

HTML: `/`, `/store`, `/admin`, `/demo`, `/incident/{incident_id}`.
Demo/API: `/api/demo/status`, `/api/demo/health`, `/api/demo/inject-failure`, `/api/demo/auto-fix`, `/api/demo/reset`, `/api/demo/customer-checkout`, `/api/demo/emails/latest`, `/api/demo/send-email`, `/api/demo/inject-db-failure`, `/api/demo/auto-failover-db`, `/api/demo/inject-timeout`, `/api/demo/inject-load`.

These routes are demo/control-center surfaces and can invoke real subprocess/Kubernetes remediations in incident paths; they are not just static pages.

### 5.9 Incident and channel routes

- `GET /api/incidents`
- `GET /api/incident/latest`
- `GET /api/incident/{id}`
- `GET|POST /api/incident/{id}/approve`
- `GET|POST /api/incident/{id}/deny`
- `POST /api/incident/{id}/chat`
- `POST /api/incident/{id}/email-reply`
- `POST /api/email/poll-now`
- `POST /api/email/chat`
- `GET /api/slack/status`
- `POST /api/slack/chat`
- `POST /api/slack/command`
- `POST /api/slack/events`
- `POST /api/slack/interactivity`

These routes expose the incident approval and bidirectional channel model. Approval URLs are embedded into email/Slack HTML and currently have no authenticated actor/session boundary.

---

## 6. Connector matrix

All connectors inherit the conceptual contract in `connectors/base.py`: `authenticate`, `locate`, optional `fetch_logs`, `poll_state`, `watch`, and `get_stats`. Every event is normalized as timestamp, connector, event type, summary, and provider-specific raw payload.

| ID | Provider/category | Registry auth fields | Registry flags | Read/write behavior |
|---|---|---|---|---|
| `aws` | AWS EC2 / infrastructure | `AWS_ACCESS_KEY_ID` required; `AWS_SECRET_ACCESS_KEY` required; `AWS_REGION` optional | watch, stats, execute | STS/EC2 auth; instance locate/state; CloudWatch/log reads; watch/stats; EC2 command execution via action |
| `azure` | Microsoft Azure / infrastructure | subscription, tenant, client ID, client secret required; location optional | watch, stats, execute | Azure identity/VM state/log reads; VM command execution |
| `gcp` | Google Cloud / infrastructure | `GCP_PROJECT_ID` required; service-account file optional; region optional | watch, stats, execute | Compute/Cloud Monitoring reads and watch; GCE command execution |
| `kubernetes` | Kubernetes / infrastructure | kubeconfig/context/namespace all optional | watch, stats, execute | pod locate/status/logs/events/stats/watch; restart, scale, exec, ConfigMap/Secret edit, rollback/apply actions |
| `vercel` | Vercel / infrastructure | `VERCEL_TOKEN` required; team optional | watch, stats, execute | deployment locate/log/state/watch/stats; redeploy and rollback |
| `github` | GitHub Actions / CI/CD | `GITHUB_TOKEN` required; repo optional | watch, stats, execute false | workflow/repo/check-run/log reads; watch/stats; PR/issue/re-run/apply-fix actions |
| `gitlab` | GitLab CI / CI/CD | `GITLAB_TOKEN` required; base URL optional | watch, stats, execute false | pipeline/project/job trace reads; watch/stats; MR/issue/apply-fix actions |
| `datadog` | Datadog / monitoring | API key + app key required; site optional | watch, stats, execute true | monitor state/log/event/stats/watch; mute monitor and alert actions |
| `grafana` | Grafana / monitoring | URL + API key required | watch, stats, execute true | alert rule/annotation state/stats/watch; silence action |
| `pagerduty` | PagerDuty / monitoring | REST API key required; from-email/routing-key optional | watch, stats, execute true | incident/on-call/dependency state/stats/watch; acknowledge, resolve, trigger/page actions |
| `snyk` | Snyk / security | API token + org ID required | watch, stats, execute true | project vulnerability state; bounded ignore action |
| `gitleaks` | local Gitleaks / security | binary path optional | watch, stats, execute false | local executable scan, state/log-like results; never includes secret text; escalation action can page PagerDuty |
| `terraform` | Terraform / IaC | state path/use-cloud/API token all optional | watch, stats, execute true | local state/drift reads and watch; init/apply actions |

### Connector lifecycle details

- Registry instances are lazily constructed and cached by connector ID.
- `/validate` explicitly clears a connector cache before checking because provider auth state may otherwise remain stale.
- Server connection state is cached in `_connection_states`; the cache tracks `status`, `last_verified`, `last_checked`, `error`, and `identity`.
- `.env` is treated as the root server credential store. The CLI’s `CredentialStore` can fall back to `~/.prash/.env`, but the FastAPI bridge uses its root-relative `.env` path.
- Resource discovery is not uniform: the server has explicit discovery logic for AWS, Kubernetes, GitHub, Vercel, and Datadog; other registered connectors can return an empty resource list even when configured.
- Provider APIs and local subprocesses are not exercised in this audit because that would require credentials and could mutate external systems.

---

## 7. Action, permission, and safety matrix

### 7.1 Dispatcher contract

`Dispatcher.run()` performs:

1. Resolve registered action.
2. Build a side-effect-free `Plan`.
3. Check `CircuitBreaker.is_open(resource)`.
4. Call `permissions.decide(mode, risk_tier, environment, grant)`.
5. Refuse or prompt as required.
6. Execute.
7. If successful, call `verify`.
8. Record actual execution in the breaker.
9. Append an audit record for refusal, skip, success, failure, or input-needed results.

`--dry-run` calls `action.dry_run()` and never executes. The dispatcher now reports refusal honestly in read-only dry runs.

### 7.2 Permission modes

- `read-only` — refuses all writes.
- `ask` — prompts for safe and approval actions.
- `auto-safe` — allows `SAFE`; prompts `APPROVAL`.
- `environment-scoped` — allows `SAFE` outside `production`; prompts in production.
- `bypass` — allows `SAFE`; still prompts approval-tier actions; never overrides `NEVER`.

`NEVER` always refuses. The current registered action set does not visibly include a `NEVER` action, but the engine supports the tier.

### 7.3 Registered actions

| Action IDs | Tier | Reversible / side effect summary |
|---|---|---|
| `open-pr`, `apply-ci-fix`, `apply-manifest-fix`, `apply-gitlab-ci-fix`, `request-secret`, `restart-pod`, `datadog-mute-monitor`, `grafana-silence-alert`, `terraform-init`, `vercel-redeploy` | SAFE | Writes PR/MR or local secret, restarts/silences/mutes, initializes Terraform, or redeploys; several are marked reversible, but “SAFE” means permission behavior, not no side effect |
| `pagerduty-acknowledge` | SAFE | Changes incident acknowledgement; marked non-reversible in its spec |
| `rollback`, `scale`, `edit-configmap`, `edit-secret`, `exec`, `execute-aws`, `execute-gcp`, `execute-azure`, `pagerduty-resolve`, `pagerduty-page`, `vercel-rollback`, `datadog-alert`, `aws-alert`, `gcp-alert`, `github-open-issue`, `gitlab-open-issue`, `snyk-ignore-issue`, `terraform-apply` | APPROVAL | External writes, execution, paging, issue creation, secret/config changes, or infrastructure apply |
| `gitleaks-escalate` | SAFE in its spec | Runs local scan and can create a PagerDuty incident; marked non-reversible despite external escalation |

`edit-secret` explicitly avoids printing values; Gitleaks tests assert leaked secret text does not reach state/log output. The full exact specs are in the individual action modules and `tests/test_actions.py`.

### 7.4 Circuit breaker

Default: five actions per resource in 60 seconds, persisted at `.prash/circuit.json`, configurable via `PRASH_CIRCUIT_MAX_ACTIONS`, `PRASH_CIRCUIT_WINDOW_SECONDS`, and `PRASH_CIRCUIT_STATE_PATH`. Reset is explicit through `prash circuit reset`. The breaker is resource-keyed, not a full provider/account-wide safety budget.

### 7.5 Audit record

`AuditLog.append()` writes JSON lines to `.prash/audit.log` with:

```text
id, ts, seq, action, risk_tier, mode, decision, environment, actor,
status, summary, verification_ok, extra
```

It is append-only by API design and used by `/api/activity` and `prash audit`. Chat execution also appends a separate audit-shaped record with actor `chat_copilot`, meaning a single chat-triggered action can have both dispatcher and chat-level records.

---

## 8. AI, diagnosis, and remediation flow

### 8.1 Model clients

`prash/brain/kimi_client.py` uses lazy AsyncOpenAI clients for DeepSeek and Kimi. It reads API keys/models from environment, recreates clients when the running event loop changes, supports structured tool calls, JSON extraction, recoverable/transient error handling, retries, and provider fallback. No model call log is persisted to a hosted database; it is logged locally.

`.env.example` also lists Gemini, while current active chat/diagnosis paths emphasize DeepSeek/Kimi/OpenAI-compatible clients. The settings UI advertises additional models that are not all wired to distinct client implementations.

### 8.2 Diagnosis schema

`Diagnosis` includes problem summary, root cause, fix description, fix type, confidence, flaky flag, file changes, category, truncation warning, speculative flag, required secrets, recommended runtime/monitoring action, ConfigMap patch, and ranked options.

`FileChange` enforces either exact/whitespace-tolerant unique edits against existing content or complete new content for genuinely new files. It rejects unsafe paths and oversized content. `Diagnosis` coerces inconsistent fix types and derives `recommended_action` from a default option when an options menu is used.

### 8.3 CI diagnosis

1. GitHub/GitLab connector fetches workflow/pipeline logs.
2. Log fetchers cap ZIP/extracted/log sizes, prioritize failing jobs, filter error lines with context, preserve matrix summaries, and avoid phantom raw-tail sections for multi-failure splitting.
3. `multi_diagnosis.py` splits independent job sections and diagnoses them concurrently with a configurable cap (`PRASH_MAX_DIAGNOSIS_CONCURRENCY`, default 3).
4. Partial results are represented as “diagnosed X of N,” not falsely “fixed.”
5. GitHub diagnosis grounds proposed edits against real repository contents and can attempt edit repair.
6. CLI rendering presents diagnoses/options; applying CI/manifest fixes runs through action dispatch and can reconcile a new branch CI run with bounded attempts and error signatures.

### 8.4 Runtime/monitoring diagnosis

`fix.py` gathers provider context from Kubernetes, AWS, GCP, Datadog, PagerDuty, or Grafana and calls the same diagnosis brain. Automatic mapping currently includes:

```text
restart_pod          → restart-pod
edit_configmap       → edit-configmap
action mute_monitor  → datadog-mute-monitor
acknowledge_incident → pagerduty-acknowledge
silence_alert        → grafana-silence-alert
```

Rollback and scale are recognized by the schema but are intentionally not always auto-dispatched through the runtime seam. The diagnosis prompt distinguishes transient/wedged pods from deterministic broken image/config/command cases and allows an explicit no-action escalation or ranked options.

### 8.5 Correlation and memory

- `correlation.py` clusters normalized events by a two-minute running-gap window, records distinct connectors, and formats multi-source context.
- `local_memory.py` stores verified fixes, repeated error signatures, category outcomes, known-good files, flaky tests, and dependency patterns in `.prash/memory.json` (or `PRASH_MEMORY_PATH`).
- `repo_memory.py` is only a local dataclass/prompt renderer. The old Supabase-backed builder was intentionally removed.
- `save_fix()` is called from fix flows after verification to learn from outcomes.

### 8.6 Confirmed model/brain caveats

- `diagnosis_agent.py` has a historical docstring describing itself as CI-shaped/unchanged even though the file and schema now include runtime/monitoring context paths; documentation is stale.
- Some server Copilot paths contain highly specific production assumptions about AWS EKS, `lear-demo`, `ap-south-1`, checkout services, and expected CPU ranges. These should not be treated as generic live telemetry behavior.
- Chat fallback strings claim stable AWS/Kubernetes telemetry even when model/provider calls fail; this is a product honesty concern distinct from the connector metrics API.

---

## 9. Watcher and notification architecture

### 9.1 CLI watcher

`prash watcher.py` supports:

- Kubernetes pod problem transitions (`CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, stuck-pending) with deduped notifications.
- Terraform state/drift polling.
- Shared connector watch-handle loops for Datadog, PagerDuty, Grafana, GitHub, and GitLab.
- Shared interface-driven parallel polling for AWS/GCP and other connectors with `poll_state + get_stats`.

Each new event can go to OS desktop notification, console, and configured team channels. Desktop notification failures are intentionally non-fatal. Channels are Slack webhook, Discord webhook, SMTP email, and Twilio WhatsApp in `notifications.py`; incident-specific rich email and Slack Block Kit paths live separately.

### 9.2 Server watcher

The FastAPI server has a distinct in-process watcher implementation around connector `WatchHandle`s and a one-second scheduler. It persists active watch metadata to YAML but stores actual handle objects and failure counters in memory. It creates notification records and WebSocket payloads; it does not reuse the CLI watcher’s dedup model in a single shared state object.

### 9.3 Duplicate/overlapping models

There are at least four watcher/event consumers in the desktop and two watcher implementations in Python. This is functional but increases the chance of duplicate polls, duplicate sockets, inconsistent deduplication, and inconsistent status semantics.

---

## 10. Incidents, chat sessions, email, and Slack

### 10.1 Incident state

`incident_manager.py` loads/saves `.prash/incidents/incidents.json` and keeps `_INCIDENTS` in memory. An incident record includes:

```text
incident_id, title, service, namespace, severity, status,
resolution_status, cluster, tags, error_summary, diagnosis,
proposed_remediation, patch_data, agent_thinking, requires_approval,
episodic_memory, created/resolved timestamps, conversation[]
```

The lifecycle is ACTIVE/INVESTIGATING → RESOLVED/RECOVERED after `kubectl` remediation, or DENIED/ESCALATED_TO_HUMAN after denial. `approve_incident()` records the approver and immediately invokes `execute_remediation()`; `deny_incident()` changes state and appends conversation messages.

The remediation subprocess currently patches a service ConfigMap, optionally scales a PostgreSQL deployment, deletes matching service pods, and then records a success message claiming health verification. The shown code does not itself perform the stated HTTP health probe before setting RESOLVED; this is a confirmed honesty/verification gap.

### 10.2 Chat session state

`chat_manager.py` persists:

- `.prash/chat_sessions/sessions_index.json`
- `.prash/chat_sessions/session_{id}.json`

Sessions contain origin, service, incident linkage, status, timestamps, messages, and attachments. Channel session reuse is based on origin and recent activity; sender identity is recorded in message text/metadata but session lookup does not strictly key by sender.

### 10.3 Email

`email_service.py` generates escaped HTML incident and Copilot emails, archives `.prash/emails/latest.html` and per-incident HTML, attempts SMTP, and records in-memory dispatched email metadata. It embeds localhost chat/approval URLs by default and has default recipient/sender fallbacks in source. `conversation_router.py` runs an IMAP Gmail listener in a daemon thread, processes up to the last five unseen messages, authorizes by configured sender/incident subject heuristics, and routes replies.

### 10.4 Slack

`slack_service.py` emits Block Kit/webhook messages with war-room, approve, and deny URLs. `conversation_router.py` routes Slack commands/events/interactivity to channel-isolated sessions and responses. Slack status/commands are exposed through unauthenticated server routes; the actual webhook URLs are local credentials.

### 10.5 Demo-specific fixed narrative

`conversation_router.py` contains a large triage-report generator with fixed host names, CPU/memory/latency values, service names, EKS details, and a 98.4% health score. `server.py`, `email_service.py`, `incident_manager.py`, and Slack code also contain default `checkout-api`/`lear-demo`/`ap-south-1` assumptions. These are appropriate for a demo path only, but several are reachable from shared Copilot/channel infrastructure and conflict with “all data dynamic” claims.

---

## 11. Persistence, configuration, and database/storage behavior

### 11.1 No database

There is no SQL schema, migration, ORM model, database client, or database file used as the application’s persistence layer. The only database-like behaviors are:

- demo incident names and a demo “database failure/failover” narrative;
- provider APIs such as CloudWatch/Datadog/PagerDuty;
- external Kubernetes/Cloud/Terraform state.

### 11.2 Local stores

| Store | Default path | Owner/use | Lifetime/notes |
|---|---|---|---|
| Credentials | root `.env` for FastAPI; `.env` or `~/.prash/.env` for `CredentialStore` | connector keys, model keys, notification secrets, behavior settings | ignored; connect validates before atomic rewrite; CLI may append secrets |
| Project/watch/config YAML | root `prash.yaml` | projects, environments, services, active watches, settings, widget layouts | currently tracked in this checkout despite ignore rule; rewritten with `yaml.safe_dump` |
| Action audit | `.prash/audit.log` or `PRASH_AUDIT_LOG_PATH` | append-only JSONL actions/results | local, sequence scans file from end; no retention enforcement |
| Circuit breaker | `.prash/circuit.json` or `PRASH_CIRCUIT_STATE_PATH` | resource → recent action timestamps | local persisted safety cap |
| Episodic memory | `.prash/memory.json` or `PRASH_MEMORY_PATH` | verified fixes/signatures/outcomes | local JSON; malformed/missing store degrades empty |
| Notifications | `.prash/notifications.json` | max 100 notifications/read state | loaded at server lifespan; persisted on updates |
| Incidents | `.prash/incidents/incidents.json` | incident records/conversation | module creates directory on import; in-memory cache reloads from disk |
| Chat | `.prash/chat_sessions/sessions_index.json` + `session_*.json` | session index and full conversations | module creates directory on import; JSON rewrite per message |
| Email previews | `.prash/emails/latest.html`, `{incident}.html` | rendered incident/Copilot email | module creates directory; in-memory dispatch list capped separately |
| Browser selection | localStorage keys `lear_active_project_id`, `lear_active_environment` | active project/environment | browser-local, not shared with backend |
| Server process state | Python globals | watches, handles, clients, activity, connections, caches | lost on process restart except selected YAML/JSON stores |

### 11.3 Configuration key families

- Models: `DEEPSEEK_API_KEY/MODEL/BASE_URL`, `KIMI_API_KEY/MODEL/BASE_URL`, `GEMINI_API_KEY`, `PRIMARY_MODEL`.
- Cloud: AWS, Azure, GCP, Kubernetes, Vercel keys.
- CI/observability/security: GitHub, GitLab, Datadog, Grafana, PagerDuty, Snyk, Gitleaks, Terraform.
- Safety: `PRASH_PERMISSION_MODE`, `PRASH_ENVIRONMENT`, circuit settings, audit path, max diagnosis concurrency.
- Watcher: interval and provider target lists.
- Channels: Slack/Discord webhooks, SMTP, Twilio/WhatsApp.
- Desktop settings: poll interval, retention, desktop notifications, degraded alerts.

### 11.4 Persistence risks

- `prash.yaml` contains project/watch/layout/settings state and is broadly rewritten by unrelated features; concurrent writers are not centrally serialized.
- Incidents/chat JSON files are read-modify-written without a cross-process lock.
- Runtime global state is not an `AppState` object and is not consistently protected for concurrent request/thread access.
- `.prash` is ignored, but directories are eagerly created by imports, so merely importing certain modules mutates the working directory.
- `retention_days` is stored/read as a setting but no broad retention cleanup behavior was found in the core stores.

---

## 12. Infrastructure, packaging, and verification

### 12.1 Packaging

`pyproject.toml` requires Python >=3.10 and declares Rich, Kubernetes, Pydantic, HTTPX, OpenAI, Plyer, Textual, boto3, Google auth/API clients, FastAPI/Uvicorn, dotenv, and PyYAML. Optional `desktop` and `dev` extras cover FastAPI server and pytest.

The frontend package uses React 19, React Router 7, Radix UI, Framer Motion, Lucide, Vite 7, TypeScript 5.8, Vitest 5, React Testing Library, Tailwind/PostCSS, and Tauri 2 packages.

### 12.2 CI

The Python CI runs on Ubuntu, Windows, and macOS, installs `.[dev]`, runs Ruff non-blocking, then pytest. Separate jobs run kind-based Kubernetes live tests and a Docker WireMock Grafana integration. The intentionally broken fixture runs only on branch `prash-broken-ci`.

### 12.3 Tauri limitations

The Rust shell currently only provides `greet`. Tauri configuration has `csp: null`, opens `localhost:1420`, and includes only default core/opener permissions. Backend process supervision is done by external Windows scripts, not the Tauri shell.

### 12.4 Safe validation performed

- `python -m compileall -q prash evals scripts tests` — **passed**.
- `pytest -q` — **not run successfully**; `pytest` is not installed in the audit environment (`/bin/bash: pytest: command not found`).
- Frontend build/tests — **not run**; `desktop/node_modules` is absent. No dependency installation was performed.
- No live cloud/Kubernetes/SMTP/IMAP/Slack/Twilio/model validation was attempted.
- No server was started and no external side effects were triggered.

---

## 13. Confirmed discrepancy, risk, and unfinished-work register

### 13.1 Confirmed implementation issues

1. **Frontend configured-count contract mismatch.** `LearContext` filters `/api/connectors` results by `c.configured`; `registry_to_json()` returns `status: configured/unconfigured` and does not add a boolean `configured`. `Integrations.tsx` correctly checks `status`, but context-level `connectedCount` can be wrong.
2. **Permissive CORS.** FastAPI uses wildcard origins with credentials.
3. **Unauthenticated WebSocket.** `/ws/events` accepts any connection and controls pause/resume/stop by watch ID.
4. **Unauthenticated incident mutation.** Approval/denial routes accept GET and POST and directly mutate/execute remediation.
5. **Settings secret response.** `get_settings()` returns configured Slack/Discord/PagerDuty values through its settings payload rather than the registry’s masking helper.
6. **Loopback-only runtime assumptions.** Root scripts, Vite proxy, WebSocket hook, email links, Slack links, and server `__main__` use localhost/127.0.0.1. This is unsuitable for a proxied browser/live preview without a relative/proxy-aware WebSocket and host binding.
7. **Multiple WebSocket connections.** Context, notifications, watcher, and service widgets each create independent sockets.
8. **Project delete watch cleanup is mismatched.** Project deletion looks for watch IDs beginning with `project_id`, while server watch IDs are `{connector}:{target}`; project deletion therefore does not reliably stop related watches.
9. **Incident “verified” claims exceed shown verification.** `execute_remediation()` sets RESOLVED and records HTTP 200/1-of-1 language after subprocess patch/delete operations, without a visible health probe in that function.
10. **Chat action execution is a high-privilege endpoint.** It parses model/user-provided argv and invokes CLI functions in-process; the endpoint substitutes an always-approve ask implementation for the request.
11. **Hardcoded/demo narrative crosses shared paths.** Triage reports, Copilot prompts/fallbacks, incident defaults, and channel HTML contain fixed infrastructure names/metrics/URLs.
12. **Resource discovery is incomplete by provider.** Registry has 13 connectors, but server resource discovery has explicit cases for only a subset.
13. **Status detail field mismatch.** Project status uses `ResourceState.message`, while the base dataclass exposes `detail`; user-facing detail can be empty.
14. **Optional-only connector configured semantics.** `is_connector_configured()` requires any env key for Gitleaks/Terraform/Kubernetes optional-only schemas; a documented executable-on-PATH/default-only setup can appear unconfigured.
15. **Dashboard/UI text remains partly hardcoded.** Dynamic connector counts coexist with provider-specific fixed summaries and status language.

### 13.2 Security and production work explicitly still planned

The Week 1 roadmap lists these as not started: typed Pydantic request models for all bodies/queries, rate limiting, CORS lockdown, credential expiry, full secrets audit/pre-commit, security headers, WebSocket authentication, server/CLI decomposition, shared state extraction, API versioning, UI token/library work, test coverage baseline, endpoint contract tests, and broader frontend tests.

These are not assumptions about missing code; they are explicitly listed in `tasks/week1/README.md` and `ROADMAP.md` as future work.

### 13.3 Documentation/spec drift

- `LEAR_HANDOFF.md` and desktop task specs describe many features as completed, while `ROADMAP.md` lists a production-hardening layer as not started.
- `diagnosis_agent.py`’s module documentation describes a CI-only/unchanged brain even though runtime/monitoring schema/context code is present.
- README, `.env.example`, registry metadata, and current provider/action registration have some stale labels (for example older “AWS read-only” wording adjacent to execute support).
- Root and desktop package/build conventions are split between browser/Tauri development and loopback-only launcher behavior.

---

## 14. Recommended next conversation step

No implementation change was made because the requested audit was explicitly a prerequisite. The owner should now specify:

1. **What change is wanted?** For example: a bug fix, security hardening, new connector/action, UI redesign, deployment/preview support, database/storage redesign, or behavior change.
2. **What is the desired scope?** One file, one feature, a production-hardening tranche, or an end-to-end redesign.
3. **What must remain unchanged?** Local-first credential posture, existing API compatibility, demo surfaces, CLI behavior, action safety semantics, or current UI design.
4. **What are the acceptance criteria?** Expected API/UI behavior, tests, security constraints, migration/backward-compatibility needs, and whether live-provider validation is allowed.
5. **What should happen next?** Implement directly on this fixed branch, produce a plan first, add tests first, or open a review/PR after implementation.

---

## 15. Path-by-path inventory

The following list is the complete tracked-file inventory captured for this audit, grouped by the repository’s top-level areas. Binary/icon files are listed because they are part of the application package; their pixels were not semantically audited as source code.

### Root and configuration

See the exact inventory below; the list is intentionally path-oriented so any file can be located without relying on a high-level summary.

### Full inventory

./LEAR_AUDIT_REPORT.md
./LEAR_AUDIT_SUPERPROMPT.md
./.env.example
./.github/workflows/always-broken.yml
./.github/workflows/ci.yml
./.gitignore
./CHANGELOG.md
./CONNECTOR_REWRITE_SPEC.md
./E2E_TEST_CHECKLIST.md
./LEAR_HANDOFF.md
./LICENSE
./PRASH_V2.md
./README.md
./ROADMAP.md
./TESTING_CHECKLIST.md
./TESTING_SETUP.md
./agent_guidelines.md
./desktop/.gitignore
./desktop/README.md
./desktop/index.html
./desktop/package-lock.json
./desktop/package.json
./desktop/postcss.config.js
./desktop/public/favicon.ico
./desktop/public/favicon.png
./desktop/public/favicon.webp
./desktop/public/lear-mark-black.B90cZWlm_Z7lBqg.webp
./desktop/public/tauri.svg
./desktop/public/vite.svg
./desktop/src-tauri/.gitignore
./desktop/src-tauri/Cargo.lock
./desktop/src-tauri/Cargo.toml
./desktop/src-tauri/build.rs
./desktop/src-tauri/capabilities/default.json
./desktop/src-tauri/icons/128x128.png
./desktop/src-tauri/icons/128x128@2x.png
./desktop/src-tauri/icons/32x32.png
./desktop/src-tauri/icons/Square107x107Logo.png
./desktop/src-tauri/icons/Square142x142Logo.png
./desktop/src-tauri/icons/Square150x150Logo.png
./desktop/src-tauri/icons/Square284x284Logo.png
./desktop/src-tauri/icons/Square30x30Logo.png
./desktop/src-tauri/icons/Square310x310Logo.png
./desktop/src-tauri/icons/Square44x44Logo.png
./desktop/src-tauri/icons/Square71x71Logo.png
./desktop/src-tauri/icons/Square89x89Logo.png
./desktop/src-tauri/icons/StoreLogo.png
./desktop/src-tauri/icons/icon.icns
./desktop/src-tauri/icons/icon.ico
./desktop/src-tauri/icons/icon.png
./desktop/src-tauri/src/lib.rs
./desktop/src-tauri/src/main.rs
./desktop/src-tauri/tauri.conf.json
./desktop/src/App.tsx
./desktop/src/__tests__/ConnectorForm.test.tsx
./desktop/src/__tests__/Integrations.test.tsx
./desktop/src/assets/react.svg
./desktop/src/components/ActivityLog.tsx
./desktop/src/components/ChatMessage.tsx
./desktop/src/components/ChatWorkspace.tsx
./desktop/src/components/Chatbot.tsx
./desktop/src/components/ConnectorForm.tsx
./desktop/src/components/Dashboard.tsx
./desktop/src/components/ErrorBoundary.tsx
./desktop/src/components/Integrations.tsx
./desktop/src/components/ModesUI.tsx
./desktop/src/components/NotificationToast.tsx
./desktop/src/components/Notifications.tsx
./desktop/src/components/PrashWindow.tsx
./desktop/src/components/ProjectCreate.tsx
./desktop/src/components/ProjectDetail.tsx
./desktop/src/components/Projects.tsx
./desktop/src/components/ServiceWidget.tsx
./desktop/src/components/Settings.tsx
./desktop/src/components/Sidebar.tsx
./desktop/src/components/StepCredentials.tsx
./desktop/src/components/StepIntegration.tsx
./desktop/src/components/StepLocation.tsx
./desktop/src/components/StepNotifications.tsx
./desktop/src/components/WatcherPanel.tsx
./desktop/src/components/WidgetConfigurator.tsx
./desktop/src/components/Wizard.tsx
./desktop/src/components/widgets/BarChart.tsx
./desktop/src/components/widgets/EventTimeline.tsx
./desktop/src/components/widgets/MetricCard.tsx
./desktop/src/components/widgets/MetricGauge.tsx
./desktop/src/components/widgets/MetricLineChart.tsx
./desktop/src/components/widgets/StatusGrid.tsx
./desktop/src/context/LearContext.tsx
./desktop/src/hooks/useConnectorStatus.ts
./desktop/src/hooks/useNotifications.ts
./desktop/src/hooks/useWatcher.ts
./desktop/src/hooks/useWebSocket.ts
./desktop/src/index.css
./desktop/src/main.tsx
./desktop/src/test/setup.ts
./desktop/src/vite-env.d.ts
./desktop/tailwind.config.js
./desktop/tsconfig.json
./desktop/tsconfig.node.json
./desktop/vite.config.ts
./evals/.gitignore
./evals/README.md
./evals/__init__.py
./evals/cases/monitoring_datadog_alert.json
./evals/cases/monitoring_pagerduty_incident.json
./evals/cases/runtime_crashloop_broken_image.json
./evals/cases/runtime_crashloop_wedged.json
./evals/cases/runtime_image_pull_back_off.json
./evals/cases/runtime_oomkilled.json
./evals/cases/verified_08a2cac8.json
./evals/cases/verified_25fb07ce.json
./evals/cases/verified_3149990d.json
./evals/cases/verified_31ffc8f9.json
./evals/cases/verified_3f94a260.json
./evals/cases/verified_3ff7a5d5.json
./evals/cases/verified_4ba89a5a.json
./evals/cases/verified_8c20908e.json
./evals/cases/verified_93aab4c2.json
./evals/cases/verified_aa690446.json
./evals/cases/verified_ba7ed093.json
./evals/cases/verified_c166d3f8.json
./evals/cases/verified_c8c5b69d.json
./evals/cases/verified_f801bff9.json
./evals/cases/verified_workflow_config_static_site.json
./evals/results/2026-08-09-post-track-d-port.json
./evals/results/2026-08-09-pre-track-d-port-baseline.json
./evals/results/2026-08-09-track-d-days-6-8-k8s.json
./evals/run_eval.py
./evals/score.py
./infrastructure/.terraform.lock.hcl
./infrastructure/main.tf
./infrastructure/outputs.tf
./infrastructure/terraform.tf
./infrastructure/variables.tf
./lear-mark-black.B90cZWlm_Z7lBqg.webp
./lear.bat
./lear.ps1
./package.json
./prash.yaml
./prash/__init__.py
./prash/actions/__init__.py
./prash/actions/alert_base.py
./prash/actions/apply_ci_fix.py
./prash/actions/apply_gitlab_ci_fix.py
./prash/actions/aws_alert.py
./prash/actions/contract.py
./prash/actions/datadog_alert.py
./prash/actions/datadog_mute.py
./prash/actions/edit_config.py
./prash/actions/exec_command.py
./prash/actions/execute_aws.py
./prash/actions/execute_azure.py
./prash/actions/execute_gcp.py
./prash/actions/gcp_alert.py
./prash/actions/github_alert.py
./prash/actions/gitlab_alert.py
./prash/actions/gitleaks_escalate.py
./prash/actions/grafana_silence.py
./prash/actions/missing_secret.py
./prash/actions/open_pr.py
./prash/actions/pagerduty_incident.py
./prash/actions/pagerduty_page.py
./prash/actions/restart_pod.py
./prash/actions/rollback.py
./prash/actions/scale.py
./prash/actions/snyk_ignore.py
./prash/actions/terraform_apply.py
./prash/actions/terraform_init.py
./prash/actions/vercel_deploy.py
./prash/admin_panel.py
./prash/audit.py
./prash/brain/__init__.py
./prash/brain/correlation.py
./prash/brain/diagnosis_agent.py
./prash/brain/edit_repair.py
./prash/brain/gitlab_log_fetcher.py
./prash/brain/kimi_client.py
./prash/brain/local_memory.py
./prash/brain/log_fetcher.py
./prash/brain/multi_diagnosis.py
./prash/brain/repo_memory.py
./prash/brain/schemas.py
./prash/chat_manager.py
./prash/circuit_breaker.py
./prash/cli.py
./prash/connector_registry.py
./prash/connectors/__init__.py
./prash/connectors/aws.py
./prash/connectors/azure.py
./prash/connectors/base.py
./prash/connectors/datadog.py
./prash/connectors/gcp.py
./prash/connectors/github.py
./prash/connectors/gitlab.py
./prash/connectors/gitleaks.py
./prash/connectors/grafana.py
./prash/connectors/kubernetes.py
./prash/connectors/pagerduty.py
./prash/connectors/snyk.py
./prash/connectors/terraform.py
./prash/connectors/testdata/broken-pod.yaml
./prash/connectors/testdata/configmap-mismatch.yaml
./prash/connectors/testdata/oom-limit.yaml
./prash/connectors/testdata/sample-app.yaml
./prash/connectors/testdata/silent-crash-pod.yaml
./prash/connectors/testdata/wedged-pod.yaml
./prash/connectors/vercel.py
./prash/conversation_router.py
./prash/credentials.py
./prash/customer_store.py
./prash/demo_page.py
./prash/dispatch.py
./prash/email_service.py
./prash/fix.py
./prash/incident_manager.py
./prash/incident_page.py
./prash/intent.py
./prash/notifications.py
./prash/permissions.py
./prash/repl.py
./prash/server.py
./prash/setup.py
./prash/slack_service.py
./prash/tui.py
./prash/ui.py
./prash/watcher.py
./prash/widget_generator.py
./pyproject.toml
./scratch_test_patch.py
./scripts/demo/aws_token.py
./scripts/demo/cluster.yaml
./scripts/demo/create-cluster.ps1
./scripts/demo/manifests/checkout-api.yaml
./scripts/demo/manifests/frontend.yaml
./scripts/demo/manifests/loadgen.yaml
./scripts/demo/manifests/namespace.yaml
./scripts/demo/manifests/payment-service.yaml
./scripts/demo/manifests/postgres-replica.yaml
./scripts/demo/manifests/postgres.yaml
./scripts/demo/manifests/shipping-service.yaml
./scripts/demo/reset-memory.ps1
./scripts/demo/scripts/deploy.ps1
./scripts/demo/scripts/deploy.sh
./scripts/demo/scripts/inject-configmap-break.ps1
./scripts/demo/scripts/inject-configmap-break.sh
./scripts/demo/scripts/inject-failure.ps1
./scripts/demo/scripts/inject-failure.sh
./scripts/demo/scripts/inject-oomkill.ps1
./scripts/demo/scripts/load_test.py
./scripts/demo/scripts/reset-configmap.ps1
./scripts/demo/scripts/reset-configmap.sh
./scripts/demo/scripts/reset-demo.ps1
./scripts/demo/scripts/reset-demo.sh
./scripts/demo/scripts/reset-oomkill.ps1
./scripts/demo/scripts/setup_datadog_monitor.py
./scripts/demo/scripts/start-load.ps1
./scripts/demo/scripts/start-load.sh
./scripts/demo/scripts/stop-load.ps1
./scripts/demo/scripts/stop-load.sh
./scripts/demo/seed-memory.ps1
./scripts/demo/seed-memory.sh
./scripts/demo/services/checkout-api/Dockerfile
./scripts/demo/services/checkout-api/app.py
./scripts/demo/services/frontend/Dockerfile
./scripts/demo/services/frontend/index.html
./scripts/demo/services/frontend/nginx.conf
./scripts/demo/services/payment-service/Dockerfile
./scripts/demo/services/payment-service/server.js
./scripts/demo/services/shipping-service/Dockerfile
./scripts/demo/services/shipping-service/app.py
./scripts/demo/teardown-infra.ps1
./scripts/demo/test_k8s_connection.py
./scripts/testing/break_aws.py
./scripts/testing/break_combined.py
./scripts/testing/break_datadog.py
./scripts/testing/break_gcp.py
./scripts/testing/break_grafana.py
./scripts/testing/break_pagerduty.py
./scripts/testing/break_snyk.py
./scripts/testing/break_vercel.py
./scripts/testing/k8s-recovery-demo/README.md
./scripts/testing/k8s-recovery-demo/app/Dockerfile
./scripts/testing/k8s-recovery-demo/app/app.py
./scripts/testing/k8s-recovery-demo/inject-failure.sh
./scripts/testing/k8s-recovery-demo/manifests/checkout-api.yaml
./scripts/testing/k8s-recovery-demo/manifests/configmap.yaml
./scripts/testing/k8s-recovery-demo/manifests/postgres.yaml
./scripts/testing/k8s-recovery-demo/reset-demo.sh
./scripts/testing/k8s-recovery-demo/setup-demo.sh
./scripts/testing/measure_correlation_accuracy.py
./scripts/verify_aws_live.py
./tasks/.demo/00_DEMO_OVERVIEW.md
./tasks/.demo/01_INFRASTRUCTURE.md
./tasks/.demo/02_MICROSERVICES.md
./tasks/.demo/03_LOAD_GENERATOR.md
./tasks/.demo/04_FAILURE_INJECTION.md
./tasks/.demo/05_EPISODIC_MEMORY.md
./tasks/.demo/06_DEMO_SCRIPT.md
./tasks/.demo/07_DATADOG_SETUP.md
./tasks/00_OVERVIEW.md
./tasks/01_BASE_INTERFACE.md
./tasks/02_DATADOG.md
./tasks/03_GRAFANA.md
./tasks/04_PAGERDUTY.md
./tasks/05_KUBERNETES.md
./tasks/06_AWS.md
./tasks/07_GCP.md
./tasks/MASTER_TASKS.md
./tasks/actions/00_ACTION_CONTRACT.md
./tasks/actions/01_KUBERNETES_ACTIONS.md
./tasks/actions/02_CI_ACTIONS.md
./tasks/actions/03_CLOUD_ACTIONS.md
./tasks/actions/04_MONITORING_ACTIONS.md
./tasks/actions/05_INFRA_ACTIONS.md
./tasks/actions/06_ALERT_ACTIONS.md
./tasks/brain/00_BRAIN_OVERVIEW.md
./tasks/brain/01_DIAGNOSIS_AGENT.md
./tasks/brain/02_MULTI_DIAGNOSIS.md
./tasks/brain/03_CORRELATION.md
./tasks/brain/04_SCHEMAS_AND_MODELS.md
./tasks/cli/00_CLI_OVERVIEW.md
./tasks/cli/01_CLI_COMMANDS.md
./tasks/cli/02_REPL_AND_INTENT.md
./tasks/cli/03_TUI_AND_UI.md
./tasks/connectors/00_BASE_INTERFACE.md
./tasks/connectors/01_KUBERNETES.md
./tasks/connectors/02_DATADOG.md
./tasks/connectors/03_GRAFANA.md
./tasks/connectors/04_PAGERDUTY.md
./tasks/connectors/05_AWS.md
./tasks/connectors/06_GCP.md
./tasks/connectors/07_AZURE.md
./tasks/connectors/08_GITHUB.md
./tasks/connectors/09_GITLAB.md
./tasks/connectors/10_VERCEL.md
./tasks/connectors/11_SNYK.md
./tasks/connectors/12_GITLEAKS.md
./tasks/connectors/13_TERRAFORM.md
./tasks/cross-cutting/00_PERMISSIONS.md
./tasks/cross-cutting/01_AUDIT_LOG.md
./tasks/cross-cutting/02_CIRCUIT_BREAKER.md
./tasks/cross-cutting/03_CREDENTIALS.md
./tasks/desktop/00_DESKTOP_OVERVIEW.md
./tasks/desktop/completed/01_BACKEND_API_BRIDGE.md
./tasks/desktop/completed/02_CONNECTOR_REGISTRY.md
./tasks/desktop/completed/03_DESIGN_SYSTEM.md
./tasks/desktop/completed/04_ONBOARDING_WIZARD.md
./tasks/desktop/completed/05_SERVICE_CONNECTIONS.md
./tasks/desktop/completed/06_SIDEBAR_NAVIGATION.md
./tasks/desktop/completed/07_PROJECT_SYSTEM.md
./tasks/desktop/completed/08_METRIC_WIDGETS.md
./tasks/desktop/completed/09_WATCHER_STATUS.md
./tasks/desktop/completed/10_AI_CHATBOX.md
./tasks/desktop/completed/11_DASHBOARD_OVERVIEW.md
./tasks/desktop/completed/12_INTEGRATIONS_PAGE.md
./tasks/desktop/completed/13_ACTIVITY_LOG.md
./tasks/desktop/completed/14_SETTINGS.md
./tasks/desktop/completed/15_NOTIFICATIONS.md
./tasks/desktop/completed/16_AI_WIDGET_GENERATION.md
./tasks/infra/00_CI_CD.md
./tasks/infra/01_HOSTING.md
./tasks/infra/02_PACKAGING.md
./tasks/questions.md
./tasks/watcher/00_WATCHER_OVERVIEW.md
./tasks/watcher/01_WATCHER_CORE.md
./tasks/watcher/02_NOTIFICATIONS.md
./tasks/week1/README.md
./tasks/week1/backend/B-01_SPLIT_SERVER.md
./tasks/week1/backend/B-02_SPLIT_CLI.md
./tasks/week1/backend/B-03_EXTRACT_STATE.md
./tasks/week1/backend/B-04_API_VERSIONING.md
./tasks/week1/security/S-01_INPUT_VALIDATION.md
./tasks/week1/security/S-02_RATE_LIMITING.md
./tasks/week1/security/S-03_CORS_LOCKDOWN.md
./tasks/week1/security/S-04_CREDENTIAL_EXPIRY.md
./tasks/week1/security/S-05_SECRETS_AUDIT.md
./tasks/week1/security/S-06_SECURE_HEADERS.md
./tasks/week1/security/S-07_WEBSOCKET_AUTH.md
./tasks/week1/testing/T-01_COVERAGE_BASELINE.md
./tasks/week1/testing/T-02_INTEGRATION_TESTS.md
./tasks/week1/testing/T-03_FRONTEND_TESTS.md
./tasks/week1/ui/U-01_DESIGN_TOKENS.md
./tasks/week1/ui/U-02_SIDEBAR_REDESIGN.md
./tasks/week1/ui/U-03_DASHBOARD_REDESIGN.md
./tasks/week1/ui/U-04_COMPONENT_LIBRARY.md
./test.env
./tests/integration/test_grafana_mock_service.py
./tests/test_actions.py
./tests/test_audit.py
./tests/test_aws_connector.py
./tests/test_azure_connector.py
./tests/test_brain_datadog_context.py
./tests/test_brain_edit_repair.py
./tests/test_brain_grafana_context.py
./tests/test_brain_k8s_context.py
./tests/test_brain_kimi_client.py
./tests/test_brain_local_memory.py
./tests/test_brain_multi_diagnosis.py
./tests/test_brain_pagerduty_context.py
./tests/test_brain_schemas.py
./tests/test_circuit_breaker.py
./tests/test_cli.py
./tests/test_connector_registry.py
./tests/test_correlation.py
./tests/test_datadog_connector.py
./tests/test_desktop_api.py
./tests/test_eval_env_loading.py
./tests/test_fix.py
./tests/test_gcp_connector.py
./tests/test_github_connector.py
./tests/test_gitlab_connector.py
./tests/test_gitleaks_connector.py
./tests/test_grafana_connector.py
./tests/test_intent.py
./tests/test_kubernetes_connector.py
./tests/test_kubernetes_connector_live.py
./tests/test_log_fetcher.py
./tests/test_notifications.py
./tests/test_pagerduty_connector.py
./tests/test_permissions.py
./tests/test_repl.py
./tests/test_service_connections.py
./tests/test_snyk_connector.py
./tests/test_terraform_connector.py
./tests/test_tui.py
./tests/test_vercel_connector.py
./tests/test_watcher.py
./tests/test_widget_generation.py
./tf_test_dir/main.tf
./uv.lock

---

## 16. Deep line-level system-design context

The owner requested a second completeness pass so a future implementation agent can understand the system before changing it. The companion artifact [`LEAR_FULL_SYSTEM_CONTEXT.md`](./LEAR_FULL_SYSTEM_CONTEXT.md) is the exhaustive source map for that purpose.

This pass is deliberately more mechanical than the narrative audit:

- It enumerates every tracked file in the checkout, including documentation, specifications, tests, fixtures, scripts, infrastructure, lockfiles, desktop assets, and the audit artifacts.
- For every UTF-8 text file it records byte size, SHA-256, total line count, structural symbols with exact start/end line ranges, detected routes/hooks/actions/network/process/persistence markers, and then emits every source line with its original 1-based line number.
- For binary files it records the path, byte size, SHA-256, and binary classification instead of inventing semantic source details.
- It distinguishes implementation, test, fixture, task/specification, documentation, configuration, generated-lock, script, infrastructure, and asset areas so future changes can be scoped against both code and the intended contract.
- Credential-looking fixture values are redacted in the companion context while key names, control flow, tests, and file/line locations remain visible. No live credential or provider call was used.

The line-level context is not a substitute for the repository itself: before any change, the implementation agent must still re-open the exact current source lines and re-check the working tree. The context is a navigation and completeness contract, not permission to assume that prose, task status, or tests are correct.

### Completeness status

- Product implementation was not changed by this audit.
- The context generator scans the current tracked-file set and reports its own excluded self-reference explicitly; the final tracked-file count is recorded in the context header.
- Any future code change invalidates the affected line ranges and hashes; regenerate the context after such a change.
