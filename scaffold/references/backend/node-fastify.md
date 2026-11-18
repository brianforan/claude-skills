# Backend: Node + TypeScript + Fastify

**Why Fastify over Express:** built-in schema validation and serialization, first-class TypeScript, faster, and it ships with `pino` structured logging (which outputs ECS for Elastic with one formatter). Express 5 is still the most widely deployed. If a project must use Express, keep everything below and swap in `pino-http` for request logging and `zod` middleware for validation.

## Tooling

| Concern | Choice |
|---|---|
| Node | current LTS (check), pinned in `.nvmrc` and `package.json` `engines` |
| Package manager | npm (`package-lock.json` committed) |
| Language | TypeScript, `strict: true`, ESM (`"type": "module"`) |
| Dev runner | `tsx watch` |
| Build | `tsc` to `dist/` |
| Validation + types | `zod` + `fastify-type-provider-zod` (schemas define both runtime checks and TS types) |
| Config | zod-validated `config.ts` reading `process.env` (dev: `node --env-file=../.env`) |
| DB | Drizzle ORM + `pg` (node-postgres) |
| Migrations | `drizzle-kit generate` → review SQL → `drizzle-kit migrate` |
| Logging | Fastify's built-in pino + `@elastic/ecs-pino-format` |
| Lint + format | ESLint (flat config, `typescript-eslint`) + Prettier |
| Tests | Vitest, using Fastify's `app.inject()` for HTTP tests (no real port) |

## Layout

```
api/
├── package.json
├── tsconfig.json
├── drizzle.config.ts
├── drizzle/                  # generated SQL migrations (committed)
├── src/
│   ├── server.ts             # entry: buildApp(), listen on API_PORT
│   ├── app.ts                # buildApp(): plugins, routes, error handler (no listen; tests use this)
│   ├── config.ts             # zod schema → typed, frozen config
│   ├── logger.ts             # pino options: ECS format, redaction, file destination
│   ├── db/
│   │   ├── client.ts         # pg Pool + drizzle()
│   │   └── schema.ts         # drizzle table definitions
│   ├── plugins/              # fastify plugins (db decorator, request id, error handler)
│   ├── services/             # business logic; no fastify imports
│   └── routes/               # thin route modules, one per resource
│       └── health.ts
└── test/
    └── health.test.ts
```

## Storage boundary

Services depend on a small repository interface (`LinkRepo`, `UserRepo`, …), not on Drizzle directly. Production wires in a Drizzle implementation; tests use an in-memory implementation. The whole HTTP test suite then runs with `app.inject()` and no database. DB-specific behaviour (constraints, atomic updates) gets a separate integration suite against the `_test` database.

## Logging setup

```ts
import { ecsFormat } from '@elastic/ecs-pino-format';

export const loggerOptions = {
  ...ecsFormat({ convertReqRes: true }),
  level: config.LOG_LEVEL,
  redact: { paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token', '*.secret'], censor: '[redacted]' },
  base: { service: { name: config.SERVICE_NAME, environment: config.NODE_ENV } },
};
```
- Build the pino instance yourself (`pino(loggerOptions, pino.multistream(streams))`) and pass it as **`loggerInstance`**. In Fastify 5 the `logger` option only takes options, not an instance. Type the dependency as `FastifyBaseLogger`, not pino's `Logger`, or the Fastify instance types stop lining up.
- Streams: stdout always, plus `pino.destination({ dest: LOG_FILE, mkdir: true })` when `LOG_FILE` is set. Give each stream entry `level: config.LOG_LEVEL`, because multistream entries default to `info`.
- Request IDs: `requestIdHeader: 'x-request-id'` + `genReqId: () => randomUUID()` uses the incoming header when present and generates one otherwise. Echo `request.id` back as `X-Request-ID` in an `onSend` hook.
- One access line per request: subclass Fastify's `LogController` (the top-level `disableRequestLogging` option is deprecated). Make `incomingRequest()` a no-op and log one ECS line in `requestCompleted()`:
  ```ts
  class EcsAccessLog extends LogController {
    override incomingRequest() {}
    override requestCompleted(_err: unknown, request: FastifyRequest, reply: FastifyReply) {
      const s = reply.statusCode;
      request.log[s >= 500 ? 'error' : s >= 400 ? 'warn' : 'info']({
        http: { request: { method: request.method }, response: { status_code: s } },
        url: { path: request.url.split('?')[0] },
        event: { duration: Math.round(reply.elapsedTime * 1e6) }, // ns
      }, 'request completed');
    }
  }
  // Fastify({ loggerInstance, logController: new EcsAccessLog({ requestIdLogLabel: 'request_id' }), ... })
  ```
  `requestIdLogLabel: 'request_id'` makes the ID field match the conventions (Fastify's default is `reqId`).
- Log with `request.log` inside handlers so the request ID is attached automatically. Never `console.log`.

## Errors

A `setErrorHandler` maps zod validation errors → 400, a custom `AppError(code, message, statusCode)` → its status, and anything else → 500 `internal_error` (logged with stack, nothing leaked). The response body follows the standard error JSON.

## package.json scripts

```
"dev": "tsx watch --env-file=../.env src/server.ts",
"build": "tsc -p tsconfig.json",
"start": "node --env-file=../.env dist/server.js",
"test": "vitest run",
"lint": "eslint . && prettier --check .",
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate"
```

## Gotchas

- ESM + TypeScript: use `"module": "NodeNext"` and include `.js` extensions in relative imports.
- `--env-file` fails if the file is missing. The README tells people to `cp .env.example .env` first.
- Keep `buildApp()` free of `listen()` so tests can `inject()` without ports.
- `fastify-type-provider-zod` needs `@fastify/swagger` as a peer dependency; install it explicitly.
- Bind `listen()` to `127.0.0.1` in dev; use `0.0.0.0` only inside a container.
- Windows: very deep project paths (over ~260 characters including `node_modules/...`) break tools that spawn bundled binaries. drizzle-kit's nested esbuild fails with `spawn … ENOENT`. Keep projects in a short path like `~/projects/<name>`.
