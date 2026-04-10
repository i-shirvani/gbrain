# CLAUDE.md

GBrain is a personal knowledge brain. Postgres + pgvector + hybrid search in a managed Supabase instance.

## Architecture

Contract-first: `src/core/operations.ts` defines ~30 shared operations. CLI and MCP
server are both generated from this single source. Skills are fat markdown files
(tool-agnostic, work with both CLI and plugin contexts).

## Key files

### Core
- `src/core/operations.ts` — Contract-first operation definitions (the foundation, ~29 ops)
- `src/core/engine.ts` — Pluggable engine interface (BrainEngine)
- `src/core/postgres-engine.ts` — Postgres + pgvector implementation
- `src/core/db.ts` — Connection management, schema initialization
- `src/core/types.ts` — All shared TypeScript types (Page, Chunk, SearchResult, Link, etc.)
- `src/core/config.ts` — Config load/save (`~/.gbrain/config.json`), env var precedence
- `src/core/import-file.ts` — importFromFile + importFromContent (chunk + embed + tags)
- `src/core/sync.ts` — Pure sync functions (manifest parsing, filtering, `slugifyPath`)
- `src/core/markdown.ts` — Markdown + frontmatter parsing (gray-matter → PageInput)
- `src/core/migrate.ts` — Embedded schema migration runner (runs on initSchema)
- `src/core/embedding.ts` — OpenAI text-embedding-3-large, batch, retry, backoff
- `src/core/storage.ts` — Pluggable storage interface (StorageBackend)
- `src/core/storage/` — Storage backends: `local.ts`, `s3.ts`, `supabase.ts`
- `src/core/supabase-admin.ts` — Supabase admin API (project discovery, pgvector check)
- `src/core/file-resolver.ts` — MIME detection, content hashing for file uploads
- `src/core/yaml-lite.ts` — Minimal YAML parser for .supabase markers and redirects
- `src/core/index.ts` — Core library exports
- `src/core/chunkers/` — 3-tier chunking: `recursive.ts`, `semantic.ts`, `llm.ts`
- `src/core/search/` — Hybrid search: `vector.ts`, `keyword.ts`, `hybrid.ts`, `expansion.ts`, `dedup.ts`

### Commands (business logic, separate from CLI wiring)
- `src/commands/sync.ts` — Sync a git repo to brain (performSync)
- `src/commands/doctor.ts` — Health diagnostics (runDoctor)
- `src/commands/check-update.ts` — GitHub release check + semver comparison
- `src/commands/upgrade.ts` — Self-update (bun/binary/clawhub detection)
- `src/commands/import.ts`, `init.ts`, `embed.ts`, `export.ts`, `files.ts`, `config.ts`, `call.ts`, `serve.ts`, `tools-json.ts`

### Entry points & manifests
- `src/cli.ts` — CLI entry point (thin dispatcher to commands)
- `src/mcp/server.ts` — MCP stdio server (generated from operations)
- `src/version.ts` — Version constant (mirrors `VERSION` file)
- `VERSION` — Plain text version (currently `0.5.1`)
- `src/schema.sql` — Full Postgres + pgvector DDL (includes files table)
- `openclaw.plugin.json` — ClawHub bundle plugin manifest
- `skills/manifest.json` — Skill registry (name, path, description)

### Docs
- `docs/GBRAIN_SKILLPACK.md` — Full agent skillpack (Sections 1–17)
- `docs/GBRAIN_RECOMMENDED_SCHEMA.md` — Recommended brain schema directories
- `docs/GBRAIN_VERIFY.md` — End-to-end installation verification runbook
- `docs/ENGINES.md` — Engine comparison and selection guide
- `docs/SQLITE_ENGINE.md` — SQLite engine documentation
- `docs/GBRAIN_V0.md` — Migration guide from V0

## Commands

Run `gbrain --help` or `gbrain --tools-json` for full command reference.

Key CLI commands:
- `gbrain init` — Initialize database + config
- `gbrain import <path>` — Bulk import files (parallel, resumable)
- `gbrain sync` — Incremental sync from git repo
- `gbrain query <text>` — Hybrid search (vector + keyword + expansion)
- `gbrain doctor` — Health diagnostics
- `gbrain check-update` — Check for new versions
- `gbrain upgrade` — Self-update
- `gbrain serve` — Start MCP stdio server
- `gbrain files migrate` — Move files between storage backends

## Testing

`bun test` runs all tests (20 unit test files + 5 E2E test files). Unit tests run
without a database. E2E tests skip gracefully when `DATABASE_URL` is not set.

