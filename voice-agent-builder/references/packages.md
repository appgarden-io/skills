# Required Packages & Dependencies

## Monorepo Package Catalog

These are the pinned versions used across the workspace (defined in `pnpm-workspace.yaml`
under `catalog:`). Using `catalog:` in `package.json` resolves to these versions:

| Package | Catalog Version | Used By |
|---------|----------------|---------|
| `hono` | `^4.11.3` | All workers, middleware |
| `zod` | `^4.3.5` | All packages (validation) |
| `wrangler` | `^4.57.0` | All workers (dev) |
| `typescript` | `^5.9.2` | All packages (dev) |
| `@cloudflare/workers-types` | `^4.20250823.0` | All workers (dev) |
| `vitest` | `^3.2.4` | Packages with tests (dev) |
| `drizzle-orm` | `^0.45.1` | Database package |
| `drizzle-kit` | `^0.31.9` | Database migrations (dev) |
| `@neondatabase/serverless` | `^1.0.0` | Database package |

## Workspace Packages

These are the internal workspace packages. Every `"workspace:*"` dependency in
the apps resolves to the local package.

### `@workspace/agent`

Shared AI agent logic — prompts, Deepgram bridge, config, schemas.

```json
{
  "name": "@workspace/agent",
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@deepgram/sdk": "^4.11.3",
    "@workspace/db": "workspace:*",
    "@workspace/logger": "workspace:*",
    "zod": "catalog:"
  }
}
```

Key dependency: `@deepgram/sdk` provides TypeScript types for the Agent API
(`AgentLiveSchema`, `AgentEvents`, etc.). The actual WebSocket connection is
built manually — the SDK is primarily used for type definitions.

### `@workspace/telnyx`

Telnyx API client — outbound calls, SMS, TeXML builders, webhook verification.

```json
{
  "name": "@workspace/telnyx",
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@workspace/logger": "workspace:*",
    "zod": "catalog:"
  }
}
```

No Telnyx SDK — all API calls are raw `fetch` with form-encoded or JSON bodies.
Webhook verification uses Web Crypto API (Ed25519) with zero npm dependencies.

### `@workspace/middleware`

Shared middleware — internal key auth, HMAC tokens, request logger, error handler.

```json
{
  "name": "@workspace/middleware",
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@workspace/logger": "workspace:*",
    "hono": "catalog:"
  }
}
```

### `@workspace/logger`

Structured JSON logger — zero dependencies.

```json
{
  "name": "@workspace/logger",
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "devDependencies": {
    "typescript": "catalog:"
  }
}
```

Implementation:

```typescript
type LogLevel = "debug" | "info" | "warn" | "error";

export interface LogContext {
  requestId?: string;
  userId?: string;
  procedure?: string;
  [key: string]: unknown;
}

const createLogEntry = (level: LogLevel, message: string, context: LogContext = {}) => ({
  level,
  message,
  timestamp: new Date().toISOString(),
  ...context,
});

export const logger = {
  debug: (message: string, context?: LogContext) =>
    console.log(createLogEntry("debug", message, context)),
  info: (message: string, context?: LogContext) =>
    console.log(createLogEntry("info", message, context)),
  warn: (message: string, context?: LogContext) =>
    console.warn(createLogEntry("warn", message, context)),
  error: (message: string, context?: LogContext) =>
    console.error(createLogEntry("error", message, context)),
};
```

Uses `console.log`/`console.warn`/`console.error` which integrates with
Cloudflare Workers' built-in log shipping when observability is enabled.

### `@workspace/db`

Database schema and utilities — Drizzle ORM + Neon Postgres. See
`references/database-schema.md` for table definitions.

## App Packages

### `voice-agent`

Cloudflare Worker for outbound voice calls.

```json
{
  "name": "voice-agent",
  "type": "module",
  "dependencies": {
    "@workspace/agent": "workspace:*",
    "@workspace/db": "workspace:*",
    "@workspace/logger": "workspace:*",
    "@workspace/middleware": "workspace:*",
    "@workspace/telnyx": "workspace:*",
    "agents": "0.5.0",
    "hono": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "catalog:",
    "typescript": "catalog:",
    "vitest": "catalog:",
    "wrangler": "catalog:"
  }
}
```

Key dependency: `agents` (Cloudflare Agents SDK `0.5.0`) provides the `Agent`
base class for Durable Objects, `getAgentByName`, and `routeAgentRequest` for
WebSocket routing. Pin to `0.5.0` — this is the version tested with the media
stream pattern.

### `sms-agent`

