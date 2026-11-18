# Secrets

## Where they hide (beyond the obvious `.env`)

- Config files: `config.json`, `settings.py`, `appsettings.json`, `application.yml`, `docker-compose.yml` `environment:` blocks
- Notebook outputs: `.ipynb` stores cell output, so a printed token stays in the file even after the code changes. Clear outputs (`jupyter nbconvert --clear-output --inplace`) or use `nbstripout`.
- Test fixtures and recorded HTTP cassettes (VCR, Polly, `.har` files), which hold real auth headers and cookies
- Package manager auth: `.npmrc` (`_authToken`), `.pypirc`, `.yarnrc.yml`, `.netrc`, `.docker/config.json`
- MCP and AI tool config: `.mcp.json`, `.claude/settings*.json` `env` blocks, `.cursor/mcp.json`
- Remote URLs: `https://user:token@github.com/...` in `.git/config`, CI files, scripts and READMEs
- Logs, crash dumps, SQLite/DB files, database dumps (`*.sql`), screenshots of dashboards
- Shell scripts that `export` keys, and Makefile variables

## Fix pattern

1. Move the value to an environment variable. The code reads `process.env.X` / `os.environ["X"]` and fails loudly with a clear message when it's missing.
2. Real values live in `.env` (ignored). Commit `.env.example` with every variable, a placeholder and a one-line comment:
   ```
   # Stripe secret key (dashboard → Developers → API keys). Use a test key locally.
   STRIPE_SECRET_KEY=sk_test_replace_me
   ```
3. For CI, use the forge's secret store (GitHub Actions `secrets.X` / environments). Workflows reference `${{ secrets.X }}` and never contain values.
4. Credential files (service-account JSON, `.pem`, keystores) go outside the repo or in an ignored path. The code takes a path from an env var.

## Already committed?

Work out which case applies. It changes what's urgent.

**Never pushed anywhere.** Nobody has it. Options, least to most effort:
- Brand-new or throwaway history: with the user's OK, start clean. Back up `.git` first, then remove it, `git init`, and make one clean initial commit. Simplest, and only appropriate when the user doesn't care about the old history.
- Keep history: rewrite with `git filter-repo` (`pip install git-filter-repo`):
  - remove a file everywhere: `git filter-repo --invert-paths --path path/to/secret.json`
  - replace a string everywhere: put `literal-secret==>REDACTED` lines in a file, then `git filter-repo --replace-text replacements.txt`
  - BFG Repo-Cleaner is an alternative for large repos.
  - filter-repo refuses to run on a non-fresh clone without `--force`. Run it on a fresh clone or back up first.

**Pushed (even to a private repo, even briefly).** Treat it as leaked:
1. **Rotate or revoke the credential first.** This is the actual fix. Tell the user exactly which keys and where to rotate them; you can't do it for them.
2. Then optionally rewrite history as above and force-push. That needs explicit approval, since it rewrites shared history and breaks other clones.
3. Forks, cached views, PR refs and scrapers may still hold it. GitHub support can purge cached views. This is why rotation comes first.

Never rewrite history or force-push without an explicit yes, and say plainly what will change.

## Ongoing protection

- Local pre-commit guard (`assets/pre-commit`): sh + git pattern checks, plus gitleaks if installed.
- gitleaks (`winget install gitleaks` / `brew install gitleaks`) and trufflehog are the standard scanners. The pre-commit framework has hooks for both.
- GitHub: secret scanning and push protection (free on public repos; Settings → Code security). It blocks pushes containing known token formats.
- Prefer short-lived or scoped credentials (OIDC from CI to cloud, fine-grained PATs, test-mode keys locally) so a leak does less damage.