Unit tests: `test/markdown.test.ts` (frontmatter parsing), `test/chunkers/recursive.test.ts`
(chunking), `test/sync.test.ts` (sync logic), `test/parity.test.ts` (operations contract
parity), `test/cli.test.ts` (CLI structure), `test/config.test.ts` (config redaction),
`test/files.test.ts` (MIME/hash), `test/import-file.test.ts` (import pipeline),
`test/upgrade.test.ts` (schema migrations), `test/doctor.test.ts` (doctor command),
`test/file-migration.test.ts` (file migration), `test/file-resolver.test.ts` (file resolution),
`test/import-resume.test.ts` (import checkpoints), `test/migrate.test.ts` (migration),
`test/setup-branching.test.ts` (setup flow), `test/slug-validation.test.ts` (slug validation),
`test/storage.test.ts` (storage backends), `test/supabase-admin.test.ts` (Supabase admin),
`test/yaml-lite.test.ts` (YAML parsing), `test/check-update.test.ts` (version check + update CLI).

E2E tests (`test/e2e/`): Run against real Postgres+pgvector. Require `DATABASE_URL`.
- `bun run test:e2e` runs Tier 1 (mechanical, all operations, no API keys)
- `test/e2e/mechanical.test.ts` — all 29 operations, concurrency, fixtures
- `test/e2e/mcp.test.ts` — MCP tool generation and protocol verification
- `test/e2e/sync.test.ts` — git repo sync pipeline
- `test/e2e/upgrade.test.ts` — check-update E2E against real GitHub API (network required)
- Tier 2 (`skills.test.ts`) requires OpenClaw + API keys, runs nightly in CI
- If `.env.testing` doesn't exist in this directory, check sibling worktrees for one:
  `find ../  -maxdepth 2 -name .env.testing -print -quit` and copy it here if found.
- Always run E2E tests when they exist. Do not skip them just because DATABASE_URL
  is not set. Start the test DB, run the tests, then tear it down.
- E2E fixtures live in `test/e2e/fixtures/` (apple-notes, companies, concepts, deals,
  large, meetings, people, projects, sources)

### API keys and running ALL tests

ALWAYS source the user's shell profile before running tests:

```bash
source ~/.zshrc 2>/dev/null || true
```

This loads `OPENAI_API_KEY` and `ANTHROPIC_API_KEY`. Without these, Tier 2 tests
skip silently. Do NOT skip Tier 2 tests just because they require API keys — load
the keys and run them.

When asked to "run all E2E tests" or "run tests", that means ALL tiers:
- Tier 1: `bun run test:e2e` (mechanical, sync, upgrade — no API keys needed)
- Tier 2: `test/e2e/skills.test.ts` (requires OpenAI + Anthropic + openclaw CLI)
- Always spin up the test DB, source zshrc, run everything, tear down.

### E2E test DB lifecycle (ALWAYS follow this)

You are responsible for spinning up and tearing down the test Postgres container.
Do not leave containers running after tests. Do not skip E2E tests.

1. **Check for `.env.testing`** — if missing, copy from sibling worktree.
   Read it to get the DATABASE_URL (it has the port number).
2. **Check if the port is free:**
   `docker ps --filter "publish=PORT"` — if another container is on that port,
   pick a different port (try 5435, 5436, 5437) and start on that one instead.
3. **Start the test DB:**
   ```bash
   docker run -d --name gbrain-test-pg \
     -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
     -e POSTGRES_DB=gbrain_test \
     -p PORT:5432 pgvector/pgvector:pg16
   ```
   Wait for ready: `docker exec gbrain-test-pg pg_isready -U postgres`
4. **Run E2E tests:**
   `DATABASE_URL=postgresql://postgres:postgres@localhost:PORT/gbrain_test bun run test:e2e`
5. **Tear down immediately after tests finish (pass or fail):**
   `docker stop gbrain-test-pg && docker rm gbrain-test-pg`

Never leave `gbrain-test-pg` running. If you find a stale one from a previous run,
stop and remove it before starting a new one.

A `docker-compose.test.yml` is also available for convenience: `docker compose -f docker-compose.test.yml up -d`.

## Skills

Read the skill files in `skills/` before doing brain operations. They contain the
workflows, heuristics, and quality rules for ingestion, querying, maintenance,
enrichment, and setup. 8 skills: ingest, query, maintain, enrich, briefing,
migrate, setup, install.

Skill registry: `skills/manifest.json` (version, paths, descriptions).
Migrations for post-upgrade agent directives: `skills/migrations/`.

## Build

