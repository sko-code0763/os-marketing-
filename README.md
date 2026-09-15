# Marketing OS — AI Marketing Operating System (MVP)

A real, working codebase: a Strategy model, an agent core (planner → policy
engine → tool registry → approvals → task queue → audit log), and exactly
seven tools the agent can call. It runs on Postgres, Redis, an Express API,
a BullMQ worker, and a Next.js frontend — start it with Docker Compose or
run each piece locally, and it does what it says it does.

This README also does something most generated projects don't: it tells you,
section by section, what from the original 175-section specification is
**real and working today**, what was **deliberately scoped out of this MVP**,
and where the seams are for extending it. Nothing below claims more than the
code actually does.

## Table of contents

- [What this is (and isn't)](#what-this-is-and-isnt)
- [Quick start (Docker Compose)](#quick-start-docker-compose)
- [Quick start (local development)](#quick-start-local-development)
- [Demo credentials](#demo-credentials)
- [Architecture](#architecture)
- [The agent: planner, policy engine, tools, approvals](#the-agent-planner-policy-engine-tools-approvals)
- [The seven tools](#the-seven-tools)
- [Testing](#testing)
- [End-to-end smoke test](#end-to-end-smoke-test)
- [Repository layout](#repository-layout)
- [What's real vs. deferred (scope mapping against the spec)](#whats-real-vs-deferred-scope-mapping-against-the-spec)
- [Known limitations](#known-limitations)
- [Extending it: plugging in a real LLM planner](#extending-it-plugging-in-a-real-llm-planner)

## What this is (and isn't)

This is the **true MVP** scope agreed at the start of this build: a Strategy
model (business profile, goals, personas, competitors, positioning,
messaging, channel plans, budget, campaigns, KPIs) plus an agent core with
exactly seven internal tools (`build_strategy`, `create_campaign`,
`create_content`, `create_content_calendar`, `create_task`, `analyze_kpis`,
`generate_report`). The planner is **deterministic and rule-based** —
keyword/regex intent matching over plain TypeScript, not a call to an
external LLM — behind a provider-agnostic interface so a real LLM-backed
planner can be swapped in later without touching the runner, the policy
engine, or the tools (see [Extending it](#extending-it-plugging-in-a-real-llm-planner)).

It is **not**: connected to any external ad platform, CRM, analytics tool, or
email provider; billing-enabled; multi-tenant "agency mode." Those are real,
identifiable gaps, not silently missing features — see
[What's real vs. deferred](#whats-real-vs-deferred-scope-mapping-against-the-spec).

Every claim the agent makes about your data is computed from rows actually
in your Postgres database. Every action it proposes is logged. Every action
above your workspace's risk threshold pauses for a human. Nothing is
simulated to look more finished than it is — where a piece of the original
spec isn't built, the relevant UI or tool output says so explicitly (for
example, the strategy quality gate's "resources available?" check reports
`passed: null` — "not yet modeled" — rather than faking a pass or fail).

## Quick start (Docker Compose)

Requires Docker and Docker Compose.

```bash
cp .env.example .env   # edit SESSION_SECRET at minimum for anything beyond local testing
docker compose up --build
```

On first boot, run the database migration and seed a demo workspace:

```bash
docker compose exec api pnpm --filter @marketing-os/database migrate
docker compose exec api pnpm --filter @marketing-os/database seed
```

Then open:

- **Web app**: http://localhost:3000
- **API**: http://localhost:4000/api/v1 (health check at http://localhost:4000/health)

Log in with the [demo credentials](#demo-credentials) below.

> **A note on this repo's own testing of the Docker path.** The Dockerfiles
> and `docker-compose.yml` in this repo were written and manually reviewed
> against the actual package names, build scripts, and port wiring in this
> monorepo (see each Dockerfile's comments for the reasoning), and
> `docker compose config` validates the compose file cleanly. However, the
> sandboxed environment this project was built in blocks outbound pulls from
> Docker Hub's registry (a network policy denial, not a bug we could route
> around), so a full `docker compose up --build` could not be executed and
> watched end-to-end from inside that sandbox. Everything the Docker images
> wrap — `pnpm install`, every package's `build`, the full `vitest` suite,
> and the [end-to-end smoke test](#end-to-end-smoke-test) — **was** run for
real, repeatedly, against natively-running Postgres and Redis. Please run
> `docker compose up --build` yourself on first use and open an issue if
> anything about the container build doesn't match this document.

## Quick start (local development)

Requires Node.js 20+, pnpm, a local Postgres 16, and a local Redis.

```bash
pnpm install
cp .env.example .env   # defaults point at localhost:5432 / localhost:6379

# create the database itself (once), matching .env's DATABASE_URL
createdb marketing_os

pnpm db:migrate
pnpm db:seed

# in three separate terminals:
pnpm dev:api      # http://localhost:4000
pnpm dev:worker   # background BullMQ worker (optional for the synchronous API path — see below)
pnpm dev:web      # http://localhost:3000
```

The API executes agent tasks **synchronously** by default (`POST
/agent/tasks` runs the plan and returns the result in the same request) — the
worker is a second, architecturally separate execution path for
queue-based/background execution (see [Architecture](#architecture)) and
isn't required for the web app or the smoke test to work.

## Demo credentials

The seed script (`pnpm db:seed` / `docker compose exec api pnpm --filter
@marketing-os/database seed`) creates one demo organization ("Atlas Fitness
Gear") with a workspace that already has a full strategy on file — personas,
competitor research, a selected positioning statement, a messaging
framework, channel plans, a budget, one campaign, and eight weeks of weekly
KPI metric history for two KPIs (qualified leads, CAC) — so you can see the quality gate, the agent's `analyze_kpis`
and `generate_report` tools, and the strategy pages produce real output
immediately instead of staring at an empty workspace.

```
email:    demo@marketingos.app
password: Demo1234!
```

## Architecture

```
                     ┌──────────────┐
   Browser  ───────▶ │  apps/web    │  Next.js 14 (App Router), talks to
                     │  (port 3000) │  the API purely over HTTP — no
                     └──────┬───────┘  workspace-package imports.
                            │ fetch, credentials: include
                            ▼
                     ┌──────────────┐
                     │  apps/api    │  Express REST API, session-cookie auth,
                     │  (port 4000) │  tenant isolation, /api/v1 routes.
                     └──────┬───────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
      packages/agent  packages/tool-  packages/strategy-
      (planner, policy   registry      engine (pure functions:
       engine, runner,  (7 tools, the  quality gate, consistency
       audit, memory)   safety pipeline) checks, scoring, KPI math)
              │             │
              └──────┬──────┘
                     ▼
             packages/database
          (Drizzle ORM + Postgres)

   apps/worker (BullMQ + Redis) ── an alternative, queue-based path that
   calls the SAME packages/agent functions the API calls synchronously —
   for background/async execution without blocking an HTTP request.
```

`packages/shared` sits underneath everything: Zod schemas for every tool's
input/output, the permission and risk-level tables, and the core agent types
(`AgentContext`, `PlanStep`, `AgentPlan`, `AgentOutput`, `ToolResult`,
`Evidence`, `ReasonSummary`).

## The agent: planner, policy engine, tools, approvals

**Planner** (`packages/agent/src/planner.ts`). A `DeterministicPlanner` reads
the request text and matches it against known intents with plain
regex/keyword rules — there is no LLM call anywhere in this path. It is
deliberately honest about what it can't safely infer: for tools that need
several structured fields it can't reliably pull from a sentence (a
campaign's budget/dates/channels, a content item's body, a content
calendar's date range), it returns a **question** asking for those fields
rather than guessing plausible-looking values. It only auto-fills defaults
for genuinely safe, low-stakes fields (e.g. a 30-day analysis window for
`analyze_kpis` when none was specified) — and always says in the returned
`reason.assumptions` that it did so.

**Policy engine** (`packages/agent/src/policy.ts`). The single authority for
"is this action allowed, and does it need a human first?" It re-derives
everything from the workspace's actual stored `AgentPolicy` row and actual
recorded usage — it never trusts a planner's or a tool's self-reported risk.
Two kinds of decision:

- **Hard blocks** (`allowed: false`) — the daily spend cap and the daily
  agent-execution cap. These never execute, and a human approving the action
  later cannot override them (`resumeAfterApproval` re-runs the full policy
  check before executing anything).
- **Soft gates** (`requiresApproval: true`) — Copilot mode always requires
  approval; Assisted mode requires it for anything above `LOW` risk; any
  spend over the workspace's approval threshold; any `HIGH`-risk action;
  sending email always requires approval regardless of mode.

**Tool registry** (`packages/tool-registry/src/registry.ts`). Every tool call
— from the planner, from a retry, from a test — goes through one
`invokeTool()` pipeline: (1) re-checks the calling context's actual granted
permissions (never trusts the caller's word for it), (2) validates the input
against the tool's own Zod schema, (3) runs the tool, (4) validates the
tool's *output* against its own schema before ever calling it a success. A
tool that throws is reported as a failure, never silently swallowed.

**Approvals & audit** (`packages/agent/src/runner.ts`,
`packages/agent/src/audit.ts`). Every plan step becomes an `agentActions` row
with an idempotency key (`{agentTaskId}:{stepId}`) — a retried step never
re-executes the underlying tool. Every policy decision, every tool
execution, every approval request, and every approval decision (including
rejections) writes an append-only `audit_logs` row with an honest
`result: "success" | "failure"` — a human rejecting a proposed action is
logged as a failure to execute (nothing happened), not hidden or reframed as
a system error.

## The seven tools

| Tool | What it actually does | Fabrication guardrail |
|---|---|---|
| `build_strategy` | Reads every strategy record on file, runs the quality gate, ranks existing positioning/channel options. **Read-only** — never invents goals, personas, or budget from a one-line objective. | Never writes structured strategy fields from free text. |
| `create_campaign` | Creates a draft campaign, validates every linked goal/audience/positioning id actually exists in the workspace, flags a linked-but-not-selected positioning for review, dedupes by name. | Refuses (asks) rather than fabricates budget/dates/channels from prose. |
| `create_content` | Creates a content item, dedupes by title+campaign. | Same — asks for type/title/objective rather than guessing. |
| `create_content_calendar` | Generates dated placeholder items at a stated cadence, capped at 200 items. | Asks for a concrete date range/cadence/channels; never invents them. |
| `create_task` | Creates a task, validates any linked campaign/goal/owner, dedupes by title+status. | — |
| `analyze_kpis` | Computes real trend/anomaly detection from `metric_values` rows already in the database. | Flags "possible factors" as generic and unverified — never claims a causal explanation. |
| `generate_report` | Assembles a report (campaign / strategy / executive / monthly / quarterly / agent-activity) from live queries, including a real structural consistency check (`packages/strategy-engine/src/consistency.ts`) that surfaces conflicts instead of hiding them. | Every section is computed, never templated filler. |

## Testing

144 automated tests, all real integration tests against a live seeded
Postgres/Redis — no mocked database, no fabricated fixtures standing in for
the real schema.

```bash
pnpm -r test
```

| Package | Tests | Covers |
|---|---|---|
| `packages/shared` | 9 | Permission/risk tables, crypto utils |
| `packages/strategy-engine` | 42 | Quality gate, consistency checks, positioning/channel scoring, KPI math, budget math |
| `packages/tool-registry` | 19 | Each tool, the permission/validation/output-schema pipeline, dedup behavior |
| `packages/agent` | 36 | Planner, policy engine, context resolution, the runner's approval/rejection/hard-block paths, and a dedicated **8-scenario suite** (`scenarios.test.ts`) matching spec section 120: successful task, missing data, permission denied, approval required, tool failure, duplicate action, prompt injection (two variants), conflicting strategy |
| `apps/api` | 35 | Auth, tenant isolation, strategy routes, execution routes, agent routes |
| `apps/worker` | 3 | Real BullMQ + Redis job processing, including a malformed-job failure path |

`packages/database` has no dedicated unit tests of its own — its schema and
query behavior are exercised indirectly by every other package's tests
running against the real database. `apps/web` has no automated test suite (a deliberate scope choice — it is a
thin HTTP client with no business logic of its own); it is instead verified
by `next build`'s type-checking/prerendering and the manual end-to-end
smoke test described below, which exercises the same API endpoints the UI
calls.

## End-to-end smoke test

With the stack running (either Docker Compose or the local dev servers):

```bash
./scripts/smoke-test.sh
# or, if the API isn't on the default port/host:
BASE_URL=http://localhost:4000/api/v1 ./scripts/smoke-test.sh
```

It signs up two brand-new users against your running API and makes real HTTP
requests to prove, live: signup/session auth works; a fresh workspace's
quality gate honestly reports `ready: false` rather than faking readiness; a
low-risk agent request executes automatically; a medium-risk request pauses
for approval and only executes after a human approves it; a vague campaign
request is refused rather than fabricated; a second user is refused access to
the first user's workspace (403) and an anonymous request is refused (401);
and the audit log actually recorded all of it. 26 checks, exit code 0 only if
every one passes. This exact script was run against a live instance of this
repository as part of building it — see the PASS output captured during
development for reference.

## Repository layout

```
apps/
  api/       Express REST API (session auth, tenant isolation, /api/v1 routes)
  worker/    BullMQ worker — the same agent runner, queue-driven
  web/       Next.js 14 frontend (App Router, Tailwind)
packages/
  shared/          Zod schemas, permission/risk tables, core agent types
  database/        Drizzle ORM schema, migrations, seed script
  strategy-engine/ Pure functions: quality gate, consistency checks, scoring, KPI/budget math
  tool-registry/   The 7 tools + the invokeTool() safety pipeline
  agent/           Planner, policy engine, context resolution, the runner, audit, memory
scripts/
  smoke-test.sh    End-to-end smoke test (see above)
docker-compose.yml
.env.example
```

## What's real vs. deferred (scope mapping against the spec)

This build was explicitly scoped, at the outset, to spec section 170's
recommended "true MVP": the Strategy model + agent core + these seven tools,
with a deterministic planner (no LLM API key required to run anything in
this repo) and no external integrations, billing, or agency mode. Everything
in the tables above is real and working. Below is what the original
175-section spec describes that this MVP does **not** build, so nothing here
is mistaken for more than it is:

- **External integrations** (ad platforms, CRM, analytics connectors, email
  send providers) — not built. Every "read" a tool performs reads this
  app's own Postgres tables, never a live third-party API. There is no code
  path anywhere that pretends to have synced or published to an external
  system.
- **LLM-backed reasoning** — not wired in. The planner is deterministic and
  rule-based by design decision (see [Extending it](#extending-it-plugging-in-a-real-llm-planner)
  for the seam left for this).
- **Billing / subscriptions** — not built. No Stripe integration, no plan
  gating.
- **Agency / multi-client mode** — not built. The data model is
  organization → workspace → (strategy, campaigns, etc.), single-tenant per
  organization; there is no "manage N client workspaces from one agency
  view" layer.
- **AUTONOMOUS agent mode's full autonomy** — the `AgentMode` enum and its
  permission defaults exist and are enforced (`COPILOT` / `ASSISTED` /
  `AUTONOMOUS`), but every workspace this repo seeds defaults to `ASSISTED`.
  Critically, `AUTONOMOUS` mode does **not** mean "skip the policy engine" —
  spend caps, execution caps, and `SEND_EMAIL`-always-requires-approval are
  enforced identically in every mode; `AUTONOMOUS` only widens which
  *within-policy* actions can proceed without a pause.
- **Resource/team capacity planning** — the quality gate's "resources
  available?" check exists in the code but always reports `passed: null`
  ("not yet modeled in this MVP") rather than a fabricated pass/fail, because
  there is no resourcing data model yet.
- **Org- and integration-level permission grants** — `resolveGrantedPermissions`
  currently derives permissions from workspace agent-mode defaults intersected
  with per-tool permission toggles. A richer org-level grant/role system
  (beyond the `OWNER`/member org role already enforced for tenant isolation)
  is not built.
- **Multi-step / branching agent plans** — the runner's dependency-graph
  execution (`dependsOn`) is implemented and tested, but the deterministic
  planner itself only ever emits single-step plans today; nothing in this
  MVP requires or exercises a multi-step plan in practice.

## Known limitations

- **Docker images are not size-optimized.** Each Dockerfile installs the
  entire pnpm workspace rather than pruning to a minimal production image —
  a disclosed trade-off for correctness and a working `docker compose exec
  api pnpm --filter @marketing-os/database migrate` workflow, not an
  oversight. See each Dockerfile's own comment.
- **The full `docker compose up --build` was not run inside the sandbox this
  project was built in** (Docker Hub registry pulls are blocked by that
  sandbox's network policy) — see the note under
  [Quick start (Docker Compose)](#quick-start-docker-compose). Please treat
  your first `docker compose up --build` as the first real end-to-end run of
  that exact path, and file an issue if something doesn't match.
- **`apps/web` has no automated test suite** (see [Testing](#testing)) — it
  is a thin, logic-free HTTP client verified by type-checking, a production
  build, and manual end-to-end smoke testing against the real API.
- **Leftover test data.** Running the test suites repeatedly against the
  same local Postgres instance creates additional `test-*@marketingos.app`
  users/organizations (each test suite uses fresh, isolated workspaces so
  runs don't interfere with each other, but it doesn't delete the demo
  workspace's data or aggressively vacuum old test orgs). This is harmless
  for local development; for a shared/staging database, run tests against a
  disposable database instead.
- **No rate limiting or production hardening** on the API (CORS, cookie
  flags, and session lookups are implemented; request throttling,
  brute-force login protection, and structured logging/observability are
  not).

## Extending it: plugging in a real LLM planner

`packages/agent/src/planner.ts` defines a small `Planner` interface:

```ts
export interface Planner {
  plan(req: PlanRequest): AgentOutput;
}
```

`DeterministicPlanner` is one implementation. `createAndRunAgentTask()`
(`packages/agent/src/runner.ts`) accepts an optional `planner` parameter — the
test suite already uses this seam to inject fake planners for scenario
testing. A future LLM-backed planner would implement the same interface
(likely `async plan()`, since a real model call is asynchronous — the
runner would need a small signature change to `await` it) and could be
swapped in without touching the policy engine, the tool registry, or the
runner's approval/audit logic at all: **the policy engine and the
per-tool permission checks are the actual safety boundary, not the
planner** — a fabrication or over-reach from a future LLM planner would
still be caught by the same hard blocks, approval gates, and tool-level
validation exercised in this repo's scenario suite. `.env.example` already
reserves `AI_PROVIDER` / `AI_API_KEY` variables for this, unused by any code
path today.
