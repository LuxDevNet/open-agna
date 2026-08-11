# OpenCode → Agnamo Development Context

> Snapshot: 2026-08-11. This is descriptive project context, not a replacement for
> `AGENTS.md` or package-local `AGENTS.md` files. Follow those files for current
> implementation rules and update this document when architecture changes.

## Direction

This repository is currently the OpenCode monorepo: an open-source AI coding
agent with TUI, CLI, web, and Electron clients over a shared agent runtime.

The working product direction is to evolve this fork into `agnamo-dev` (name not
final): a personal, local-first agent control plane that feels authored for one
user rather than a generic rebrand. Preserve OpenCode's strong runtime seams and
replace product identity, defaults, workflows, memory, review, and orchestration
deliberately.

The most valuable target composition is:

1. one durable local agent engine;
2. replaceable desktop, TUI, CLI, web, and IDE clients;
3. isolated workspaces for parallel agents;
4. a review and attention queue rather than an undifferentiated chat list;
5. capability-scoped approvals and credentials;
6. inspectable, reversible personal memory;
7. provider-neutral models and portable MCP/ACP integration;
8. optional hosted services without making the local core dependent on them.

Do not begin with a broad string-replacement rebrand. First establish the
desired product boundary, retain upstream mergeability where useful, and
separate personal distribution/configuration from changes that truly require a
runtime fork.

## Architectural Summary

The repository is a Bun workspace coordinated by Turbo. It uses TypeScript,
Effect v4 beta, Effect Schema, SQLite/Drizzle, SolidJS, OpenTUI, Vite, and
Electron.

The dependency direction is load-bearing:

```text
Schema ───────▶ Core ───────▶ Server
   │             ▲              ▲
   └────────▶ Protocol ──────────┘
   │             │
   └────────────▶ Client

sdk-next = Client + Core + Server composition
opencode = product/compatibility composition
```

- `packages/schema`: browser-safe wire and storage contracts only.
- `packages/protocol`: public Effect `HttpApi` contracts and middleware
  placement.
- `packages/core`: current domain runtime: sessions, events, tools,
  permissions, providers, plugins, projects, workspaces, persistence, and
  system context.
- `packages/server`: concrete handlers, middleware, auth, and route assembly.
- `packages/client`: generated Promise and Effect clients. Never hand-edit
  `src/generated` or `src/generated-effect`.
- `packages/sdk-next`: embeds the real Core/Server stack behind an in-process
  fetch handler and exposes it through the typed Client.
- `packages/opencode`: shipping product composition plus substantial V1
  compatibility code, CLI commands, daemon/server code, MCP, and legacy session
  runtime.
- `packages/llm`: provider-neutral, Schema-first LLM request/event runtime.
- `packages/plugin`: public plugin contracts, including current Effect and
  Promise facades.
- `packages/codemode`: confined execution over explicit schema-described tools.

Client runtime code may depend on Schema and Protocol, never Core or Server.
Keep generic infrastructure packages such as `effect-drizzle-sqlite`, `llm`,
and `codemode` free of application-specific state.

## Product Surfaces

### Shared Solid application

`packages/app/src/entry.tsx` and `packages/app/src/app.tsx` compose the shared
web/desktop interface. The provider tree separates app-global state,
server-scoped state, settings, connection lifecycle, tabs, permissions, and
notifications.

Important seams:

- `packages/app/src/context/platform.tsx`: web/desktop capability abstraction.
- `packages/app/src/context/global.tsx`: known server connections.
- `packages/app/src/context/server-sdk.tsx` and `server-sync.tsx`: per-server
  client and synchronization state.
- `packages/app/src/context/command.tsx`: command palette/keybinding registry.
- `packages/app/src/context/settings.tsx`: persisted behavior and appearance.
- `packages/app/src/pages/session/composer`: modular prompt, permission,
  question, follow-up, revert, and todo docks.
- `packages/app/src/pages/session/timeline`: custom projected and virtualized
  session timeline.

The app currently contains legacy and new layout generations. Treat that as
migration debt, not a pattern to reproduce. Check the current gate and sunset
logic in `packages/app/src/context/settings.tsx` before modifying either path.

### Desktop

`packages/desktop` is a thin Electron shell:

- `src/main/index.ts`: main-process composition.
- `src/main/ipc.ts`: IPC handler registration.
- `src/preload/index.ts` and `src/preload/types.ts`: the only renderer bridge,
  exposed as `window.api`.
- `src/main/sidecar.ts` and `src/main/server.ts`: local server lifecycle.
- `src/main/wsl` and `packages/app/src/wsl`: Windows/WSL runtime support.

Renderer code must not bypass `window.api`. Keep native capabilities,
credential access, filesystem dialogs, updater behavior, and sidecar lifecycle
outside the renderer.

