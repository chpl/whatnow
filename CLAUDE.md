# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

whatnow is an LLM-backed day planner. It ships as one Go binary that embeds a SvelteKit web UI and also runs a Bubble Tea terminal UI. Data lives in SQLite. The user brings their own Anthropic or OpenAI-compatible API key.

## Commands

Tooling: Go 1.27+, Node 24 (`web/.nvmrc`), and [just](https://github.com/casey/just). Run `just` from the repo root.

| Command | What it does |
|---|---|
| `just dev` | Runs the API on `127.0.0.1:7777` and the Vite dev server on `http://localhost:5173`. Vite proxies `/api` to the API. |
| `just test` | Runs `go test ./...`, then `cd web && npm test` (Vitest). |
| `just lint` | Runs `go vet ./...`, then `svelte-check` (`npm run check`). |
| `just e2e` | Builds the web UI, then runs Playwright against a server built with `-tags e2e`. |
| `just build` | Builds the web UI into `internal/web/dist/`, then builds `./whatnow`. |
| `just demo` | Runs on `.demo/whatnow.db` and port 7778, on a fake clock. Move the clock with `just clock 13:45`. |
| `just serve` / `just tui` | Runs the web-only server or the terminal UI with `go run`. |

Single tests:

```sh
go test ./internal/planner -run TestName
cd web && npx vitest run src/lib/plan.test.ts
cd web && npx playwright test e2e/today.spec.ts
```

TUI snapshot tests compare against `internal/tui/testdata/*.golden`. After you check a view change by hand, rewrite the goldens with:

```sh
go test ./internal/tui -update
```

Before a PR, `CONTRIBUTING.md` asks for `just test`, `just lint`, and `just e2e`.

## Architecture

### Entry point and modes

`cmd/whatnow/main.go` picks a subcommand:

- `whatnow` / `whatnow tui`: the TUI. It also serves the web UI on the same address, unless another process already holds that address.
- `whatnow serve`: the web UI only.
- `whatnow host`: multi-user hosted mode with Google sign-in (`internal/host`). See `docs/hosting.md`.
- `whatnow jira setup|push`: sends Jira tickets from a local machine to hosted whatnow.

### One App per database

`internal/app.Open` builds every part for one SQLite database and starts the background goroutines. The server, the TUI, and each hosted user all start from it.

```
store.Open (migrations)  → *sql.DB
service.New(db)          → the only write path
stream.NewHub()          → in-process pub/sub, fanned out to SSE clients
cleanup.Runner           → turns captured raw text into tasks/events/blockers (light model)
planner.Planner          → day plan, "what now?", wrap-up, reflection (main model)
agent.Agent              → chat turns with tool calls (main model)
integrations.Runner      → gcal, jira, linear, reviewqueue pulls
```

Background loop: the planner subscribes to `batch.applied` and `batch.undone` hub messages. `planner/triggers.go` decides whether a batch calls for a replan.

Hosted mode (`app.Hosted`) changes three things for each user. Secrets are sealed in the user's database instead of the OS keychain. Outbound HTTP goes through `internal/outbound`, which blocks private addresses. Review Queue is disabled. `host.Registry` keeps one `App` open per signed-in user and closes it after 15 idle minutes.

### Writes, batches, and undo

`internal/service` is the only write path for tasks, blockers, events, and days. Every write belongs to a batch (`NewBatch`, with actor `user` or `agent`). Every write records an activity row with before and after snapshots. Undo replays a whole batch in reverse. New write paths must go through `service` so that undo and replan keep working.

### LLM calls

- `internal/llm` is the provider-neutral `Client` interface (`Complete` for structured JSON, `Chat` for streamed tool-calling turns). `anthropic.go` and `openai.go` implement it.
- `internal/settings` picks the provider and the main and light models. API keys live in the OS keychain (`internal/secrets`), never in the database. `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` also work.
- `internal/prompt.System` builds system prompts in a fixed layer order: core instructions, user style, "about me", then live context. The cache marker sits on the last stable layer.
- `internal/facts` computes free time, eligibility, blockers, and annotations ("due today", "pushed 4 times"). It does no I/O. Facts reach the model as annotations, not as filters.
- The planner validates each model reply against the facts. A rejected reply gets one retry with the problems attached. If the retry also fails, code builds a fallback answer.
- `internal/llm/fake` answers by fixed rules keyed on the request's `SchemaName`. The e2e build (`cmd/whatnow/e2e.go`, build tag `e2e`) swaps it in and uses an in-memory keychain. Playwright tests look for the fake's fixed strings, so keep them in sync. The e2e server must be built with the tag, because an untagged binary would wipe the real keychain's whatnow keys.

### HTTP API and web client

- `internal/httpapi` serves REST under `/api/v1/` and the SSE stream. `internal/web` serves the embedded build from `internal/web/dist/` and falls back to `200.html` for client routes.
- `web/` is SvelteKit with `adapter-static` and Svelte 5 runes. Shared state lives in `web/src/lib/*.svelte.ts`. Route groups: `(app)` for today and tasks, `(ritual)` for morning and wrap.
- The TUI and the web UI stay in sync through the hub. The TUI is a companion: settings, wrap-up, nudges, and task editing exist only in the web UI.

### Integrations

Each integration in `internal/integrations/<name>` implements `Integration`: it declares its settings fields (`Text` fields go in the database, `Secret` fields go in the secrets store) and pulls either events (written directly) or tickets (the user picks which become tasks). Pulls run only when the user asks.

### Migrations

SQL migrations are embedded from `internal/store/migrations/NNNN_*.sql` (per-user database) and `internal/host/migrations/` (hosted accounts database). Add a new numbered file. Never edit an applied one.

## Conventions

From `CONTRIBUTING.md`:

- Match the surrounding code's naming, comment density, and idioms. Keep the diff to what the PR is for.
- Comments say why, not what. Errors shown to the user are plain sentences and never contain a token or key.
- Go: `go vet` clean, table tests where they fit, test names describe behavior (`TestPullOffersAssignedIssuesMineFirst`).
- Web: TypeScript, Svelte 5 runes, Vitest test names start with "Should".
- A new feature gets a short design spec in `docs/superpowers/specs/`, like the existing ones.
- Commit messages follow Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`.
- License is AGPL-3.0.
