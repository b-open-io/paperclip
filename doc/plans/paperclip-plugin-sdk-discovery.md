# Paperclip Plugin SDK Discovery

**Date:** 2026-03-14
**Status:** Reference — internal bOpen use only

This document records what was discovered about Paperclip's plugin system during bOpen's evaluation of the Tortuga integration strategy. The source of truth for any single detail remains the Paperclip source and `PLUGIN_SPEC.md`.

---

## 1. Overview

Paperclip has a fully realized plugin system. Plugins run as isolated worker processes, communicate with the host over JSON-RPC, and can inject UI into named slots in the Paperclip frontend. The system is not experimental — it has 21 server-side service files, a published SDK package, four example plugins, and a scaffolding CLI.

The key constraint for bOpen: the SDK is internal to the Paperclip monorepo and not yet published to npm. Plugin dependencies use `file:` tarball paths or `workspace:*` references.

---

## 2. Server-Side Implementation

### 2.1 Service Layer

All plugin services live in `/Users/satchmo/code/paperclip/server/src/services/` and follow a factory-function pattern. The 21 services and their responsibilities:

| Service file | Responsibility |
|---|---|
| `plugin-registry.ts` | CRUD for `plugins` and `plugin_config` tables — the persistence layer |
| `plugin-lifecycle.ts` | State-machine controller: `installed → ready → disabled / error / uninstalled` |
| `plugin-loader.ts` | Resolves installed plugin paths, reads manifests, returns UI contribution metadata |
| `plugin-worker-manager.ts` | Spawns and terminates plugin worker child processes |
| `plugin-job-scheduler.ts` | Cron scheduling for declared jobs |
| `plugin-job-coordinator.ts` | Coordinates job execution across the cluster |
| `plugin-job-store.ts` | Persists job run history and status |
| `plugin-event-bus.ts` | Routes domain events to plugin workers |
| `plugin-capability-validator.ts` | Gate that enforces declared manifest capabilities before each host API call |
| `plugin-config-validator.ts` | Validates operator-supplied config against `instanceConfigSchema` |
| `plugin-tool-dispatcher.ts` | Routes agent tool calls to the correct plugin worker |
| `plugin-tool-registry.ts` | Catalog of tools contributed by installed plugins |
| `plugin-state-store.ts` | Key-value state storage scoped by plugin and entity |
| `plugin-secrets-handler.ts` | Resolves secret references without exposing raw values |
| `plugin-stream-bus.ts` | SSE push channel from worker to plugin UI |
| `plugin-runtime-sandbox.ts` | Node `vm`-based sandbox for loading plugin modules |
| `plugin-manifest-validator.ts` | Schema validation for `PaperclipPluginManifestV1` |
| `plugin-dev-watcher.ts` | File watcher for hot-reload during local plugin development |
| `plugin-host-services.ts` | Assembles the host API surface exposed to each worker |
| `plugin-host-service-cleanup.ts` | Tears down host services on worker shutdown |
| `plugin-log-retention.ts` | Prunes plugin log rows on a schedule |

### 2.2 Plugin Lifecycle State Machine

```
installed --> ready --> disabled
    |            |          |
    |            +--> error-+
    |            |
    |     upgrade_pending
    |            |
    +--------> uninstalled
```

Transitions are enforced by `plugin-lifecycle.ts`. Moving to `ready` starts the worker process; moving out of `ready` stops it gracefully. Workers get 10 seconds (configurable) before SIGTERM then SIGKILL.

### 2.3 REST Routes

