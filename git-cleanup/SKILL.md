---
name: git-cleanup
description: Make a new or existing project safe to put in git and publish. Finds and fixes secrets, personal info (names, emails, home paths, employers), deployment details, private notes and AI files (CLAUDE.md, .claude/, .mcp.json); scaffolds .gitignore, .env.example, README, LICENSE and a pre-commit guard. Use when initializing a repo, first pushing to GitHub, making a repo public, or checking what would leak.
---

# git-cleanup

Get a project to a state where everything git will publish is something the user would be comfortable with a stranger, recruiter, client or future employer reading, forever, and where the basics are in place so it stays that way.

Why it's worth care: commits are permanent (deleting a file later doesn't remove it from history), public GitHub repos are scraped for keys within minutes of a push, and the most embarrassing leaks usually aren't keys at all: a home path with a real name in it, a client's name in a decision note, a server IP in a deploy script, a CLAUDE.md full of personal preferences.

## Principles

- **Rotate, don't just delete.** A secret that was ever pushed is compromised; removing it from history only limits future exposure.
- **Separate, don't destroy.** Private material moves somewhere outside the published surface; it isn't deleted. Deleting the user's notes is never the fix.
- **Show the process, hide the person.** Engineering reasoning (trade-offs, alternatives, benchmarks) is valuable to publish. Who, where, how much, for whom and why-personally is not.
- **The ignore file is public too.** `.gitignore` is committed, so an entry like `notes-on-acme-contract.md` leaks the thing it hides. Revealing names go in `.git/info/exclude` (local, never committed) or the user's global excludes file.
- **The user decides what's personal.** The scan finds candidates; the user rules on judgment calls (decision notes, AI instruction files, LICENSE name, whether to show the deploy setup).
- **Ask before anything hard to undo or outward-facing:** rewriting history, deleting or moving files, changing global git config, committing, pushing, changing repo visibility. Additive, local changes (new `.gitignore` entries, `.env.example`, hook in `.git/hooks`) are fine to make once the plan is agreed.

## Workflow

### 1. Orient

Work out, mostly from the filesystem and git:
- New project or existing? Already a git repo? Has commits? Has a remote, and has anything been pushed (`git branch -r`, `git log --oneline @{u} 2>/dev/null`)?
- Target visibility: public, private, or undecided. Ask if it isn't clear. Treat undecided as public: private repos get made public later, get shared, get forked.
- Stack (for language-specific ignore patterns and layout conventions).

If it isn't a repo yet, don't `git init` until the scan and ignore rules are done. That way the first commit is already clean and there's no history to rewrite.

### 2. Collect identity terms

No search can guess the user's employer or city. Collect the obvious ones yourself (git `user.name` / `user.email`, OS username, hostname; see `references/scan-checklist.md` §4), then ask the user, in one message, for anything else they wouldn't want public, for example:
- full name, nicknames, other handles/usernames
- personal and work emails
- employer, clients, coworkers' names, internal project codenames
- city/address, personal domains, server names or IPs

Save them, including the auto-detected ones, one per line to `.git/info/private-terms`. It's inside `.git`, so never committed, and the pre-commit hook reads it. If there's no repo yet, keep the list in mind for the scan and write the file right after `git init`. Don't echo the list into project files or docs.

### 3. Scan

Use your own search tools (Grep/Glob, `git grep`, `git log`); nothing needs installing. Work through `references/scan-checklist.md` in order: what would be published → sensitive file names → secret patterns → personal info → deployment details → history (whenever commits exist) → repo hygiene. It has the patterns, the usual false alarms, and the commands for history.

**Keep secret values out of the conversation:** for secret patterns, list `path:line` only, never the matched value. The checklist shows how.

If `gitleaks` is installed, run `gitleaks dir <project-dir> --redact` (and `gitleaks git --redact` for history) too. It has hundreds of provider-specific rules. Suggest installing it if it's missing, but don't depend on it.