### TUI and CLI

- `packages/tui`: SolidJS rendered through OpenTUI, with routes, themes,
  keymaps, plugin slots, and built-in feature plugins.
- `packages/cli`: newer command framework and native wrapper.
- `packages/opencode/src/index.ts`: existing shipping command entrypoint and
  compatibility composition.

`packages/cli/package.json` currently exposes a binary named `lildax` while most
of the repository remains named OpenCode. The reason is not documented here;
treat this as an inconsistency to investigate before adopting it as a rebrand
pattern.

### Shared visual layers

- `packages/ui`: design system, themes, icons, i18n, and component primitives.
- `packages/session-ui`: message/tool rendering, worker-based Markdown
  processing, and Pierre-based file/diff review.
- `packages/storybook`: isolated product/component development with app-context
  mocks.
- `packages/identity`: shared mark assets.
- `packages/desktop/icons/{dev,beta,prod}`: channel-specific packaged icons.

The timeline, Markdown worker pipeline, and Pierre diff virtualization are
specialized performance-sensitive systems. Extend them through their existing
models and worker boundaries; do not casually replace them.

### Hosted and public surfaces

- `packages/web`: Astro marketing and documentation site.
- `packages/console`: hosted control plane, auth, billing, mail, and support.
- `packages/enterprise`: separate enterprise-facing SolidStart surface.
- `packages/stats`: usage/statistics product.
- `packages/function` and `packages/slack`: integration surfaces.

These are separable from the local agent product. Decide explicitly whether the
fork keeps, removes, or replaces each hosted surface.

## Current Session Runtime

The current Core session architecture is durable and event-sourced:

```text
prompt admission
  → durable session_input / PromptAdmitted
  → advisory SessionExecution wake
  → process-global SessionRunCoordinator
  → Location-scoped SessionRunner
  → one llm.stream(request) per provider turn
  → durable assistant/tool projections
  → reload history
  → explicit continuation
```

Key files:

- `packages/core/src/session/input.ts`: durable steer/queue admission.
- `packages/core/src/session/execution/local.ts`: Session-ID-to-Location routing.
- `packages/core/src/session/run-coordinator.ts`: same-session serialization,
  wake coalescing, interruption, and cross-session concurrency.
- `packages/core/src/session/runner/llm.ts`: current provider-turn orchestration.
- `packages/core/src/session/history.ts`: model-visible history projection.
- `packages/core/src/session/context-epoch.ts`: durable system-context baseline
  and chronological updates.
- `packages/core/src/event.ts`: durable event publication and replay.

Preserve these invariants:

- admission and execution are separate;
- a provider turn has one explicit `llm.stream` call;
- projected history is reloaded before continuation;
- Session execution routing is process-global and Session-ID based;
- model resolution, tools, permissions, filesystem, and runner are
  Location-scoped;
- local drains are not clustered or crash-recovered yet;
- steer and queue have different promotion semantics;
- generic tool-output bounding occurs once at settlement.

V1 remains load-bearing in `packages/opencode/src/session`, plugin, MCP, and
parts of the shipping clients. Current/Core and V1 are not interchangeable.
Avoid adding durable product behavior to the V1 monolith unless compatibility
requires it.

## Scope and Dependency Injection

Core uses a two-level Effect graph:

- process-global services: database, event store, application tools, project
  lookup, Session store/execution, and Location service map;
- Location-scoped services: config, agents, commands, references, integrations,
  catalog, plugins, filesystem, PTY, skills, permissions, tools, models, and
  Session runner.

The principal implementation is in:

- `packages/core/src/effect/layer-node.ts`
- `packages/core/src/effect/app-node.ts`
- `packages/core/src/location-services.ts`
- `packages/core/src/location-service-map.ts`

`Location.Ref` is a directory plus optional workspace identity. The
`LocationServiceMap` caches a complete scoped service graph for each Location.
Do not promote Location-scoped services to globals for convenience.

## Tools, Permissions, and Code Mode

Core has one canonical opaque tool representation:

- `packages/core/src/tool/tool.ts`: `Tool.make`.
- `packages/core/src/tool/application-tools.ts`: process-scoped application
  registrations.
- `packages/core/src/tool/tools.ts`: registration-only Location service.
- `packages/core/src/tool/registry.ts`: effective lookup, materialization,
  invocation, validation, and bounded settlement.
- `packages/core/src/permission.ts`: wildcard allow/deny/ask policy and pending
  approvals.

Location registrations override application registrations. Materialization
captures the advertised tool set for one provider turn, and settlement rejects
stale calls rather than executing a replacement tool with the same name.

Execution authorization belongs in trusted tool leaves, not in a generic
registry callback. Approval UI should eventually expose the exact capability
being granted: command, working directory, paths, network origins, secret
access, and tool/MCP identity.