Routes live in `/Users/satchmo/code/paperclip/server/src/routes/plugins.ts`. All require board-level authentication.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/plugins` | List plugins, filter by status |
| GET | `/api/plugins/available` | List bundled example plugins |
| POST | `/api/plugins/install` | Install from npm package name or local path |
| POST | `/api/plugins/:id/uninstall` | Soft delete or hard purge |
| POST | `/api/plugins/:id/enable` | Transition to `ready` |
| POST | `/api/plugins/:id/disable` | Transition to `disabled` |
| POST | `/api/plugins/:id/upgrade` | Upgrade to a new version |
| GET | `/api/plugins/:id/health` | Run plugin health check |
| GET | `/api/plugins/ui-slots` | Return all UI slot contributions for the frontend |
| GET | `/api/plugins/:id/agent-tools` | List tools contributed by a plugin |
| POST | `/api/plugins/:id/tools/:toolName/execute` | Execute a plugin tool |
| POST | `/api/plugins/:id/webhooks/:endpointKey` | Deliver an inbound webhook |
| GET | `/api/plugins/:id/bridge/stream/:channel` | SSE stream from worker to UI |
| GET | `/api/plugins/:id/logs` | Retrieve plugin log entries |

Plugin UI bundles are served by a separate route file at `/Users/satchmo/code/paperclip/server/src/routes/plugin-ui-static.ts` under `/_plugins/:pluginId/ui/*`. Only plugins in `ready` status are served. Content-hashed filenames receive one-year immutable cache headers.

---

## 3. SDK Package

**Location:** `/Users/satchmo/code/paperclip/packages/plugins/sdk/`
**Package name:** `@paperclipai/plugin-sdk`
**Import split:** worker code imports from `@paperclipai/plugin-sdk`; UI code imports from `@paperclipai/plugin-sdk/ui`

### 3.1 Worker-Side Exports

```ts
import { definePlugin, runWorker, z } from "@paperclipai/plugin-sdk";
import { createTestHarness } from "@paperclipai/plugin-sdk";
import { createPluginBundlerPresets } from "@paperclipai/plugin-sdk";
import { startPluginDevServer } from "@paperclipai/plugin-sdk";
```

`definePlugin(definition)` — accepts a `PluginDefinition` object and returns a sealed `PaperclipPlugin`. This is the only top-level call a plugin worker needs.

`runWorker(plugin, import.meta.url)` — starts the JSON-RPC host loop. Called after the default export in every worker entrypoint.

`z` — Zod re-exported so plugin authors do not add a separate dependency.

### 3.2 PluginDefinition Lifecycle Hooks

| Hook | Required | Notes |
|---|---|---|
| `setup(ctx)` | Yes | Registers all handlers. Must complete synchronously after resolution — no async callbacks post-setup. |
| `onHealth()` | No | Returns `{ status: "ok" | "degraded" | "error", message?, details? }`. Polled by the host health dashboard. |
| `onConfigChanged(newConfig)` | No | Hot-reload config without restarting. If absent, host restarts the worker instead. |
| `onShutdown()` | No | Graceful shutdown — 10 s deadline before SIGTERM. |
| `onValidateConfig(config)` | No | Called at startup, on config save, and via "Test Connection" button. |
| `onWebhook(input)` | No | Required if webhooks are declared in the manifest. |

### 3.3 PluginContext — Host API Surface

The `ctx` object passed to `setup()` contains these clients, each capability-gated:

| Property | Type | Required capability |
|---|---|---|
| `ctx.manifest` | `PaperclipPluginManifestV1` | — |
| `ctx.config` | `PluginConfigClient` | — |
| `ctx.events` | `PluginEventsClient` | `events.subscribe` / `events.emit` |
| `ctx.jobs` | `PluginJobsClient` | `jobs.schedule` |
| `ctx.launchers` | `PluginLaunchersClient` | — |
| `ctx.http` | `PluginHttpClient` | `http.outbound` |
| `ctx.secrets` | `PluginSecretsClient` | `secrets.read-ref` |
| `ctx.activity` | `PluginActivityClient` | `activity.log.write` |
| `ctx.state` | `PluginStateClient` | `plugin.state.read` / `plugin.state.write` |
| `ctx.entities` | `PluginEntitiesClient` | — |
| `ctx.projects` | `PluginProjectsClient` | `projects.read` / `project.workspaces.read` |
| `ctx.companies` | `PluginCompaniesClient` | `companies.read` |
| `ctx.issues` | `PluginIssuesClient` | `issues.read` / `issues.create` / `issues.update` |
| `ctx.agents` | `PluginAgentsClient` | `agents.read` / `agents.pause` / `agents.resume` / `agents.invoke` |
| `ctx.goals` | `PluginGoalsClient` | `goals.read` / `goals.create` / `goals.update` |
| `ctx.data` | `PluginDataClient` | — |
| `ctx.actions` | `PluginActionsClient` | — |
| `ctx.streams` | `PluginStreamsClient` | — |
| `ctx.tools` | `PluginToolsClient` | `agent.tools.register` |
| `ctx.metrics` | `PluginMetricsClient` | `metrics.write` |
| `ctx.logger` | `PluginLogger` | — |

Notable clients:

**`ctx.state`** — scoped key-value store. Scope kinds: `instance`, `company`, `project`, `project_workspace`, `agent`, `issue`, `goal`, `run`. Key is a five-part composite `(pluginId, scopeKind, scopeId, namespace, stateKey)`. Completely isolated between plugins.

**`ctx.agents.sessions`** — two-way chat with agents. `create()`, `list()`, `sendMessage()` (streaming via `onEvent` callback), `close()`. Requires `agent.sessions.*` capabilities.

**`ctx.streams`** — push real-time events from worker to plugin UI via SSE. `open(channel, companyId)` / `emit(channel, event)` / `close(channel)`. The UI side uses `usePluginStream(channel)`.

**`ctx.tools`** — register handlers for agent tools declared in the manifest. Tool names are auto-namespaced by `pluginId` at runtime.

### 3.4 UI-Side Exports

```ts
import { usePluginData, usePluginAction, useHostContext, usePluginStream, usePluginToast } from "@paperclipai/plugin-sdk/ui";
```

| Hook | Purpose |
|---|---|
| `usePluginData<T>(key, params?)` | Fetches data from `ctx.data.register(key, handler)` in the worker. Returns `{ data, loading, error, refresh }`. |
| `usePluginAction(key)` | Returns an async function that calls `ctx.actions.register(key, handler)` in the worker. |
| `useHostContext()` | Returns `{ companyId, entityId, entityType, userId, ... }` — the Paperclip context the slot is rendering in. |
| `usePluginStream<T>(channel, options?)` | Subscribes to SSE pushed by `ctx.streams.emit(channel, event)`. Returns `{ events, lastEvent, connected, close }`. |
| `usePluginToast()` | Triggers a host toast notification from plugin UI. |

### 3.5 Testing

`createTestHarness({ manifest, capabilities?, config? })` — returns a `TestHarness` with a fully typed in-memory `ctx`. Enforces declared capabilities. No server required.

Harness methods: `seed()`, `setConfig()`, `emit()`, `runJob()`, `getData()`, `performAction()`, `executeTool()`, `getState()`, `simulateSessionEvent()`. Exposes `harness.logs`, `harness.activity`, `harness.metrics` for assertions.

### 3.6 Bundler Presets

`createPluginBundlerPresets(input?)` — returns esbuild and rollup baseline configs for `worker`, `manifest`, and `ui` bundles. Worker targets Node 20 ESM; UI targets browser ES2022 ESM and externalizes `react`, `react-dom`, and `@paperclipai/plugin-sdk/ui`. Presets are plain objects — no plugins included; rollup users add their own.

### 3.7 Dev Server

`startPluginDevServer(options)` — serves `dist/ui/` on localhost with SSE hot-reload events. Default port `4177`. Used via the generated `dev:ui` script: `paperclip-plugin-dev-server --root . --ui-dir dist/ui --port 4177`.

---

## 4. Plugin Manifest (PaperclipPluginManifestV1)

Manifest type exported from `@paperclipai/plugin-sdk`. Full surface from the kitchen-sink example:

### 4.1 Top-Level Fields

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | Stable reverse-DNS style: `"@scope/name"` becomes `"scope.name"` in the manifest ID |
| `apiVersion` | `1` | Currently always `1` |
| `version` | `string` | SemVer |
| `displayName` | `string` | Human-readable name |
| `description` | `string` | |
| `author` | `string` | |
| `categories` | `PluginCategory[]` | `"connector"`, `"workspace"`, `"automation"`, `"ui"` |
| `capabilities` | `PluginCapability[]` | Declarative allowlist — see section 4.2 |
| `entrypoints.worker` | `string` | Relative path to built worker JS (e.g. `"./dist/worker.js"`) |
| `entrypoints.ui` | `string` | Relative path to built UI directory (e.g. `"./dist/ui"`) |
| `instanceConfigSchema` | `JsonSchema` | JSON Schema for operator-facing configuration form |
| `jobs` | `PluginJobDeclaration[]` | Scheduled job declarations |
| `webhooks` | `PluginWebhookDeclaration[]` | Inbound webhook endpoint declarations |
| `tools` | `PluginToolDeclaration[]` | Agent tool declarations |
| `ui.slots` | `PluginUiSlotDeclaration[]` | UI extension slot declarations |
| `ui.launchers` | `PluginLauncherDeclaration[]` | Toolbar button / modal launcher declarations |

### 4.2 Capability Strings

Full list from the kitchen-sink manifest:

```
companies.read
projects.read
project.workspaces.read
issues.read
issues.create
issues.update
issue.comments.read
issue.comments.create
agents.read
agents.pause
agents.resume
agents.invoke
agent.sessions.create
agent.sessions.list
agent.sessions.send
agent.sessions.close
goals.read
goals.create
goals.update
activity.log.write
metrics.write
plugin.state.read
plugin.state.write
events.subscribe
events.emit
jobs.schedule
webhooks.receive
http.outbound
secrets.read-ref
agent.tools.register
instance.settings.register
ui.sidebar.register
ui.page.register
ui.detailTab.register
ui.dashboardWidget.register
ui.commentAnnotation.register
ui.action.register
```

### 4.3 Job Declaration

```ts
{
  jobKey: string;       // stable identifier, matched in ctx.jobs.register(key, fn)
  displayName: string;
  description?: string;
  schedule: string;     // cron expression, e.g. "*/15 * * * *"
}
```

### 4.4 Webhook Declaration

```ts
{
  endpointKey: string;  // matched in onWebhook(input) via input.endpointKey
  displayName: string;
  description?: string;
}
```

Webhook deliveries arrive at `POST /api/plugins/:pluginId/webhooks/:endpointKey`. The plugin receives `{ endpointKey, headers, rawBody, parsedBody?, requestId }` and is responsible for signature verification.

### 4.5 Tool Declaration

```ts
{
  name: string;               // matched in ctx.tools.register(name, decl, fn)
  displayName: string;
  description: string;
  parametersSchema: JsonSchema;
}
```

Tool names are auto-namespaced by `pluginId` at runtime. Tool handlers receive `(params, runCtx)` where `runCtx` contains `{ agentId, runId, companyId, projectId }`.

### 4.6 UI Slot Types

All slot types and which entity types each supports:

| Slot type | entityTypes | Notes |
|---|---|---|
| `page` | — | Full-page route; uses `routePath` field |
| `settingsPage` | — | Plugin settings page in admin UI |
| `dashboardWidget` | — | Main dashboard widget |
| `sidebar` | — | Global sidebar entry |
| `sidebarPanel` | — | Expandable sidebar panel |
| `projectSidebarItem` | `project` | Item under a project in sidebar; supports `order` |
| `detailTab` | `project`, `issue` | Tab on project or issue detail pages; supports `order` |
| `taskDetailView` | `issue` | Full detail view replacement for an issue |
| `toolbarButton` | `project`, `issue` | Button in entity toolbar |
| `contextMenuItem` | `project`, `issue` | Item in entity context menu |
| `commentAnnotation` | `comment` | Renders below each comment |
| `commentContextMenuItem` | `comment` | Item in comment context menu |

### 4.7 Launcher Declaration

Launchers sit in `ui.launchers[]` and wire a toolbar button to a modal or other action:

```ts
{
  id: string;
  displayName: string;
  placementZone: "toolbarButton";    // only zone currently
  entityTypes: string[];
  action: {
    type: "openModal";
    target: string;                  // exportName from the UI module
  };
  render: {
    environment: "hostOverlay";
    bounds: "wide" | "narrow";
  };
}
```

---

## 5. Example Plugins

Four examples ship in the Paperclip monorepo at `packages/plugins/examples/`. Three are registered in the API as installable examples via `GET /api/plugins/available`.

| Example | Package | Purpose |
|---|---|---|
| `plugin-hello-world-example` | `@paperclipai/plugin-hello-world-example` | Minimal. One `dashboardWidget` slot, one capability (`ui.dashboardWidget.register`). Starting point for UI-only plugins. |
| `plugin-file-browser-example` | `@paperclipai/plugin-file-browser-example` | Demonstrates `projectSidebarItem`, `detailTab`, `commentAnnotation`, `commentContextMenuItem`. Uses `project.workspaces.read` to access workspace filesystem paths. Has `instanceConfigSchema`. |
| `plugin-kitchen-sink-example` | `@paperclipai/plugin-kitchen-sink-example` | Comprehensive reference. All 13 slot types, 3 tools, 1 scheduled job, 1 webhook endpoint, 37 capabilities, streaming, state, secrets, agent sessions. |
| `plugin-authoring-smoke-example` | — | Generated output from `create-paperclip-plugin`. Exists to verify the scaffolder produces a working plugin. Not registered in the available examples list. |

---

## 6. Scaffolding CLI

**Location:** `/Users/satchmo/code/paperclip/packages/plugins/create-paperclip-plugin/`

```bash
node packages/plugins/create-paperclip-plugin/src/index.ts <name> \
  [--template default|connector|workspace] \
  [--output <dir>] \
  [--display-name <name>] \
  [--description <text>] \
  [--author <name>] \
  [--category connector|workspace|automation|ui] \
  [--sdk-path <path-to-sdk>]