`bun build --compile --outfile bin/gbrain src/cli.ts`

Cross-platform builds:
`bun run build:all` produces `bin/gbrain-darwin-arm64` and `bin/gbrain-linux-x64`.

## Pre-ship requirements

Before shipping (/ship) or reviewing (/review), always run the full test suite:
- `bun test` — unit tests (no database required)
- Follow the "E2E test DB lifecycle" steps above to spin up the test DB,
  run `bun run test:e2e`, then tear it down.

Both must pass. Do not ship with failing E2E tests. Do not skip E2E tests.

## CHANGELOG voice

CHANGELOG.md is read by agents during auto-update (Section 17). The agent summarizes
the changelog to convince the user to upgrade. Write changelog entries that sell the
upgrade, not document the implementation.

- Lead with what the user can now DO that they couldn't before
- Frame as benefits and capabilities, not files changed or code written
- Make the user think "hell yeah, I want that"
- Bad: "Added GBRAIN_VERIFY.md installation verification runbook"
- Good: "Your agent now verifies the entire GBrain installation end-to-end, catching
  silent sync failures and stale embeddings before they bite you"
- Bad: "Setup skill Phase H and Phase I added"
- Good: "New installs automatically set up live sync so your brain never falls behind"

## Version migrations

Create a migration file at `skills/migrations/v[version].md` when a release
includes changes that existing users need to act on. The auto-update agent
reads these files post-upgrade (Section 17, Step 4) and executes them.

**You need a migration file when:**
- New setup step that existing installs don't have (e.g., v0.5.0 added live sync,
  existing users need to set it up, not just new installs)
- New SKILLPACK section with a MUST ADD setup requirement
- Schema changes that require `gbrain init` or manual SQL
- Changed defaults that affect existing behavior
- Deprecated commands or flags that need replacement
- New verification steps that should run on existing installs
- New cron jobs or background processes that should be registered

**You do NOT need a migration file when:**
- Bug fixes with no behavior changes
- Documentation-only improvements (the agent re-reads docs automatically)
- New optional features that don't affect existing setups
- Performance improvements that are transparent

**The key test:** if an existing user upgrades and does nothing else, will their
brain work worse than before? If yes, migration file. If no, skip it.

Write migration files as agent instructions, not technical notes. Tell the agent
what to do, step by step, with exact commands. See `skills/migrations/v0.5.0.md`
for the pattern.

## Schema state tracking

`~/.gbrain/update-state.json` tracks which recommended schema directories the user
adopted, declined, or added custom. The auto-update agent (SKILLPACK Section 17)
reads this during upgrades to suggest new schema additions without re-suggesting
things the user already declined. The setup skill writes the initial state during
Phase C/E. Never modify a user's custom directories or re-suggest declined ones.

## GitHub Actions SHA maintenance

All GitHub Actions in `.github/workflows/` are pinned to commit SHAs. Before shipping
(`/ship`) or reviewing (`/review`), check for stale pins and update them:

```bash
for action in actions/checkout oven-sh/setup-bun actions/upload-artifact actions/download-artifact softprops/action-gh-release gitleaks/gitleaks-action; do
  tag=$(grep -r "$action@" .github/workflows/ | head -1 | grep -o '#.*' | tr -d '# ')
  [ -n "$tag" ] && echo "$action@$tag: $(gh api repos/$action/git/ref/tags/$tag --jq .object.sha 2>/dev/null)"
done
```

If any SHA differs from what's in the workflow files, update the pin and version comment.

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.
The skill has specialized workflows that produce better results than ad-hoc answers.

**NEVER hand-roll ship operations.** Do not manually run git commit + push + gh pr
create when /ship is available. /ship handles VERSION bump, CHANGELOG, document-release,
pre-landing review, test coverage audit, and adversarial review. Manually creating a PR
skips all of these. If the user says "commit and ship", "push and ship", "bisect and
ship", or any combination that ends with shipping — invoke /ship and let it handle
everything including the commits. If the branch name contains a version (e.g.
`v0.5-live-sync`), /ship should use that version for the bump.

Key routing rules:
- Product ideas, "is this worth building", brainstorming → invoke office-hours
- Bugs, errors, "why is this broken", 500 errors → invoke investigate
- Ship, deploy, push, create PR, "commit and ship", "push and ship" → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Update docs after shipping → invoke document-release
- Weekly retro → invoke retro
- Design system, brand → invoke design-consultation
- Visual audit, design polish → invoke design-review
- Architecture review → invoke plan-eng-review
- Save progress, checkpoint, resume → invoke checkpoint
- Code quality, health check → invoke health
