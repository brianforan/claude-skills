# Conventions (every project)

## Layout

One repo per project, with apps side by side. Include only the folders the project uses.

```
<project>/
├── api/                  # backend (see backend/*.md)
├── web/                  # frontend (see frontend/*.md)
├── compose.yaml          # local services: postgres, elastic stack (profile)
├── infra/
│   └── filebeat.yml      # only with Elastic logging
├── docs/
│   └── decisions/        # 0001-initial-stack.md, ...
├── scripts/              # dev helpers; no hard-coded hosts, paths or names
├── logs/                 # local log files shipped to Elastic (ignored)
├── .env.example          # every variable, placeholder values, one comment each
├── .editorconfig
├── .gitignore / .gitattributes   # via git-cleanup
└── README.md
```

A single-app project (API only, CLI) can put the app at the root instead of in `api/`. Use the same internal layout either way.

## Naming

- Folders and files: `kebab-case`, except where the language has its own convention (Python modules `snake_case`, React components `PascalCase.tsx`).
- Env vars: `SCREAMING_SNAKE_CASE`, prefixed by area: `DATABASE_URL`, `LOG_LEVEL`, `LOG_FILE`, `API_PORT`, `VITE_API_BASE_URL`.
- Database: see `data/postgres.md`.

## Configuration

- One root `.env` (ignored) and one `.env.example` (committed) for the whole repo. Apps read env vars, and dev scripts load the root `.env`.
- Each app has a single typed config module that validates on startup and fails fast with a clear message naming the missing variable. No `os.environ` or `process.env` reads scattered through the code.
- Defaults are safe for local dev; production values come only from env.

## Local ports

| Service | Port |
|---|---|
| API | 8000 |
| Web (Vite dev) | 5173 (proxies `/api` → 8000) |
| Postgres | 5432 |
| Elasticsearch | 9200 |
| Kibana | 5601 |

Infra services bind to `127.0.0.1` only, never `0.0.0.0`.

## API shape (any backend)

- Routes live under `/api/...`. `GET /health` (outside `/api`) returns `{"status":"ok"}` and checks the DB when there is one.
- JSON errors always look like `{"error": {"code": "not_found", "message": "...", "request_id": "..."}}`.
- Each request gets a request ID: use the incoming `X-Request-ID` header or generate one. Return it in the response header and include it in every log line for that request (see `observability/elastic.md`).
- Validate input at the edge (Pydantic / Zod). Handlers receive typed data.

## Logging (non-negotiable, every project)

- Structured JSON logs only, in ECS format (Elastic Common Schema) so Kibana understands them without mapping work.
- Write to stdout always. When `LOG_FILE` is set, also write to that file, which is how Filebeat picks them up locally.
- Levels: `debug` dev detail, `info` lifecycle and requests, `warning` recoverable oddities, `error` failed operations (with stack trace), `critical` the app can't continue. Default `LOG_LEVEL=info`.
- Never log secrets, tokens, passwords, full request bodies or personal data. Redact at the logger, not at the call site.
- Details per stack are in the backend files; the shared infra is in `observability/elastic.md`.

## Testing

- Every app ships with at least one real test that runs in CI-like conditions (no network, no shared state).
- Tests that need Postgres use a separate database (`<name>_test`), created by the test setup, never the dev database.

## Docs

- README sections in order: what it is (from the interview's purpose answer), getting started (prereqs, `cp .env.example .env`, start infra, run each app), configuration (env var table), project layout, logging (how to open Kibana and find the data view), decisions link, license.
- Decision records in `docs/decisions/NNNN-title.md`. Write a new one whenever a choice would surprise someone reading the code later. **Technical rationale only.** Personal motivations (learning goals, career, preferences about tools "for familiarity") never go in committed files.

## Git

- Default branch `main`, short-lived feature branches (`feat/...`, `fix/...`).
- Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`, with optional scope `feat(api):`.
- The publishing hygiene itself (ignore rules, secrets, identity, pre-commit guard) belongs to the `git-cleanup` skill.

## .editorconfig

```
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
trim_trailing_whitespace = true

[*.py]
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```
