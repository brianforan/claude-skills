# Decision logs, private notes and AI-assistant files

## The test

For any note, doc or instruction file, ask: **is this about the project, or about me?**

- **About the project → publish.** Trade-offs, alternatives considered, benchmarks, constraints of the problem, what went wrong and what was learned. This shows engineering judgment and is often the most valuable thing in a portfolio repo.
- **About the person → private.** Money (rates, salary, costs of their own accounts), employer and client identities, job hunting, negotiations, health or time pressure, opinions about specific people, frustrations, legal/contract details, anything that would read badly quoted out of context.
- **Personal motivations are always private**, even mild ones: learning goals, career positioning, "chose X to get familiar with industry tooling", "wanted to try Y". The user has asked that these never be committed. A decision record states the technical rationale. If the only real reason was personal, record the decision and its consequences without a "why", and keep the motivation in `.private/` if they want it at all.

Most real notes mix both. The usual answer is to **split**, not choose.

## Decision records (show the dev process)

Public decisions go in `docs/decisions/NNNN-short-title.md` using `assets/decision-record.md` (a lightweight ADR: Context, Decision, Alternatives, Consequences). Number sequentially. Never edit a decided record's substance; supersede it with a new one.

Sanitizing example:

> **Private original:** "Going with SQLite instead of Postgres because Acme won't pay for a managed DB and I'm out of runway until the Henderson invoice clears in March."
>
> **Public record:** "Context: single-node deployment, low write volume, minimal operational overhead wanted. Decision: SQLite with WAL mode. Alternatives: managed Postgres (rejected: cost and ops overhead outweigh benefits at current scale). Consequences: revisit if concurrent writers or multi-node deployment become requirements."

Keep the reasoning, remove who/where/how-much/for-whom. Show the user the proposed sanitized version before writing it. Their voice and their call.

## Where private material lives

Offer these; the default is (a):

a. **`.private/` in the project, never committed.** Easy to keep next to the code, which is why it's the default. `.private/` is in the base `.gitignore` (the name is generic, so the entry reveals nothing). Risk: `git add -f` or someone editing `.gitignore`. The pre-commit hook catches private terms but not every personal sentence. Adding it to the user's global excludes file too gives belt and braces.
b. **Outside the repo** (`~/notes/<project>/`). Zero risk of committing. Slightly less convenient.
c. **Separate private repo** (e.g. `<project>-notes`) if they want history and sync for their notes.

Move files, don't delete them, and tell the user where each one went.

Typical candidates: `NOTES.md`, `TODO.md`, `journal/`, `devlog.md`, `scratch*`, `ideas.md`, meeting notes, client correspondence, invoices and estimates, `docs/` drafts with names in them.

## AI-assistant files

| File / dir | Default | Notes |
|---|---|---|
| `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.github/copilot-instructions.md`, `.cursor/rules/`, `.windsurfrules`, `.cursorrules` | Commit, after scrubbing | Project instructions are normal to share now and help contributors. Remove personal paths, names, personal preferences ("I like terse answers") and anything about the user's job or clients. |
| `CLAUDE.local.md` | Ignore | Personal per-project instructions. That's what it's for. |
| `~/.claude/CLAUDE.md` (user-level) | Not in repo | Where personal preferences belong. Suggest moving them here. |
| `.claude/settings.json` | Commit if useful | Shared permissions and hooks. Check for absolute paths and `env` values. |
| `.claude/settings.local.json` | Ignore | Personal permissions; often contains paths. |
| `.claude/commands/`, `.claude/agents/`, `.claude/skills/` | Commit if project-relevant | Scrub like any other doc. |
| `.mcp.json`, `.cursor/mcp.json` | Commit only without secrets | Use `${ENV_VAR}` expansion for tokens and paths; otherwise ignore it. |
| Agent memory, transcripts, chat exports, plans (`.specstory/`, `.aider*` history, exported chats, `memory/` dirs written by agents) | Ignore | These routinely contain verbatim secrets, personal details and half-formed opinions. |

Cross-tool tip: keep shared instructions in `AGENTS.md` (read by many agents) and have `CLAUDE.md` import it with a line `@AGENTS.md`, plus any Claude-specific additions.

**AI disclosure** (e.g. `Co-Authored-By` trailers, a README note): a values choice, not a security one. Mention it if relevant and let the user decide; don't add or strip it unilaterally.