Use `packages/codemode` for curated, schema-described, budgeted tool access.
Code Mode is not a generic host sandbox and must not receive ambient filesystem,
network, or credential authority.

## Models and Providers

`packages/llm` separates four axes:

1. Protocol: request lowering and stream state machine.
2. Endpoint: URL/path/query.
3. Auth: headers or request signing.
4. Framing: SSE, event stream, or another transport frame format.

They compose through `Route.make(...)`. Prefer adding provider facades or
reusing compatible protocols over cloning a protocol implementation.

There are still parallel runtime paths:

- legacy/product runtime defaults to the AI SDK and can opt into the native
  adapter;
- current Core SessionRunner uses the native `@opencode-ai/llm` route and has a
  narrower provider set.

Do not assume every provider listed by the shipping UI works through every
session path. Verify current lowering support in
`packages/core/src/session/runner/model.ts` and
`packages/opencode/src/session/llm`.

## Plugin and Integration Seams

Prefer designed extension points before forking runtime code:

1. `PluginV2` and `PluginHost` in `packages/core/src/plugin.ts` and
   `packages/core/src/plugin/host.ts` for Location-scoped agents, catalogs,
   model hooks, integrations, commands, references, and skills.
2. `ApplicationTools.Service` and `sdk-next` tool registration for
   process-scoped custom tools.
3. `packages/core/src/system-context` for stable, attributable personal context.
4. `AISDK.Service` hooks for provider SDK/language-model construction.
5. `packages/llm` routes for first-class native provider support.
6. `Integration.Service` for credentials, OAuth, and connected accounts.
7. TUI plugin routes, slots, commands, themes, and attention packs.
8. App command registry, Platform interface, and composer docks for UI features.

MCP currently lives primarily in the V1 `packages/opencode/src/mcp` layer.
Current Core does not yet have a canonical Location-scoped MCP registration
design. A future MCP/Core bridge should register canonical tools through
`Tools.Service`/`ToolRegistry`, preserve permission and settlement semantics,
and avoid flattening hundreds of tools into every model request.

ACP is a strong future client/agent interoperability boundary; MCP should remain
the tool/resource boundary.

## High-Value Product Features

### Build first

- Workspace manager: one task owns a worktree/branch, terminals, ports, setup,
  lifecycle, and cleanup policy.
- Attention queue: running, blocked, approval required, checks failed, and
  review ready.
- Review pipeline: structured diff, changed-file map, tests/lints, risk summary,
  comments, and explicit apply/commit/PR actions.
- Typed local control API: durable session events, ephemeral progress,
  approvals, artifacts, and workspace lifecycle.
- Capability-scoped grants: one-shot/session/project scope for filesystem,
  command, network, secrets, and MCP access.

### Build next

- ACP client/server interoperability and adapters for external agents.
- Optional inspectable workflow graphs: plan → implement → verify → review.
- Personal memory with source, timestamp, scope, confidence, retention, edit,
  and delete controls.
- Model routing by capability, privacy, locality, cost, and latency.
- Session continuity across desktop, TUI, CLI, and IDE.
- Secure OS-keychain credential vault with scoped injection.

### Later differentiation

- Interactive MCP/tool surfaces with explicit sandbox boundaries.
- Best-of-N candidates isolated by worktree and compared by checks, cost,
  latency, and diff.
- Docker, SSH, and self-hosted remote execution without mandatory cloud.
- Optional encrypted sync, hosted sandboxes, team policy/audit, and managed
  routing.

## Competitive Design Lessons

Research date: 2026-08-11. Product capabilities change quickly; verify linked
sources before making architecture decisions.

- ChatGPT/Codex: strong session continuity, subscription onboarding, local
  open CLI/app-server seam, sandbox and approval controls; proprietary desktop
  and cloud remain lock-in.
- Cursor: benchmark for parallel-agent, worktree, steering, diff review, and
  polished orchestration UX; proprietary runtime/routing and metering are the
  trade-off.
- Bastani Atomic: useful model for explicit stages, artifacts, checks,
  verifier passes, gates, checkpoints, and human approvals.
- AtomicBot Atomic Agent: separate local-first desktop automation project;
  useful for local-model and broad personal-agent comparison. “Atomic Code” is
  ambiguous, so keep these projects distinct.
- Nous Hermes Agent: broad personal-agent runtime with providers, memory,
  skills, subagents, schedules, messaging, and multiple sandbox backends.
- Hermes One (`fathah/hermes-desktop`): separate community Electron client,
  not an official Nous project.
- Goose: strong reference for an open desktop/CLI/API agent, MCP extensions,
  ACP, and distributable configurations.
- OpenHands: strong event-sourced runtime, replay, workspace abstraction, and
  client/server separation.
