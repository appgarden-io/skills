# Middleware Package (`packages/middleware`)

Shared middleware for both voice and SMS agent workers. Handles authentication,
request logging, error handling, and HMAC token management.

## Package Structure

```
packages/middleware/
├── package.json
├── src/
│   ├── index.ts                # Barrel export
│   ├── internal-key-auth.ts    # Shared secret auth middleware
│   ├── hmac-token.ts           # HMAC token sign/verify (for browser testing)
│   ├── request-logger.ts       # Request lifecycle logging
│   └── global-error-handler.ts # Unhandled error catch-all
└── tsconfig.json
```

## Internal Key Auth

Protects endpoints that should only be called by the backend (not publicly).
Checks the `X-Internal-Key` header against an env var.

```typescript
import { createMiddleware } from "hono/factory";

export const internalKeyAuth = (envKeyName: string) =>
  createMiddleware(async (c, next) => {
    const key = c.req.header("X-Internal-Key");
    const expected = (c.env as Record<string, string>)[envKeyName];

    if (!key || key !== expected) {
      return c.json({ error: "Unauthorized" }, 401);
    }

    await next();
  });
```

**Usage in routes:**
```typescript
app.post("/calls/initiate", internalKeyAuth("VOICE_AGENT_INTERNAL_KEY"), handler);
app.post("/sms/initiate", internalKeyAuth("SMS_AGENT_INTERNAL_KEY"), handler);
```

The env var name is passed as a string so the same middleware works across workers
with different secret names.

## HMAC Token (for Browser Testing)

Used by the BrowserSession Durable Object to authenticate WebSocket connections.
This is NOT JWT — it's a simpler `base64url(payload).base64url(hmac_signature)` format.

### Sign a Token

```typescript
export async function signToken(
  secret: string,
  payload: Record<string, unknown>
): Promise<string> {
  const payloadBytes = new TextEncoder().encode(JSON.stringify(payload));
  const payloadB64 = base64urlEncode(payloadBytes);

  const key = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(secret).buffer,
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"]
  );

  const signature = await crypto.subtle.sign("HMAC", key, payloadBytes.buffer);
  const sigB64 = base64urlEncode(new Uint8Array(signature));

  return `${payloadB64}.${sigB64}`;
}
```

### Verify a Token

```typescript
export async function verifyToken(
  secret: string,
  token: string
): Promise<{ valid: boolean; payload?: Record<string, unknown> }> {
  const dotIndex = token.indexOf(".");
  if (dotIndex === -1) return { valid: false };

  const payloadB64 = token.slice(0, dotIndex);
  const sigB64 = token.slice(dotIndex + 1);

  // Decode both parts
  const payloadBytes = base64urlDecode(payloadB64);
  const sigBytes = base64urlDecode(sigB64);

  // Verify HMAC signature
  const key = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(secret).buffer,
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["verify"]
  );

  const valid = await crypto.subtle.verify("HMAC", key, sigBytes.buffer, payloadBytes.buffer);
  if (!valid) return { valid: false };

  // Parse and check expiry
  const payload = JSON.parse(new TextDecoder().decode(payloadBytes));
  if (typeof payload.exp === "number" && payload.exp < Date.now()) {
    return { valid: false };
  }

  return { valid: true, payload };
}
```

### Token Flow (Browser Voice Testing)

```
1. Frontend requests token via API: POST /voiceTest/createToken
2. Backend signs token with 60s TTL:
   signToken(VOICE_AGENT_INTERNAL_KEY, { exp: Date.now() + 60_000 })
3. Frontend connects WebSocket with token in URL:
   wss://voice-agent/agents/browser-session/{sessionId}?token={token}
4. BrowserSession DO verifies token on connect:
   verifyToken(this.env.VOICE_AGENT_INTERNAL_KEY, token)
5. If invalid/expired → close(4001, "Unauthorized")
```

## Request Logger

Adds a unique `requestId` to each request and logs lifecycle events:

```typescript
export const requestLogger = createMiddleware(async (c, next) => {
  const requestId = crypto.randomUUID();
  const start = Date.now();

  logger.info("Request started", {
    requestId,
    method: c.req.method,
    path: c.req.path,
  });

  await next();

  logger.info("Request completed", {
    requestId,
    method: c.req.method,
    path: c.req.path,
    status: String(c.res.status),
    durationMs: String(Date.now() - start),
  });
});
```

## Global Error Handler

Catches unhandled errors and returns structured 500 responses:

```typescript
export const globalErrorHandler = (err: Error, c: Context) => {
  logger.error("Unhandled error", {
    message: err.message,
    stack: err.stack,
  });

  return c.json(
    { error: "Internal server error" },
    500
  );
};
```

**Usage:**
```typescript
const app = new Hono();
app.onError(globalErrorHandler);
app.use("*", requestLogger);
```

## Gotchas

- **`X-Internal-Key` header** — The backend must send this header when calling agent
  workers. Without it, the request is rejected with 401.
- **Token is NOT JWT** — No header segment, no algorithm declaration. Just
  `base64url(payload).base64url(hmac)`. Do not try to decode it with JWT libraries.
- **Token expiry uses milliseconds** — `payload.exp` is `Date.now() + ttlMs`, not
  Unix seconds. Keep TTL short (60s) for browser testing tokens.
- **Web Crypto API only** — All crypto uses `crypto.subtle` (available in CF Workers).
  No Node.js `crypto` module. No npm dependencies needed.
- **base64url encoding** — Standard base64 with `+→-`, `/→_`, padding stripped.
  Must decode with the reverse transformation before `atob()`.