Pattern matches are a net, not proof. Also read the judgment-heavy files yourself: notes, decision records, AI instruction files, READMEs and docs. "We chose Postgres because my manager at Acme insisted" won't match any pattern.

### 4. Triage and present a plan

Group findings for the user, highest stakes first:

| Bucket | Typical items | Default action |
|---|---|---|
| Secrets | API keys, tokens, private keys, credential files, `.tfstate`, creds in URLs | Move to env/ignored file, add `.env.example`; rotate anything ever pushed |
| Personal info | names, emails, home paths, hostnames, employer/clients, EXIF GPS, commit author email | Replace with relative paths / roles / fake data; fix commit identity |
| Deployment | server IPs/hosts, SSH targets, deploy scripts, provider configs, account IDs | Parameterize and commit a template; keep the real one local |
| Private notes and decisions | journals, TODOs, decision logs with personal reasoning | Split: sanitized decision record public, personal notes private |
| AI meta | CLAUDE.md, AGENTS.md, `.claude/`, `.cursor/`, `.mcp.json`, agent memory/transcripts | Keep shared project instructions, move personal ones local, ignore state |
| Hygiene | missing ignore rules, `.gitattributes`, README, LICENSE, large files | Scaffold |

For each item give the concrete proposed change. Get the user's go-ahead on the plan, and get individual answers on the judgment calls, before editing.

### 5. Remediate

Read the reference file for each bucket that has findings:

- Secrets, history rewrites, rotation: `references/secrets.md`
- Personal info, commit identity, file metadata: `references/personal-info.md`
- Deployment and infrastructure details: `references/deployment.md`
- Decision logs, private notes, AI-assistant files: `references/private-vs-public.md`
- Directory structure, README, LICENSE, ignore/attributes files: `references/scaffold.md`

Templates in `assets/`: `gitignore-base` (merge it with the stack's standard patterns, don't overwrite an existing `.gitignore`), `gitattributes`, `decision-record.md`, `pre-commit`.

When editing code to remove a secret or path, keep the program working: read from an env var with a clear error if missing, and document the variable in `.env.example` and the README.

### 6. Guardrails

- **Pre-commit guard.** Copy `assets/pre-commit` into the repo's hooks dir (check `git config core.hooksPath` first; if husky, lefthook or the pre-commit framework is in use, add a step to that instead of overwriting their hook). It's plain `sh` + `git`, so it needs nothing else installed. It blocks commits that stage sensitive file names, high-confidence secrets or terms from `.git/info/private-terms`, and runs gitleaks too when present. It lives in `.git/hooks`, so it's local and never published.
- **Commit identity.** Recommend the forge's no-reply email for this repo (`references/personal-info.md`). Set it per-repo with the user's OK; changing `--global` config needs an explicit yes.
- **Forge settings** (tell the user; you can't click these for them): GitHub secret scanning and push protection, "Keep my email addresses private" and "Block command line pushes that expose my email".
- Optionally suggest a global excludes file for `.private/`, `CLAUDE.local.md` and OS junk so every project is covered (global config, so ask).

### 7. Verify and report

- Repeat the scan for the categories that had findings. Everything left should be fixed or consciously accepted. Accepted values go in `.git/info/git-cleanup-allow`, one literal per line, or a `gitleaks:allow` comment on the line.
- Show exactly what will be published: `git add --dry-run -A` (or `git status --short`), plus `git status --short --ignored` so the user sees what's being held back.
- Report: what changed, what was kept private and where it lives now, and what's left for the user: keys to rotate, LICENSE choice, forge settings, history rewrite if they deferred it.

Don't commit or push unless the user asks. If they do, a fresh repo's first commit is a good moment to double-check `git config user.email`.

## Out of scope

Code style, linters, formatters, commit message conventions, CI test pipelines and branch strategy belong to a standards skill (the `scaffold` skill in this repo). This one stays focused on what gets published and the minimum structure around it. If the user wants those, point that out rather than expanding here.