- Cline: strong reviewable edits, approvals, checkpoints, hooks, and MCP UX.
- Zed: clean native-agent, ACP-agent, and terminal-agent separation.
- Superset: strong workspace/orchestration UX, but Elastic License 2.0 is
  source-available rather than conventional open source.
- Void: archived editor fork; evidence against maintaining a full editor fork
  only to gain agent features.

Primary sources:

- OpenAI Codex: https://github.com/openai/codex
- Cursor Agent: https://cursor.com/docs/agent/overview
- Bastani Atomic: https://github.com/bastani-inc/atomic
- AtomicBot Agent: https://github.com/AtomicBot-ai/atomic-agent
- Nous Hermes Agent: https://github.com/NousResearch/hermes-agent
- Hermes One: https://github.com/fathah/hermes-desktop
- Goose: https://github.com/aaif-goose/goose
- OpenHands: https://github.com/OpenHands/OpenHands
- Cline: https://github.com/Cline/cline
- Zed AI: https://zed.dev/docs/ai/overview
- Superset: https://github.com/superset-sh/superset

## Anti-Patterns to Avoid

- Product state owned by the Electron renderer.
- A full editor fork merely to add agent chat.
- Treating “privacy mode” as equivalent to local execution.
- Persistent global auto-approval.
- Every MCP server/tool exposed to every agent.
- Concurrent writers sharing one checkout.
- Silent merging or promotion of agent output.
- Opaque memory or summarization that silently changes future behavior.
- Self-modifying skills without versioning, provenance, and rollback.
- Credentials stored as plaintext defaults.
- Proprietary routing, subscriptions, and session state coupled so tightly that
  users cannot migrate.
- More parallel agents than the review UX can safely supervise.

## Rebrand and Personalization Order

1. Define product promise, retained OpenCode compatibility, upstream merge
   strategy, and final name.
2. Create a distribution layer for default agents, skills, plugins, MCP
   servers, policy, themes, and personal context.
3. Add a custom theme using
   `packages/ui/src/theme/desktop-theme.schema.json`.
4. Replace identity assets in `packages/identity`,
   `packages/desktop/icons/{dev,beta,prod}`, app assets, and web assets.
5. Update product names through typed i18n dictionaries, package metadata,
   desktop bundle/app IDs, storage namespaces, and deep-link protocol.
6. Add workflow/review/memory features through the extension seams above.
7. Rename package scopes and public APIs only when the migration and publishing
   strategy is explicit.

Never hardcode new user-visible strings. Follow package-local localization rules
and preserve product names, placeholders, code tokens, and keyboard labels
intentionally.

## Known Migration Risks

- Current and V1 session/plugin/MCP stacks coexist.
- `packages/app` and `packages/session-ui` currently consume a vendored Client
  tarball, not necessarily the workspace `packages/client`; inspect
  `packages/app/V1_API_MIGRATION.md`.
- Current Core MCP/plugin tool parity is incomplete.
- Native current-provider coverage is narrower than legacy AI SDK coverage.
- Local Session execution has no clustered ownership or automatic post-crash
  continuation recovery.
- The UI has legacy/new layout and theme generations.
- Session/timeline changes require production benchmark baselines.
- The repository carries many patched dependencies; upgrades must reconcile
  `/patches`.
- Desktop and WSL behavior depend on platform-native PTY/watcher packages.
- A complete rebrand spans app, TUI, CLI, package scopes, desktop IDs/icons,
  storage keys, deep links, web/docs, hosted services, translations, and update
  channels.

## Working Commands and Generated Boundaries

- Default branch: `dev`; use `dev` or `origin/dev`, not assumed `main`.
- Root install/runtime: Bun 1.3.14 from `package.json`.
- Typecheck from package directories with `bun typecheck`; do not invoke `tsc`
  directly.
- Tests cannot run from repository root. Run the smallest relevant package test.
- After public Protocol/Server `HttpApi` changes:
  `cd packages/client && bun run generate`.
- Never edit `packages/client/src/generated` or `src/generated-effect`.
- Regenerate the legacy JS SDK with `./packages/sdk/js/script/build.ts`.
- Local app UI requires separate backend and frontend processes as documented
  in `packages/app/AGENTS.md`.
- Do not restart app/server processes while debugging app behavior.

## Documentation Maintenance

When a major feature or migration lands, update:

1. this file for stable architecture and product direction;
2. the owning package's `AGENTS.md` for implementation constraints;
3. `specs/` for detailed design and state-machine behavior;
4. generated clients after public API changes;
5. the competitive notes only when a decision depends on a reverified source.

Keep this file compact enough to load as working context. Put detailed
investigations, benchmark results, migration checklists, and decision records in
dedicated documents rather than continually expanding this overview.
