# Automation catalog — for the recommend phase of `harness-adopt`

Self-contained reference for step 6 (recommend repo-specific automations). Use it
to turn detected repo signals into **report-only** suggestions in
`adoption-report.md`. Distilled from Anthropic's `claude-automation-recommender`
(the `claude-code-setup` plugin) so it works without that skill installed.

Rules for using this catalog:

- **Report-only.** Never create, install, or enable anything. List suggestions with a one-line *why* (the concrete signal) and the opt-in step. Enablement is the user's trust decision.
- **Top 1–2 per category**, the most valuable for *this* repo. Skip irrelevant categories.
- **Gap-aware.** The harness already ships a baseline (below) — only recommend what it does **not** already cover.
- Go beyond this list when the repo's stack warrants it (web-search a stack-specific MCP/hook).

## What the harness already ships — do NOT re-recommend

- **Hooks:** dangerous-command block, generated-edit block, `bd-prime` (SessionStart). → It does **not** ship format/lint/type-check/test or notification hooks; those are fair game.
- **Agents:** `code-reviewer`, `docs-researcher`, `planner`, `spec-reviewer`, `implementer`, `claude-max`, `fable-max`, `fable-xhigh`. → `security-reviewer`, `test-writer`, `api-documenter`, `performance-analyzer`, `ui-reviewer`, `dependency-updater`, `migration-helper` are gaps.
- **Skills:** brainstorming, planning, TDD, systematic-debugging, subagent-driven-development, verification, etc.
- **Tracking:** beads. **Codex:** the bundled `codex-adapter`. **Navigation:** the separate `code-intel` plugin (serena+CBM+ast-grep).

## Phase 1 — detect (read these before recommending)

- Manifests: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `*.csproj`, `Gemfile`.
- Tool config: `.prettierrc*`, `eslint.config.*`/`.eslintrc*`, `ruff.toml` or `[tool.ruff]`, `tsconfig.json`, `mypy.ini`/`pyrightconfig.json`.
- Tests: `tests/`, `__tests__/`, `*.test.*`, `*.spec.*`, `pytest.ini`.
- Infra/CI: `.github/workflows`, `docker-compose.yml`, `Dockerfile`, k8s manifests, `.mcp.json`.
- Services: deps like `@supabase/*`, `pg`/`postgres`, `@aws-sdk/*`, `@sentry/*`, `stripe`; GitHub/GitLab remote; Linear refs (`ABC-123`).
- Scale/shape: file count (>500 = navigation-heavy), auth/payment/PII code paths, frontend vs API-only.

## MCP servers — signal → suggest

| Repo signal | MCP server | Why |
|---|---|---|
| Popular libs/SDKs (React, FastAPI, Prisma, Stripe, AWS SDK, `@anthropic-ai/sdk`) | **context7** | live docs → fewer hallucinated/outdated APIs |
| Frontend with UI/e2e/screenshots | **Playwright** (or Puppeteer for headless/scrape) | drive the running app |
| `@supabase/supabase-js` | **Supabase** | query tables, auth, storage |
| `pg`/`postgres`, raw SQL, migrations | **PostgreSQL** (Neon/Turso for those) | inspect real data, schema |
| GitHub remote, PR/issue workflow | **GitHub** | issues, PRs, Actions, releases |
| Linear / GitLab | **Linear** / **GitLab** | issue + sprint integration |
| `@aws-sdk/*`, Terraform/CDK | **AWS** | Lambda, S3, DynamoDB |
| Cloudflare Workers/Pages/R2/D1 · Vercel | **Cloudflare** / **Vercel** | edge deploy/config |
| `@sentry/*` · Datadog | **Sentry** / **Datadog** | error/APM investigation |
| Team Slack · Notion docs | **Slack** / **Notion** | notifications · docs |
| `docker-compose.yml` · k8s manifests | **Docker** / **Kubernetes** | container/cluster ops |

Setup note: prefer a **checked-in `.mcp.json`** so the whole team gets the same servers; debug with `claude --mcp-debug`.

## Hooks — signal → suggest (in `.claude/settings.json`)

| Signal | Hook |
|---|---|
| Prettier / ESLint config | PostToolUse(Edit\|Write): auto-format / auto-fix |
| Ruff / Black / isort | PostToolUse: format+lint Python |
| `gofmt` (go.mod) / `rustfmt` (Cargo.toml) | PostToolUse: format on edit |
| `tsconfig.json` / mypy·pyright | PostToolUse: type-check (`tsc --noEmit` / `mypy`) |
| Test dir + framework (jest/pytest/vitest) | PostToolUse: run related tests on edit |
| `credentials.json`/`secrets.*` beyond `.env` | PreToolUse: block edits (harness `.gitignore` already covers `.env`) |
| Lock files (`package-lock`, `Cargo.lock`, `poetry.lock`) | PreToolUse: block direct edits (change via package manager) |
| Long unattended sessions | Notification hook: `permission_prompt` / `idle_prompt` → sound/desktop alert |

Notification matchers: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`.

## Subagents — signal → suggest (gaps only, in `.claude/agents/`)

| Signal | Agent | Model · Tools |
|---|---|---|
| Auth / payment / PII code | **security-reviewer** | sonnet · Read,Grep,Glob (read-only) |
| Few tests vs source | **test-writer** | sonnet · +Write |
| REST/GraphQL/OpenAPI routes | **api-documenter** | sonnet · +Write |
| DB-heavy / hot paths | **performance-analyzer** | sonnet · Read,Grep,Glob,Bash |
| Frontend components | **ui-reviewer** | sonnet · Read,Grep,Glob |
| Outdated deps / `npm audit` | **dependency-updater** | sonnet · +Bash |
| Major framework upgrade | **migration-helper** | opus · +Bash |

Access guide: read-only (Read,Grep,Glob) for reviews; +Write for generation; +Bash for migrations/testing. Default model sonnet; opus for complex migration/architecture; haiku for cheap repetitive checks.

## Language servers — language → LSP plugin

`typescript-lsp` (TS/JS) · `pyright-lsp` (Python) · `gopls-lsp` (Go) · `rust-analyzer-lsp` (Rust) · `clangd-lsp` (C/C++) · `jdtls-lsp` (Java) · `kotlin-lsp` · `swift-lsp` · `csharp-lsp` · `php-lsp` · `lua-lsp`. (For Python the `code-intel` plugin already brings serena's LSP backend.)

## Custom skills — when suggesting one

Place in `.claude/skills/<name>/SKILL.md`. Useful frontmatter:

```yaml
---
name: skill-name
description: what it does + when to use it
disable-model-invocation: true  # user-only (side effects: deploy/send/commit)
user-invocable: false           # Claude-only (background knowledge)
allowed-tools: Read, Grep, Glob # restrict tools
context: fork                   # run isolated; pair with: agent: Explore
---
```

Good repo-specific candidates: `api-doc` (OpenAPI template), `create-migration`, `gen-test`, `new-component`, `pr-check`, `release-notes`.

## Output in `adoption-report.md`

A short "Recommended automations (opt-in)" section, grouped by category, each line:
**suggestion — why (concrete signal) — opt-in step**. End by noting the user can ask for more in any category.
