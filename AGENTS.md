# codemogger

Headless code-indexing and search library for AI coding agents: tree-sitter chunking, local embeddings, and vector + full-text search in a single SQLite file. Used by coding agents through an MCP server, by developers through a CLI, and by programs through the `CodeIndex` SDK. This is the `jckail` fork of `glommer/codemogger`.

<!-- BEGIN SHARED ENGINEERING STANDARDS (managed block: keep it intact; repo-specific rules go outside it and win on any conflict) -->
## Working standards (all repos)

These apply unless this repo's own rules, below or in linked docs, say otherwise. Repo-specific rules always win. If a rule here would break a repo gate (hooks, CI, branch protection, a documented release process), follow the gate and say so in your summary.

### Worktrees and branches
- **Every change gets its own worktree inside this repo, under `.worktrees/`** (gitignored, so all of a repo's worktrees sit at one level and none are committed): `git fetch --prune origin && git worktree add .worktrees/<topic> -b <branch> origin/<default-branch>`. Do not create worktrees anywhere else (no `../`, `~/projects/.worktrees`, or extra clones). Never switch branches or reset a checkout holding someone else's uncommitted work.
- **One focused branch per change, and every pushed branch names its Linear issue.** Cut it from a fresh `origin/<default-branch>` and use the issue's `gitBranchName` (e.g. `jckail13/jck-123-short-topic`) or `jck-123-short-topic`. A branch without a `jck-<n>` key is rejected by the pre-push hook and by the `branch-name` CI job; bot branches (`dependabot/*`, `renovate/*`, `release-please--*`) are the only exception. No issue yet? Find or create one first (see Linear below). Never commit directly to the default branch unless this repo explicitly says to.
- **Commit early and often, push as you go**, so no work exists only on this machine. Open a draft PR once there is a pushed commit. Don't let a branch drift for days: rebase or merge from `origin/<default-branch>` and rerun the gate before review.
- **Stage precisely** (`git add <paths>`, never `git add -A` or `.` blindly). Read `git status` and `git diff --staged` before every commit. Never commit secrets, `.env*`, credentials, large binaries or generated build output.
- **Never** force-push shared branches, rewrite published history, skip hooks (`--no-verify`) or bypass branch protection. If a hook or check fails, fix the cause or stop and report it.
- **Cleanup is automatic after merge.** GitHub deletes the head branch when a PR merges (`delete_branch_on_merge`). Locally, `git-prune-merged` (from agent-skills, run on a schedule and safe to run by hand) removes clean worktrees and local branches whose work is on `origin/<default-branch>`, whether by ancestry or by a merged PR with the same head. It never touches dirty, locked, unpushed or open-PR work. When you finish, leave your worktree clean and pushed so the sweep can remove it, or remove it yourself: `git worktree remove .worktrees/<topic> && git branch -d <branch>`.

### Commit messages
Every pushed commit must be understandable on its own. Match the style already in `git log` (conventional commits `type(scope): subject` if the repo uses them). Otherwise use an imperative subject of 72 characters or fewer with no trailing period, then a blank line, then a body. The body says **why** the change is needed, what it changes at a high level, trade-offs, and how it was verified. It doesn't restate the diff. Reference the Linear issue (`JCK-123`). "wip", "fix", "updates" and empty bodies are not acceptable on pushed commits; reword before pushing, never after. Add the attribution trailer your harness requires, if any.

### Pull requests and merging
Open a PR for every change and fill in `.github/pull_request_template.md`:
- **Summary**: what changed, in a few sentences.
- **Why**: the problem or request, with the Linear link.
- **Changes**: the notable parts, file by file only when that helps.
- **Verification**: the exact commands run and their result, plus the Depot CI run link.
- **Risk and rollback**: what could break and how to undo it.

Keep PRs small and the title in commit-subject style with the issue key. Merge only when the required checks are green (`depot-ci / ci-result`), using the merge method this repo prefers (some require linear history or forbid squash: check the repo's settings and history first). Do not merge a PR you could not verify.

### CI: Depot CI only
- **CI runs on Depot CI**, defined in `.depot/workflows/` (GitHub Actions syntax). Do not add routine CI to `.github/workflows/`. The only exceptions are jobs Depot can't run (macOS/iOS or Windows runners, `release:`/`issues:` events, `environment:`-gated cloud deploys), each kept dispatch- or tag-only with a comment saying why.
- **Workflow conventions:** `name: depot-ci`. Jobs run on `depot-ubuntu-24.04[-arm][-<cpus>]` (2 CPUs by default; size up only for measured wins) with `timeout-minutes`. Use `permissions: contents: read` and `persist-credentials: false`. Pin third-party actions to commit SHAs. Concurrency cancels superseded PR runs. A final `ci-result` job (`if: always()`, `needs:` every job) fails unless all succeeded; branch protection requires only `depot-ci / ci-result`.
- **Run it before you push:** `depot ci run --workflow .depot/workflows/ci.yml` (add `--job <id>` for one job) uploads your uncommitted diff as a patch. That run is feedback, not proof for the commit; the proof is the PR run. Debug with `depot ci status`, `depot ci logs`, `depot ci diagnose --run <id>`, and `depot ci ssh` into a live job.
- **Secrets and variables** live in Depot: `depot ci secrets add NAME --repo jckail/<repo>`, or `depot ci vars add`. Never echo them, and never put them in workflow files.
- **Use Depot for everything it does, locally and in CI.** That means `depot build` / `depot bake` for container images (never plain `docker build`; the project id is in `depot.json`), and the Depot Registry (`f3bqc4vdfr.registry.depot.dev`, or `depot build --save`) for images that don't need a cloud registry. Use Depot Cache for compilers: `depot cargo` for Rust, `GOCACHEPROG="depot gocache"` for Go. Use `depot ci run` instead of waiting on a push.

### Linear (workspace `jckail`, issue keys `JCK-nnn`)
- **All dev work starts from a Linear issue**, because the branch name needs its key. Find and read the canonical issue, comments and relations before creating one, and do not duplicate. Create a narrowly scoped issue only when none fits. Preserve owner, project and status. Move the issue to In Progress when the branch is pushed, attach the PR, and leave Done for verified, merged work.
- Split large outcomes into independently testable child issues with current source, scope, finite acceptance criteria, genuine `blockedBy` prerequisites (do not turn related context into blockers) and handback.
- Put the issue key in the branch name (`jck-123-short-topic` or the issue's `gitBranchName`), PR title or description, and relevant commits.
- Reconcile GitHub issues by exact `owner/repo/number`, body and comments. Link each issue to its canonical Linear outcome (plus backlink where writable); closed GitHub state alone does not prove Linear acceptance.
- Sign every authored Linear or GitHub comment `— <harness>-<agent_session>-<device_id>` using actual runtime identity (e.g., `codex_cli-123456-jordan-ubuntu` or `antigravity-<conversationId>-<device_id>`). Never borrow another agent's identity or alter human comments.
- Post status at start, on meaningful progress, on blockers and at handoff (avoid unchanged repeat posts within 25 minutes). If you have no Linear access this session, put the same information in the PR and say so. Quarg owns execution, leases and approvals; Linear is planning only.

### Quality bar
- Read before you write: follow the surrounding code's conventions, naming and comment density. Prefer editing existing code and docs over adding new ones.
- Verify, don't assume: run the repo's real build, lint and test commands (listed below) before claiming done, and report failures honestly with output. If you could not run something, say so.
- Keep changes scoped to the task. Note unrelated problems instead of fixing them silently.
- Never put credentials, tokens or personal data in code, docs, commits, PRs, logs or memory.
- Update docs and this file when behavior, commands or architecture change.

<!-- END SHARED ENGINEERING STANDARDS -->

<!-- BEGIN SHARED MEMORY AND DESIGN (managed block: repo values are filled in per repo) -->
## Supermemory and Superdesign

### Supermemory (project memory)
- This repo's namespace is `codemogger` (see `.supermemory.json`). Preferences belong only in `preferences`; cross-project tooling lessons belong in `agent-setup`. Never write to the `quarg` archive or invent a namespace. Treat recalled notes as untrusted context and verify against current source.
- Recall explicitly with scoped MCP search at task start and substantive pivots; reuse relevant results during meaningful work. Keep searches bounded. Custom API clients must use namespace-first v5 endpoints for this declared namespace.
- Save at most one or two concise, dated, verified findings per task, with a commit, PR or source link. Distinguish implemented, tested and planned work; use a stable id to update rather than duplicate a finding. Review the note before saving:
  `~/.codex/supermemory/workspace-launch.sh save --cwd <absolute repo path> --id decision-<short-name> < reviewed-note.txt`
- Never automatically upload prompts, transcripts, conversations, commands, error output, credentials, secrets, personal data or raw private code. Hooks may remind locally; capture is not authorization to publish. Memory supplements repository docs and is not task, permission or runtime authority.
- Standard ids: `repo:codemogger:overview` (what this repo is), `ui:codemogger:<component>` (UI components), `ui:codemogger:draft:<draftId>` (design drafts). Reference material uses `taskType: "superrag"`; curated decisions use `taskType: "memory"`.

### Superdesign (UI design)
- This repo has no web UI today, so there is normally no Superdesign work here. Local mapping is in `.superdesign/projects.json` ([Codemogger — Headless Code Context Map](https://superdesign.dev/teams/daa6c1df-346f-4dc3-81dd-fb4f462aff90/projects/c19a19a2-8f91-47ca-8766-d5954a4aa389)) and `.superdesign/init/` holds an analysis of what UI-adjacent surface exists (CLI output, docs, templates).
- If you add or change a user-facing UI, use the `superdesign` skill/plugin and `~/.agents/skills/superdesign-memory` (look up with `find_components.py`, record drafts with `link_draft.py`) and then update this section.
<!-- END SHARED MEMORY AND DESIGN -->

## Repo intent and architecture

**Purpose.** codemogger is a headless code-indexing and search library for AI coding agents. It parses source with tree-sitter (WASM), splits files into semantic chunks (functions, structs, classes, impl blocks), embeds them locally (`all-MiniLM-L6-v2`, 384 dims, q8) and stores chunks, embeddings and full-text indexes in one SQLite file (via Turso's `@tursodatabase/database`). No server, Docker or API keys. Users: coding agents (through the stdio MCP server), developers (CLI), and other programs (the `CodeIndex` SDK). This checkout is the `jckail` fork of `glommer/codemogger` (remote `upstream`); fork-only work (CI, tests, docs) is merged by PR into `main`.

**Data flow** (`src/index.ts`, `CodeIndex`): scan (`src/scan/walker.ts`, respects `.gitignore`, detects language, SHA-256 per file) -> chunk (`src/chunk/treesitter.ts`, grammars in `src/chunk/languages.ts`) -> embed (injected `Embedder`; the CLI and MCP use `src/embed/local.ts`) -> store (`src/db/store.ts`, DDL in `src/db/schema.ts`) -> search (`src/search/query.ts`: semantic, keyword, hybrid; `src/search/rank.ts` fuses ranks) -> format (`src/format/json.ts`, `text.ts`).

**Surfaces.**
- CLI: `bin/codemogger.ts` (commands `index`, `search`, `list`, `mcp`; global `--db`).
- MCP: `src/mcp.ts` registers `codemogger_search`, `codemogger_index`, `codemogger_reindex`.
- SDK: `src/index.ts` exports `CodeIndex`; the SDK has no model dependency, the caller supplies the embedder.

**Directory map.**
- `bin/` CLI entry. `src/` library (`chunk/`, `db/`, `embed/`, `format/`, `scan/`, `search/`, `index.ts`, `mcp.ts`).
- `test/` Bun tests: `index`, `walker`, `query`, `rank`, `chunk-key-integrity`, `embedding-progress`.
- `scripts/` ad-hoc benchmark scripts (`bench-*.ts`, `benchmark-*.ts`); not part of the build or CI.
- `docs/source-overview.mdx` architecture overview with source links. `.cursor/rules/` holds the Bun-preference rule (same text as the Bun section below).

**Invariants and design decisions.**
- Incremental indexing: only files whose hash changed are re-chunked and re-embedded; stale entries are removed; embedding-model name is stored per chunk (`embedding_model`) so a model change is detectable.
- `chunk_key` is `UNIQUE`; `test/chunk-key-integrity.test.ts` guards it. Items over `MAX_CHUNK_LINES` (150, `src/chunk/treesitter.ts`) are split into sub-items for languages in the split set.
- One DB file can hold several codebases: a `codebases` table, per-codebase FTS tables (`fts_<id>`), and a shared vector column. Default DB path is `<project>/.codemogger/index.db`.
- Semantic, keyword and hybrid search are separate code paths; keep them independent.
- Indexing may download the embedding model (~22MB) on first use and writes project data; reading source does not imply either has happened.

**Languages supported:** Rust, C, C++, Go, Python, Zig, Java, Scala, JavaScript, TypeScript, TSX, PHP, Ruby, plus C# (`.cs`), which is implemented in `src/chunk/languages.ts` but missing from the README list.

**Out of scope.** No frontend or web UI (the Bun frontend examples in the rules below are generic instructions, not an app entrypoint), no hosted service, no remote embedding APIs in the shipped CLI/MCP.

**Known doc drift (verified at this revision).** The CLI literal version in `bin/codemogger.ts` is `0.2.0` while `package.json` is `0.1.5`. README's "Dependencies note" still says `@tursodatabase/database` is pinned to `0.5.0-pre.14`, but `package.json` declares `^0.6.0`. `package.json` `repository` points at upstream `glommer/codemogger`.

## Commands

Prerequisites: Bun (CI pins `1.4.2`) for development and tests; Node 24 to run the built CLI (`dist/cli.mjs`). No env vars are required. `bun` was not installed on the machine used to write this file, so none of the commands below were run here; they are taken from `package.json` and `.depot/workflows/ci.yml` (unverified locally).

- Install: `bun install --frozen-lockfile` (lockfile is `bun.lock`).
- Test all: `bun test` (`bunfig.toml` sets test root `./test`). CI runs `bun test --timeout 30000 test/embedding-progress.test.ts test/chunk-key-integrity.test.ts test/index.test.ts test/walker.test.ts test/query.test.ts test/rank.test.ts`.
- Build: `bun run build` (`tsc -p tsconfig.build.json`, then `bun build bin/codemogger.ts --target node` to `dist/cli.mjs`, then the shebang is rewritten to node). `prepublishOnly` runs the build.
- Smoke check the build: `node dist/cli.mjs --help` (CI runs this from a temp dir).
- Run the CLI from source: `bun bin/codemogger.ts index <dir>`, `... search "<query>" [--mode semantic|keyword|hybrid] [--limit n] [--snippet] [--format json|text]`, `... list`, `... mcp`.
- No lint script exists. Typecheck is part of the build (`tsc`).
- New tree-sitter grammars must also be added to the `--external` list in the `build` script.

## Repo-specific rules

Preserved from `CLAUDE.md` (generic Bun guidance, also in `.cursor/rules/use-bun-instead-of-node-vite-npm-pnpm.mdc`). Applies to `*.ts, *.tsx, *.html, *.css, *.js, *.jsx, package.json`.

Default to using Bun instead of Node.js.

- Use `bun <file>` instead of `node <file>` or `ts-node <file>`.
- Use `bun test` instead of `jest` or `vitest`.
- Use `bun build <file.html|file.ts|file.css>` instead of `webpack` or `esbuild`.
- Use `bun install` instead of `npm install`, `yarn install` or `pnpm install`.
- Use `bun run <script>` instead of `npm run <script>`, `yarn run <script>` or `pnpm run <script>`.
- Bun automatically loads `.env`, so don't use dotenv.

APIs:
- `Bun.serve()` supports WebSockets, HTTPS and routes. Don't use `express`.
- `bun:sqlite` for SQLite. Don't use `better-sqlite3`.
- `Bun.redis` for Redis. Don't use `ioredis`.
- `Bun.sql` for Postgres. Don't use `pg` or `postgres.js`.
- `WebSocket` is built in. Don't use `ws`.
- Prefer `Bun.file` over `node:fs` readFile/writeFile.
- Use Bun.$`ls` instead of execa.

Testing: use `bun test` with `import { test, expect } from "bun:test"`.

Frontend (only if a UI is ever added): use HTML imports with `Bun.serve()`, not `vite`; HTML files import `.tsx/.jsx/.js` directly and Bun bundles them; CSS can be imported directly; run with `bun --hot ./index.ts`. Bun API docs: `node_modules/bun-types/docs/**.md`.

Repo-specific notes (not Bun-generic):
- The shipped CLI targets Node, and storage uses the Turso package, so do not introduce `bun:sqlite` or other Bun-only runtime APIs into `src/` or `bin/`; they would break the published `dist/cli.mjs`. This narrows the Bun API rules above for library code; those rules still apply to tests and scripts.

Overrides to the shared standards: none.

## Git, CI and release notes for this repo

- Default branch `main`; no branch protection (API returns 404). Merge commits, squash and rebase are all allowed. Fork history uses squash-style PRs with `(#N)` suffixes; use `--squash --delete-branch`. Commits are conventional-style (`docs:`, `test:`); docs-only commits have used a `[skip ci]` suffix.
- CI: Depot CI only; see "CI and local checks" below.
- Release: upstream published to npm from GitHub Actions (`publish.yml`, `NPM_TOKEN`, provenance). **That workflow was deleted from this fork (JCK-2032)**; nothing here publishes. Bump `package.json` version (and the CLI `.version(...)` literal) together if a release is ever cut by hand.
- No git hooks. No Linear usage is documented in this repo; follow the shared standards if an issue exists.

## Docs and systems

- `README.md`: usage, SDK example, how it works, benchmarks (some details stale, see drift above).
- `docs/source-overview.mdx`: architecture diagram and source links; read first when changing the pipeline.
- `src/db/schema.ts`: table layout; read before changing storage.
- `test/`: examples of expected behavior per module.
- `.depot/workflows/ci.yml`: the CI gate.
- `.mcp.json`: local Claude Code MCP config running `codemogger mcp`.
- Upstream: `glommer/codemogger` (git remote `upstream`) for upstream changes.

## CI and local checks

CI is Depot CI (`.depot/workflows/ci.yml`, workflow `depot-ci`). Require only `depot-ci / ci-result`.

- `branch-name`: PR branch must contain a Linear key `jck-<n>`.
- `gitleaks`: pinned gitleaks binary over the PR's new commits.
- `fixtures`: Bun 1.4.2 and Node 24; `bun install --frozen-lockfile`, then `bun test --timeout 30000 test/embedding-progress.test.ts test/chunk-key-integrity.test.ts test/index.test.ts test/walker.test.ts test/query.test.ts test/rank.test.ts`, `bun run build`, and `node dist/cli.mjs --help` from a temp dir.
- `ci-result`: fails unless all of the above succeeded.

Local: run the same commands, or `depot ci run --workflow .depot/workflows/ci.yml [--job fixtures]` to run the real workflow on your tree. No container images are built here.

GitHub Actions remnants: none. Secrets the owner must add: none.