Cloudflare Worker for AI SMS conversations.

```json
{
  "name": "sms-agent",
  "type": "module",
  "dependencies": {
    "@anthropic-ai/sdk": "^0.71.2",
    "@workspace/agent": "workspace:*",
    "@workspace/db": "workspace:*",
    "@workspace/logger": "workspace:*",
    "@workspace/middleware": "workspace:*",
    "@workspace/telnyx": "workspace:*",
    "agents": "0.5.0",
    "hono": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "catalog:",
    "typescript": "catalog:",
    "wrangler": "catalog:"
  }
}
```

Key dependency: `@anthropic-ai/sdk` (`^0.71.2`) for direct Claude API calls.
Voice uses Claude through Deepgram's LLM provider; SMS calls Claude directly.

## Dev Tool Versions

| Tool | Version | Purpose |
|------|---------|---------|
| `pnpm` | ≥9 | Package manager (required for workspace:* protocol) |
| `wrangler` | `^4.57.0` | Cloudflare Workers CLI |
| `ngrok` | Latest | Tunnel for local Telnyx webhook development |
| `drizzle-kit` | `^0.31.9` | Database migration generation |

## Wrangler Configuration

Each worker needs a `wrangler.jsonc` that declares Durable Object bindings,
SQLite migrations, compatibility flags, and dev ports. The backend also needs
service bindings to talk to the agent workers. Getting these wrong causes
silent failures or deploy errors.

### Voice Agent `wrangler.jsonc`

```jsonc
{
  "name": "your-voice-agent",       // Must match service binding name in backend
  "main": "src/index.ts",
  "workers_dev": true,
  "compatibility_date": "2026-01-07",
  "compatibility_flags": ["nodejs_compat"],  // Required for Deepgram SDK types
  "dev": { "port": 8790 },
  "durable_objects": {
    "bindings": [
      { "name": "CALL_SESSION", "class_name": "CallSession" },
      { "name": "BROWSER_SESSION", "class_name": "BrowserSession" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["CallSession"] },
    { "tag": "v2", "new_sqlite_classes": ["BrowserSession"] }
  ],
  "observability": { "enabled": true, "head_sampling_rate": 1 }
}
```

Key points:
- **`nodejs_compat`** — Required. Without it, `Buffer` and other Node.js APIs
  used by the Deepgram SDK types won't be available.