```

`<name>` must be a valid npm package name (scoped or unscoped, lowercase).

The scaffolder generates a complete plugin project containing:

- `package.json` with `@paperclipai/plugin-sdk` dependency and `paperclipPlugin` metadata block
- `tsconfig.json` targeting ES2022 / NodeNext
- `esbuild.config.mjs` using `createPluginBundlerPresets`
- `rollup.config.mjs` using `createPluginBundlerPresets`
- `vitest.config.ts`
- `src/manifest.ts` — minimal manifest with one `dashboardWidget` slot
- `src/worker.ts` — `definePlugin` with `issue.created` handler, `health` data key, `ping` action key
- `src/ui/index.tsx` — `DashboardWidget` using `usePluginData` and `usePluginAction`
- `tests/plugin.spec.ts` — harness-based test for all three registrations
- `.gitignore`, `README.md`

When run outside the Paperclip monorepo, the scaffolder packs `@paperclipai/plugin-sdk` and `@paperclipai/shared` into `.paperclip-sdk/*.tgz` tarballs using `pnpm pack` and wires them as `file:` dependencies. The `pnpm.overrides` block ensures the shared peer resolves to the same tarball.

The generated `dev:ui` script runs the SDK's built-in dev server:

```bash
paperclip-plugin-dev-server --root . --ui-dir dist/ui --port 4177
```

Install a scaffolded plugin into a running Paperclip instance:

```bash
curl -X POST http://127.0.0.1:3100/api/plugins/install \
  -H "Content-Type: application/json" \
  -d '{"packageName":"/path/to/plugin","isLocalPath":true}'
```

---

## 7. Worker–Host Protocol

Communication is JSON-RPC 2.0 over stdio (newline-delimited). The SDK handles framing via `MESSAGE_DELIMITER`. The host calls into the worker (host-to-worker methods): `initialize`, `health`, `validateConfig`, `configChanged`, `shutdown`, `onEvent`, `runJob`, `handleWebhook`, `getData`, `performAction`, `executeTool`. The worker calls back into the host (worker-to-host) for all `ctx.*` operations.

The sandbox layer (`plugin-runtime-sandbox.ts`) uses Node's `vm` module to isolate plugin module loading. Default allowed globals: `console`, `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval`, `URL`, `URLSearchParams`, `TextEncoder`, `TextDecoder`, `AbortController`, `AbortSignal`. Module specifiers are allowlisted.

---

## 8. Strategic Implication for Tortuga

This discovery changed the Tortuga integration approach from fork to plugin.

**Previous approach:** fork Paperclip, add Tortuga features inline. Problem: maintenance burden, constant merge conflicts with upstream, inability to ship Tortuga independently.

**Current approach:** ship `@bopen-io/tortuga-plugin` — a standard Paperclip plugin. No fork required.

What the plugin can provide using the documented SDK surface:

| Tortuga feature | SDK mechanism |
|---|---|
| Agent template sync from ClawNet registry | `ctx.jobs` (scheduled sync) + `ctx.state` (cursor tracking) + `ctx.http` (outbound to ClawNet) + `ctx.agents` (list/invoke agents) |
| Skills as Paperclip agent tools | `ctx.tools.register()` — each skill becomes a tool callable by Paperclip agents |
| Fleet monitoring UI | `dashboardWidget` slot + `usePluginData` polling `ctx.agents.list()` |
| Webhook integrations | `webhooks` manifest declaration + `onWebhook` handler |
| Per-agent or per-project state | `ctx.state` with appropriate scope kind |
| Real-time logs or streaming output | `ctx.streams` (worker → UI SSE) |
| ClawNet API key storage | `secrets.read-ref` capability + `ctx.secrets.resolve()` |

The plugin boundary also isolates Tortuga from Paperclip internals: breaking changes in Paperclip's internals do not affect the plugin as long as the SDK surface (capability-gated, versioned) remains stable.

### Immediate Next Steps

1. Scaffold: `node packages/plugins/create-paperclip-plugin/src/index.ts @bopen-io/tortuga-plugin --output ~/code/tortuga-plugin`
2. Define the manifest — start with `agents.read`, `http.outbound`, `secrets.read-ref`, `plugin.state.read/write`, one `dashboardWidget` slot, one agent tool (ClawNet sync)
3. Implement ClawNet template sync as a scheduled job
4. Wire fleet monitoring widget using `usePluginData` + `usePluginStream`
5. Publish to a registry once `@paperclipai/plugin-sdk` is available on npm; use tarball deps in the interim

---

## 9. File Locations Quick Reference

| Item | Path |
|---|---|
| Server services | `/Users/satchmo/code/paperclip/server/src/services/plugin-*.ts` |
| REST routes | `/Users/satchmo/code/paperclip/server/src/routes/plugins.ts` |
| UI static routes | `/Users/satchmo/code/paperclip/server/src/routes/plugin-ui-static.ts` |
| SDK package root | `/Users/satchmo/code/paperclip/packages/plugins/sdk/` |
| SDK main exports | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/index.ts` |
| SDK UI exports | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/ui/hooks.ts` |
| SDK types | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/types.ts` |
| definePlugin | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/define-plugin.ts` |
| createTestHarness | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/testing.ts` |
| createPluginBundlerPresets | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/bundlers.ts` |
| Dev server | `/Users/satchmo/code/paperclip/packages/plugins/sdk/src/dev-server.ts` |
| Scaffolding CLI | `/Users/satchmo/code/paperclip/packages/plugins/create-paperclip-plugin/src/index.ts` |
| Hello World example | `/Users/satchmo/code/paperclip/packages/plugins/examples/plugin-hello-world-example/` |
| Kitchen Sink example | `/Users/satchmo/code/paperclip/packages/plugins/examples/plugin-kitchen-sink-example/` |
| File Browser example | `/Users/satchmo/code/paperclip/packages/plugins/examples/plugin-file-browser-example/` |
| Authoring Smoke example | `/Users/satchmo/code/paperclip/packages/plugins/examples/plugin-authoring-smoke-example/` |
| Plugin spec | `/Users/satchmo/code/paperclip/doc/plugins/PLUGIN_SPEC.md` |
