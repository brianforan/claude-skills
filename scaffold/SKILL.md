---
name: scaffold
description: Start a new project the user's standard way. Interviews for purpose, shape and stack (FastAPI or Node/TypeScript API, React frontend, Postgres, Elastic logging), then generates a consistent layout, configs, local infra, README and first decision record. Also the source of their coding standards: use it to check or align an existing project with them.
---

# scaffold

Every project the user starts should look the same: same layout, same tooling, same logging, same docs. Then any project is familiar at a glance and to any agent working in it. The `references/` files *are* the user's standards. Treat them as decisions already made, not suggestions to re-debate. When a reference and your general knowledge disagree, the reference wins. If it looks outdated or wrong, say so and propose an edit to the reference file rather than silently deviating.

## Modes

- **New project:** run the interview, then build.
- **Align an existing project:** read `references/conventions.md` plus the stack files that match what the project uses, compare, and propose changes as a list. Don't restructure without a yes: moving files breaks imports and history.
- **Question about standards** ("how do I name tables?"): answer from the relevant reference.

## Interview

Ask in rounds, so each answer can shape the next question. Use the AskUserQuestion tool for multiple-choice rounds where available. Keep each round short.

**Round 1, purpose (free text).** Ask what the project does and who it's for, in a sentence or two, and for a name. Suggest a kebab-case name if they don't have one. This seeds the README intro.

**Round 2, shape and stack.**
- Shape: full-stack web app / API only / frontend only / CLI or script
- Backend (if any): Python + FastAPI / Node + TypeScript (Fastify)
- Frontend (if any): React (Vite + TS)
- Database: Postgres / none

The options come from the files in `references/backend/`, `references/frontend/` and `references/data/`. List what's there. If the user wants something without a reference file, flag it as off-standard. Offer to write a new reference for it first (the better long-term choice), or proceed with your best judgment and note the gap.

**Round 3, operations.**
- Logging: Elastic local stack (default: structured logs + Elasticsearch/Kibana in Docker) / structured logs to stdout only for now
- Visibility: public / private / undecided
- License (only if public): MIT / Apache-2.0 / none

**Round 4, confirm.** Show a short summary: name, purpose, stack, folder location, and the tree you'll create. Ask where to create it if not obvious; default to the parent folder of their other projects. Build on a yes.

Don't ask *why* they chose a stack. Rationale in the repo comes from the reference files' technical reasons. **Personal motivations never go into committed files.** If the user mentions one ("I want to learn Elastic", "it's popular with employers"), it can go in `.private/` if they want it recorded, never in decision records, the README, comments or commit messages.

## Build

1. **Read the references** for the chosen stack: `conventions.md` always, plus each chosen `backend/`, `frontend/` and `data/` file, and `observability/elastic.md` if logging uses Elastic.
2. **Check current versions** before writing anything that pins them (Node LTS, Python, Postgres major, Elastic stack, key packages). Use `npm view <pkg> version`, `uv`/`pip index versions`, or the project's release page. Memory goes stale; references deliberately avoid hard-coding versions.
   - **Check peer ranges too:** `npm view <pkg> peerDependencies`. The newest version of one tool often isn't supported by another yet (e.g. a TypeScript major that typescript-eslint doesn't accept). Pin the newest *compatible* version and say why.
   - **Runtimes:** use the runtime already installed if it's still in support (e.g. Node 24 while 26 is the newest LTS). Tell the user when a newer LTS exists rather than requiring it.
3. **Prefer official generators, then conform.** For example `npm create vite@latest web -- --template react-ts` or `uv init`. They track upstream changes. Then reshape the output to match the references, and delete template leftovers the standard doesn't use (template README, nested `.gitignore`, config for tools you replaced, demo assets). When a generator's defaults differ from a reference (a different linter, a newer compiler), follow the reference, then **tell the user about the drift** at the end. It may mean the standard should change.
4. **Lay down the shared skeleton** from `conventions.md`: root README, `.env.example`, root `compose.yaml` (from `assets/compose.yaml`, keeping only the services chosen), `infra/filebeat.yml` and `.env.example` (from `assets/`), `docs/decisions/`, `.editorconfig`, logs dir ignored.
5. **Build each app** per its reference: structure, config loading, logging setup, health endpoint, one example route/page, one passing test, lint/format config.
6. **Write `docs/decisions/0001-initial-stack.md`** using `~/.claude/skills/git-cleanup/assets/decision-record.md`, with the technical rationale from the references. Technical reasons only, as above.
7. **Verify cheaply:** install deps, run lint, run tests, start the API and hit `/health`, build the frontend. Check `docker compose version` first. If Docker isn't installed, tell the user Postgres and the Elastic stack need Docker Desktop (or Podman), and verify only what runs without it. Don't pull Docker images without asking (the Elastic images are several hundred MB). Offer `docker compose up` as the user's next step, or run it if they say yes.
8. **Make it safe to publish:** follow the `git-cleanup` skill (scan, ignore rules, `git init`, pre-commit guard, identity). Don't commit unless the user asks.
9. **Report:** the tree, how to run everything (copy of the README "Getting started"), what was verified, and anything off-standard.

## Keeping the standards alive

When the user corrects a convention during a build ("actually I want pnpm"), ask whether it's a one-off or the new standard. If it's the new standard, edit the reference file so the next project gets it. That's how this skill stays the single source of truth.