- **`new_sqlite_classes`** — Each DO class that extends `Agent` needs its own
  migration tag. The Agents SDK uses SQLite internally for state persistence.
  Add new classes in a new migration tag (don't modify existing ones).
- **Binding names** — `CALL_SESSION` and `BROWSER_SESSION` are referenced in
  code via `env.CALL_SESSION`. The `class_name` must exactly match the exported
  class name from `index.ts`.

### SMS Agent `wrangler.jsonc`

```jsonc
{
  "name": "your-sms-agent",         // Must match service binding name in backend
  "main": "src/index.ts",
  "compatibility_date": "2026-01-07",
  "compatibility_flags": ["nodejs_compat"],
  "dev": { "port": 8792 },          // Different port from voice agent!
  "durable_objects": {
    "bindings": [
      { "name": "SMS_SESSION", "class_name": "SmsSession" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["SmsSession"] }
  ],
  "observability": { "enabled": true, "head_sampling_rate": 1 }
}
```

### Backend `wrangler.jsonc`

The backend connects to agent workers via **service bindings** — not HTTP URLs.
This is critical: in production, service bindings call workers directly over
Cloudflare's internal network (zero latency overhead, no public URL needed).

```jsonc
{
  "name": "your-backend",
  "main": "src/index.ts",
  "compatibility_date": "2026-01-07",
  "compatibility_flags": ["nodejs_compat"],
  "dev": { "port": 8787 },
  "services": [
    { "binding": "VOICE_AGENT", "service": "your-voice-agent" },
    { "binding": "SMS_AGENT", "service": "your-sms-agent" }
  ],
  "observability": { "enabled": true, "head_sampling_rate": 1 }
}
```

Key points:
- **`"service"` value must match the `"name"` in the agent's wrangler.jsonc.**
  If voice agent's name is `"acme-voice-agent"`, then the backend must use
  `{ "binding": "VOICE_AGENT", "service": "acme-voice-agent" }`.
- In code, `env.VOICE_AGENT.fetch(...)` calls the worker. The URL hostname is
  arbitrary — only the path matters (e.g., `https://voice-agent/calls/initiate`).
- In local dev, Wrangler automatically creates local bindings between workers
  running on their respective ports.

### Agents SDK WebSocket Routing

The Agents SDK routes WebSocket connections based on URL path convention:

```
wss://{host}/agents/{agent-name}/{instance-id}
```

The `agent-name` is the **kebab-case** of the class name:
- `CallSession` → `call-session`
- `BrowserSession` → `browser-session`
- `SmsSession` → `sms-session`

This is handled by the `routeAgentRequest()` catch-all in `index.ts`. The
`instance-id` becomes the Durable Object ID (each call/session gets its own DO).

## Environment Files

### `.dev.vars` (per worker, local development only)

Each worker (`apps/voice-agent/`, `apps/sms-agent/`) has its own `.dev.vars`
file that Wrangler reads automatically in dev mode. Format is `KEY=value` (no
quotes, no export).

**Voice agent `.dev.vars`:**
```
TELNYX_API_KEY=KEY_xxx
TELNYX_PUBLIC_KEY=abc123...
TELNYX_ACCOUNT_SID=a1b63975-...
TELNYX_TEXML_APP_SID=290165...
TELNYX_PHONE_NUMBER=+61240727243
DEEPGRAM_API_KEY=dg-xxx
CF_ACCOUNT_ID=9a7d04f1a...
CF_GATEWAY_ID=my-gateway
CF_AIG_TOKEN=aig-xxx
VOICE_AGENT_INTERNAL_KEY=random-uuid-here
PUBLIC_URL=https://abc123.ngrok-free.app
```

**SMS agent `.dev.vars`** (all of the above, plus):
```
DB=postgresql://user:pass@host/dbname?sslmode=require
SMS_AGENT_INTERNAL_KEY=another-random-uuid
PUBLIC_URL=https://def456.ngrok-free.app
```

### `.dev.vars.production` (per worker, production secrets)

Same format as `.dev.vars` but with production values. Used with
`wrangler secret bulk .dev.vars.production` to push all secrets at once.
Never commit this file.

## Gotchas

- **`agents` version pinning** — Pin `agents` to `0.5.0`, not `catalog:`. The
  Agents SDK is evolving quickly and breaking changes happen between minor
  versions. The media stream + Durable Object pattern is tested against 0.5.0.
- **Catalog resolution** — `"hono": "catalog:"` in package.json resolves to the
  version in `pnpm-workspace.yaml`. If you see "unresolved catalog" errors, ensure
  `pnpm-workspace.yaml` has a `catalog:` section with the dependency.
- **No Telnyx SDK** — We don't use the `telnyx` npm package. All Telnyx API calls
  are raw `fetch`. The TeXML API uses form-encoded bodies; the messaging API uses
  JSON. Mixing these up causes silent failures.
- **No Deepgram SDK at runtime** — `@deepgram/sdk` is used for TypeScript types
  only. The actual WebSocket connection to Deepgram is built manually with the
  native `WebSocket` API (available in Cloudflare Workers).
- **`@anthropic-ai/sdk`** — Only the SMS agent needs this (direct Claude API
  calls). The voice agent routes Claude through Deepgram's LLM provider, so it
  never imports the Anthropic SDK.
- **Service bindings vs fetch** — In production, backend-to-worker communication
  uses `env.VOICE_AGENT.fetch(...)` (service binding), not `fetch(url)`. The URL
  hostname in service binding calls is ignored — only the path matters.
- **Service name mismatch** — The `"service"` value in the backend's wrangler.jsonc
  must exactly match the `"name"` in the agent worker's wrangler.jsonc. A mismatch
  causes "service not found" errors in production and silent failures in local dev.
- **DO migration tags are append-only** — Never modify an existing migration tag.
  To add a new DO class, create a new tag. Modifying existing tags causes "migration
  mismatch" deploy errors that require manual resolution.
- **`nodejs_compat` flag** — Without this, imports from Node.js built-ins (`Buffer`,
  `crypto`, etc.) fail at runtime. The Agents SDK and Deepgram SDK types both depend
  on it.
- **Port conflicts** — Each worker must use a unique port in `dev.port`. Voice on
  8790, SMS on 8792, backend on 8787. Wrangler will silently fail or bind to a
  random port if there's a conflict.
- **`.dev.vars` is NOT `.env`** — Wrangler reads `.dev.vars` (no dot-env format
  differences, but no `export` prefix, no quoted values). Don't confuse with
  dotenv files.
- **WebSocket routing class name** — The Agents SDK derives the URL path from the
  class name in kebab-case (`CallSession` → `/agents/call-session/{id}`). If you
  rename a DO class, the WebSocket URL changes and Telnyx/browser connections break.
