# Scaffolding the basics

> The user's own standards live in the `scaffold` skill (`~/.claude/skills/scaffold/references/conventions.md` and its stack files). Where they cover something, they win over this file. This file is the generic fallback for projects outside those stacks.

Keep this minimal. The goal is a repo a stranger can understand and run, not a template showcase. For existing projects, **propose** structural moves and don't make them without a yes: moving files breaks imports, paths and muscle memory.

## Layout

Follow the stack's idiom first; generic structure second.

- Python: `pyproject.toml`, `src/<package>/`, `tests/`
- Node/TS: `package.json`, `src/`, `test/` or co-located `*.test.ts`
- Go: `go.mod`, `cmd/<app>/`, `internal/`
- Rust: Cargo defaults (`src/`, `tests/`, `examples/`)
- .NET: `src/<Project>/`, `tests/<Project>.Tests/`, `.sln` at root

Common to all:
```
README.md
LICENSE                 # if public
.gitignore
.gitattributes
.env.example            # if the app reads env vars
docs/
  decisions/            # public decision records (assets/decision-record.md)
scripts/                # dev/ops helpers, parameterized, no hardcoded hosts or paths
```
Local-only (ignored): `.env`, `.private/`, `*.local.*`, `CLAUDE.local.md`.

## .gitignore

Start from `assets/gitignore-base`, then add the stack's standard patterns (the github/gitignore templates for the language and framework). If a `.gitignore` exists, merge: add what's missing and don't reorder or delete their entries. Revealing entries go in `.git/info/exclude` instead.

Already-tracked files aren't affected by new ignore rules. Untrack them with `git rm --cached <path>` (keeps the local file) and tell the user they're still in history if that matters (see `secrets.md`).

## .gitattributes

Copy `assets/gitattributes`. It normalizes line endings (important on Windows, where CRLF/LF churn creates noisy diffs) and marks binaries. On an existing repo, after adding it, `git add --renormalize .` shows any files whose endings change. Mention that, since it can produce a large one-time diff.

## README skeleton

```markdown
# <Project name>

One or two sentences: what it is and who it's for.

## Quick start
<prerequisites>
<install/run commands>

## Configuration
Copy `.env.example` to `.env` and fill in values. | Variable | Purpose | Required |

## Project structure
Brief tour of top-level dirs (only if non-obvious).

## Design decisions
See [docs/decisions](docs/decisions/).

## License
<name>, see [LICENSE](LICENSE).
```
No personal contact details unless the user wants them. Point to the forge's issues instead.

## LICENSE

Ask, and explain briefly: **no license means all rights reserved.** People can view the code but can't legally reuse it. Common choices: MIT (permissive, short), Apache-2.0 (permissive plus patent grant), GPL-3.0 / AGPL-3.0 (copyleft). For a portfolio piece they don't want copied, no license is a legitimate choice. The copyright holder line is a personal-info decision (see `personal-info.md`).

## Public-repo extras (only when public and only if wanted)

- `SECURITY.md`: how to report vulnerabilities. Point to GitHub private vulnerability reporting rather than a personal email.
- `CONTRIBUTING.md` and an issue template: only if they actually want contributions.
- Dependabot (`.github/dependabot.yml`) for dependency updates.

## Size and binaries

- GitHub warns at 50 MB per file and rejects 100 MB. Use Git LFS for large assets that must be versioned, or keep them out (download script, release assets).
- Don't commit build output, dependency dirs, datasets with real people's data, or generated files that can be rebuilt.

## Default branch

`main` is the current convention (`git config --global init.defaultBranch main` is global: ask). For an existing `master` repo, renaming is optional and affects remotes. Mention it, don't push it.
