# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Cloudflare Worker that runs [OpenClaw](https://github.com/openclaw/openclaw) (formerly Moltbot/Clawdbot) personal AI assistant inside a Cloudflare Sandbox container. The worker proxies HTTP/WebSocket requests, handles Cloudflare Access auth, and manages the container lifecycle.

## Commands

```bash
npm install              # Install dependencies
npm run dev              # Vite dev server (client only)
npm run start            # wrangler dev (local worker + container)
npm run build            # Build worker + React admin UI
npm run deploy           # Build and deploy to Cloudflare
npm run typecheck        # TypeScript check (no emit)
npm run lint             # oxlint src/
npm run lint:fix         # oxlint --fix src/
npm run format           # oxfmt --write src/
npm test                 # vitest run (single pass)
npm run test:watch       # vitest (watch mode)
npm run test:coverage    # vitest run --coverage
npx wrangler tail        # Live worker logs
npx wrangler secret list # Check configured secrets
```

Local dev setup:
```bash
cp .dev.vars.example .dev.vars
# Edit .dev.vars with: ANTHROPIC_API_KEY, DEV_MODE=true, DEBUG_ROUTES=true
npm run start
```

## Architecture

```
Browser
  │
  ▼
Cloudflare Worker (src/index.ts)
  - Hono app: auth middleware, route mounting, catch-all proxy
  - Starts OpenClaw in sandbox, proxies HTTP + WebSocket
  │
  ▼
Cloudflare Sandbox Container (Dockerfile)
  - OpenClaw gateway on port 18789
  - start-openclaw.sh: R2 restore → onboard → config patch → launch
```

**Request flow:** Public routes → (CF Access auth) → Admin/API/Debug routes → catch-all proxies to container on `MOLTBOT_PORT` (18789).

## Source Layout

```
src/
├── index.ts          # Main Hono app: middleware stack, route mounting, WS proxy
├── types.ts          # MoltbotEnv interface (all env vars), AppEnv, AccessUser
├── config.ts         # Constants (ports, timeouts, paths)
├── auth/             # Cloudflare Access JWT validation middleware
├── gateway/
│   ├── process.ts    # Find/start the OpenClaw process in sandbox
│   ├── env.ts        # buildEnvVars(): maps worker env → container env
│   ├── r2.ts         # R2 bucket mounting via s3fs
│   ├── sync.ts       # syncToR2() for backup
│   └── utils.ts      # waitForProcess() helper
├── routes/
│   ├── api.ts        # /api/admin/* (devices, gateway restart, storage sync)
│   ├── admin.ts      # /_admin/* static file serving
│   └── debug.ts      # /debug/* (processes, logs, version)
└── client/           # React admin UI (Vite build → dist/client/)
```

## Key Patterns

### Environment Variable Naming
Worker secrets use `MOLTBOT_*` names; the container receives different names. Mapping happens in `src/gateway/env.ts`:
- `MOLTBOT_GATEWAY_TOKEN` → `OPENCLAW_GATEWAY_TOKEN`
- `DEV_MODE` → `OPENCLAW_DEV_MODE`

### Adding a New API Endpoint
1. Add route handler in `src/routes/api.ts`
2. Add types if needed in `src/types.ts`
3. Update `src/client/api.ts` if the admin UI needs it
4. Add tests

### Adding a New Environment Variable
1. Add to `MoltbotEnv` interface in `src/types.ts`
2. If passed to container, add to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`
4. Document in `README.md` secrets table

### CLI Commands in Container
Always include `--url ws://localhost:18789` when calling the OpenClaw CLI from the worker. Commands take 10-15 seconds due to WebSocket overhead — use the `waitForProcess()` helper with `CLI_TIMEOUT_MS = 20000`.

Success detection uses case-insensitive check: `stdout.toLowerCase().includes('approved')`

### Docker Cache Busting
When changing `start-openclaw.sh`, bump the comment in Dockerfile:
```dockerfile
# Build cache bust: 2026-02-06-v28-openclaw-upgrade
```

## R2 Storage Gotchas

- Use `rsync -r --no-times` (not `rsync -a`) — s3fs doesn't support setting timestamps
- Check mount status via `mount | grep s3fs`, not from `sandbox.mountBucket()` error messages
- The mount directory `/data/moltbot` IS the R2 bucket — never `rm -rf` it
- R2 mounting only works in production, not with `wrangler dev`
- Backups stored under `openclaw/` prefix (migrated from legacy `clawdbot/`)

## Testing

Tests use Vitest, colocated with source files (`*.test.ts`). Coverage in `test/` directory.

Run a single test file:
```bash
npx vitest run src/auth/jwt.test.ts
```

## DEV_MODE vs E2E_TEST_MODE

- `DEV_MODE=true` — Skips CF Access auth AND bypasses OpenClaw device pairing
- `E2E_TEST_MODE=true` — Skips CF Access auth but keeps device pairing
- Neither mode requires `CF_ACCESS_TEAM_DOMAIN` or `CF_ACCESS_AUD`
