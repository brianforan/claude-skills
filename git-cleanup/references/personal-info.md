# Personal information

## What counts

| Kind | Examples | Usual hiding spots |
|---|---|---|
| Identity | real name, handles, personal/work email, phone | commit author, LICENSE, package manifest `author`, README "contact", code comments (`// TODO(name)`), docstrings |
| Machine | home paths (`C:\Users\name\…`, `/home/name`, `/Users/name`), hostname, LAN IPs, MAC addresses | lockfiles, logs, stack traces, `.vscode/settings.json`, launch configs, scripts, AI instruction files, tests with absolute paths |
| Location | city, address, timezone + workplace combos, GPS | EXIF in photos and screenshots, sample data, README |
| Relationships | employer, clients, coworkers, internal codenames | decision notes, commit messages, branch names, fixtures, comments |
| Accounts | cloud account IDs, Discord/Slack IDs, Stripe account IDs, analytics IDs | config, deploy files, ARNs |
| File metadata | Office doc author/company, PDF author, image EXIF | `docs/`, `assets/` |

## Fix patterns

- **Paths:** make them relative to the repo, `~`/`$HOME`/`%USERPROFILE%`, or an env var. In docs write `<repo>/…` or `~/…`.
- **Names in content:** replace with roles ("the client", "the reviewer") or obviously fictional ones. Keep meaning, drop identity.
- **Sample and test data:** use reserved fake values so nothing real slips in: `example.com` / `.test` domains (RFC 2606), IPs `192.0.2.x`, `198.51.100.x`, `203.0.113.x` (RFC 5737), phone `555-0100`–`555-0199`, a faker library for bulk data.
- **Images:** strip metadata with `exiftool -all= file.jpg` (or re-export or screenshot). Check screenshots for visible tabs, usernames, notifications and file paths.
- **Office/PDF:** clear document properties (Word: File → Info → Inspect Document; `exiftool -all=` works on PDFs).
- **TODO tags:** `TODO(name)` → `TODO:` or a handle the user is happy to show.
- **Package manifests:** `author` / `authors` / `maintainers` fields are a deliberate public choice. Ask: real name, handle, or omit.
- **LICENSE:** the copyright line needs a holder. Ask whether that's their legal name, a handle, or "The <project> contributors". (A pseudonym is weaker if they ever need to enforce the license; mention it, but it's their call.)

## Commit identity

Every commit embeds author name and email, and they're public on any public repo forever.

- **Email:** use the forge's no-reply address. On GitHub it's `<id>+<username>@users.noreply.github.com` (Settings → Emails shows it). GitLab and Codeberg have equivalents. Enable "Keep my email addresses private" and "Block command line pushes that expose my email".
- **Name:** real name or handle. User's choice, but make it a conscious one.
- Set per repo with the user's OK: `git config user.email "…"` and `git config user.name "…"`. For "all personal projects use X", a conditional include in the global config does it (global change: ask):
  ```
  [includeIf "gitdir:~/projects/"]
      path = ~/.gitconfig-personal
  ```
- **Existing history** with the wrong identity: rewrite with `git filter-repo --mailmap mailmap.txt` (line format: `New Name <new@email> <old@email>`). It's a history rewrite: same rules as `secrets.md` (ask first, back up, force-push only on a yes). If history was already pushed publicly, the old email is already harvested; rewriting still stops future exposure.
- Also check branch names, tag messages and commit messages for names and clients (`git log --all --format=%B | grep -i <term>`). Rewriting messages needs `git filter-repo --message-callback`.
