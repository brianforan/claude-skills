# Scan checklist

How to audit a project with the tools already present: the Grep/Glob tools, `git` and `git grep`. Work through the sections in order and collect findings as `path:line, category, what` (redacted). Use judgment: a pattern match is a candidate, not a verdict.

## Never print secret values

The scan output lands in the conversation. For **secret** patterns, list locations only (`git grep -l` / `-c`, the Grep tool with `files_with_matches` or `count`), or cut output to `path:line` (`git grep -n … | cut -d: -f1,2`). To check whether a hit is a placeholder, look at just the start of the value (`… | sed -E 's/(=\s*.{4}).*/\1…/'`). Never echo a full key, token or password into the chat, a file or a commit.

Personal-info hits (names, paths) can be shown; the user needs to see them to decide.

## 1. What would be published

- Git repo: `git ls-files -co --exclude-standard` lists tracked files plus untracked-not-ignored ones, which is exactly what `git add -A` would commit. `git grep --untracked -I` searches that same set; add `--no-index` outside a repo.
- Not a repo yet: everything except dependency/build dirs (`node_modules`, `.venv`, `venv`, `dist`, `build`, `target`, `.next`, `bin`, `obj`, `__pycache__`). No ignore rules apply yet.
- Large files: anything over 50 MB (GitHub warns at 50 MB and rejects at 100 MB). `git ls-files -co --exclude-standard -z | xargs -0 du -k 2>/dev/null | sort -rn | head`.

## 2. Sensitive file names

Flag if present in the publishable set:

| Category | Names |
|---|---|
| Secrets (block) | `.env`, `.env.*` (except `.example/.sample/.template`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`, `*.ppk`, `*.kdbx`, `*.ovpn`, `id_rsa`, `id_ed25519`, `id_ecdsa`, `credentials*.json`, `*service-account*.json`, `client_secret*.json`, `.netrc`, `.htpasswd`, `.pypirc`, `kubeconfig*`, `*.tfstate*` |
| Secrets (review) | `*.tfvars`, `*.har`, `*.sqlite`, `*.db`, `*.dump`, `*.bak`, SQL files named like dumps or exports (not migrations) |
| AI local state (block) | `CLAUDE.local.md`, `.claude/settings.local.json`, `.aider*`, `.specstory/` |
| AI files (review) | `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`, `.claude/**`, `.cursor/**`, `.mcp.json` |
| Private notes (review) | `*notes*`, `journal*`, `diary*`, `todo*`, `scratch*`, `ideas*`, `devlog*`, `meeting*` (`.md/.txt/.org`), and decision folders (`adr/`, `decisions/`) |
| Deployment (review) | `fly.toml`, `vercel.json`, `netlify.toml`, `render.yaml`, `app.yaml`, `serverless.yml`, `Procfile`, `ecosystem.config.*`, `Caddyfile`, `nginx.conf`, `inventory*`, `hosts.ini`, `docker-compose*.yml`, `compose.yaml`, `*.tf`, `deploy*.sh/.ps1`, `.github/workflows/*deploy*`, `.vercel/`, `.netlify/`, `.firebaserc` |
| Junk | `.DS_Store`, `Thumbs.db`, `desktop.ini` |

## 3. Secret patterns (content)

Extended regex, usable with `git grep -E` or the Grep tool. Skip lockfiles and minified files (`*-lock.json`, `*.lock`, `go.sum`, `*.min.js`, `*.map`) except for the first block of patterns, which have too distinctive a shape to be false alarms.

High confidence (distinctive formats):
```
-----BEGIN ([A-Z]+ )?PRIVATE KEY
(AKIA|ASIA)[0-9A-Z]{16}
gh[pousr]_[A-Za-z0-9]{36,}
github_pat_[A-Za-z0-9_]{60,}
sk-ant-[A-Za-z0-9_-]{20,}
sk-(proj-|svcacct-)?[A-Za-z0-9_-]{32,}
[rs]k_live_[0-9A-Za-z]{20,}
xox[abprs]-[0-9A-Za-z-]{10,}
hooks\.slack\.com/services/[A-Za-z0-9/]+
discord(app)?\.com/api/webhooks/[0-9]+/
AIza[0-9A-Za-z_-]{35}
npm_[A-Za-z0-9]{36}
PrivateKey *= *[A-Za-z0-9+/]{42,43}=
```
Credentials inside URLs: `[a-z][a-z0-9+.-]*://[^ :@/'"]+:[^ @/'"]+@`. Ignore it when the password part is a placeholder (`${VAR}`, `$VAR`, `<…>`, `password`, `changeme`, `local-dev…`, `xxx`).

Lower confidence (read each hit): assignments like `(password|passwd|secret|token|api[_-]?key|access[_-]?key|client[_-]?secret)["']?\s*[:=]\s*["']?[A-Za-z0-9_\-+/=.]{8,}`. It's real only if the value looks random (mixed letters and digits, not an identifier like `getToken`, not `process.env.X` / `os.environ[...]`). JWTs: `eyJ[A-Za-z0-9_-]{10,}\.eyJ`.

Also check the remotes: `git remote -v` showing `https://user:token@…` is a leaked token in `.git/config`.

## 4. Personal info

**Identity terms.** Build the list once and store it in `.git/info/private-terms` (one per line; the pre-commit hook reads it):
- `git config user.name`, `git config user.email` (skip a no-reply address), and the email's local part
- OS username (`echo "$USERNAME$USER"`) and machine name (`hostname`)
- whatever the user adds: other handles, employer, clients, city, personal domains, server names

Search case-insensitively for whole words: `git grep -n -i -w -I -F -f .git/info/private-terms`.

**Patterns:**
- Home paths: `[A-Za-z]:[\\/]+Users[\\/]+[^\\/ "']+`, `/Users/[^/ "']+`, `/home/[^/ "']+`. Generic names like `Public`, `runner` or `user` are fine.
- Emails: `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}`. Ignore `example.com/.org`, `*.test`, `noreply`/no-reply addresses, `git@github.com`, `user:pass@host` (covered above) and filenames like `logo@2x.png`.
- Phone numbers: `\(?[0-9]{3}\)?[- ][0-9]{3}-[0-9]{4}`. Ignore `555-01xx`.
- Package manifests: `author`/`authors`/`maintainers` in `package.json`, `pyproject.toml`, `Cargo.toml`.
- Images and PDFs: photos may carry GPS EXIF. `git grep` can't see it; list `*.jpg/*.jpeg/*.heic/*.png` and suggest stripping with `exiftool -all=` (if installed) or re-exporting. PDFs: `grep -a -o '/Author *([^)]*)' file.pdf`.

## 5. Deployment details

- IPv4: `(^|[^0-9A-Za-z.])([0-9]{1,3}\.){3}[0-9]{1,3}([^0-9A-Za-z.]|$)`. Ignore version strings (`v1.2.3.4`, `Chrome/108.0.0.0`), SVG path data (lines with `d="`, `points=`, `viewBox`), `127.*`, `0.0.0.0`, documentation ranges (`192.0.2.*`, `198.51.100.*`, `203.0.113.*`), and public DNS (`1.1.1.1`, `8.8.8.8`). LAN ranges (`10.*`, `192.168.*`, `172.16–31.*`) are lower severity.
- SSH/rsync targets: `(ssh|scp|rsync|sftp) .*[A-Za-z0-9._-]+@[A-Za-z0-9.-]+\.[a-z]+`
- Internal hostnames: lowercase names ending in `.internal`, `.lan`, `.corp`, `.intranet`, `.home.arpa`, `.local`. Ignore filenames like `settings.local.json` and identifiers like `Tab.LOCAL`.
- Cloud IDs: `arn:aws[a-z-]*:[a-z0-9-]+:[a-z0-9-]*:[0-9]{12}:`, `s3://[a-z0-9.-]+`

## 6. History (when there are commits)

Fixing the working tree doesn't clean history. Check:
- Authors: `git log --all --format='%an <%ae>%n%cn <%ce>' | sort -u` (non-noreply emails are public on every commit)
- Files ever added: `git log --all --diff-filter=A --name-only --format= | sort -u`. Compare against section 2, especially sensitive files that were later deleted.
- Content: `git log --all -p -G '<pattern>' --format='%h' --name-only` per high-confidence secret pattern and for the private terms. Report commit and file, not the value.
- Commit messages: `git log --all --format='%h %s%n%b' | grep -i -w -F -f .git/info/private-terms`
- Whether it's already public: `git branch -r` non-empty means anything above may already be out. See `secrets.md`.

## 7. Repo hygiene

- `.gitignore`, `.gitattributes`, README and LICENSE (if public) exist.
- These would be ignored if created. `git check-ignore -q --no-index <path>` exits 0 when ignored. Probe `.env`, `.env.local`, `CLAUDE.local.md`, `.claude/settings.local.json`, `x.pem`, `terraform.tfstate`, plus `node_modules/x` / `.venv/x` for the stack.
- `.env` exists but there's no `.env.example`.
- `git config user.email` is set and is a no-reply address.
- The pre-commit hook is installed (`git config core.hooksPath` or `.git/hooks/pre-commit`), and an existing hook isn't overwritten.

## Accepting a finding

A value the user has reviewed and is fine publishing (a public business email, a documented public key) goes in `.git/info/git-cleanup-allow`, one literal per line. Or put `gitleaks:allow` in a comment on that line. The hook honours both.
