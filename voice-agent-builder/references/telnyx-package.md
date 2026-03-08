# Telnyx Package (`packages/telnyx`)

Handles all Telnyx API interactions: outbound calls, SMS, webhook verification,
and TeXML generation. Zero npm dependencies — uses Web Crypto API for Ed25519
signature verification.

## Package Structure

```
packages/telnyx/
├── package.json
├── src/
│   ├── index.ts              # Barrel export
│   ├── client.ts             # API functions (calls, SMS)
│   ├── types.ts              # Schemas, interfaces, stream message types
│   ├── texml.ts              # TeXML XML builders
│   └── verify-signature.ts   # Ed25519 webhook verification
└── tsconfig.json
```

## package.json

```json
{
  "name": "@workspace/telnyx",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "dependencies": {
    "@workspace/logger": "workspace:*",
    "zod": "catalog:"
  }
}
```

Note: No `telnyx` npm package needed. We use the raw HTTP API directly because:
1. TeXML API uses form-encoded bodies (not JSON)
2. SMS API uses the v2 REST endpoint directly
3. Keeps the bundle small for Cloudflare Workers

## Types (`types.ts`)

### Input Validation Schemas

```typescript
import { z } from "zod/v4";

const E164_REGEX = /^\+[1-9]\d{1,14}$/;

export const initiateCallSchema = z.object({
  to: z.string().regex(E164_REGEX, "Invalid E.164 phone number format"),
  customerId: z.string().uuid(),
  agentConfig: z.unknown().optional(),
  customerContext: z.unknown().optional(),
});

export const initiateSmsSchema = z.object({
  to: z.string().regex(E164_REGEX, "Invalid E.164 phone number format"),
  customerId: z.string().uuid(),
  organizationId: z.string().min(1),
});
```

### API Interfaces

```typescript
export interface CreateOutboundCallOptions {
  apiKey: string;
  accountSid: string;
  from: string;
  to: string;
  applicationSid: string;
  url?: string;  // Per-call voice URL override
  machineDetection?: "Enable" | "DetectMessageEnd";
  asyncAmd?: boolean;
  amdStatusCallbackUrl?: string;
}

export interface RedirectCallOptions {
  apiKey: string;
  accountSid: string;
  callSid: string;
  texmlUrl: string;
}

export interface TelnyxCallResponse {
  sid: string;
  status: string;
  [key: string]: unknown;
}

export interface SendSmsOptions {
  apiKey: string;
  from: string;
  to: string;
  text: string;
  webhookUrl?: string;
}

export interface TelnyxSendResult {
  id: string;
  status: string;
}
```

### Telnyx Inbound SMS Webhook

```typescript
export interface TelnyxInboundSms {
  data: {
    event_type: "message.received";
    id: string;
    payload: {
      id: string;
      from: { phone_number: string };
      to: Array<{ phone_number: string }>;
      text: string;
      media: Array<{ url: string; content_type: string }>;
      received_at: string;
    };
  };
}
```

### Telnyx Media Stream WebSocket Messages

These are the message types sent over the WebSocket from Telnyx during a media
stream connection. The CallSession Durable Object must handle all of these:

```typescript
export interface TelnyxStreamConnected { event: "connected"; protocol: string; version: string; }
export interface TelnyxStreamStart {
  event: "start";
  sequence_number: string;
  start: {
    user_id?: string;
    call_control_id?: string;
    media_format: { encoding: string; sample_rate: number; channels: number; };
    custom_parameters?: Record<string, string>;
  };
  stream_id: string;
}
export interface TelnyxStreamMedia {
  event: "media";
  sequence_number: string;
  media: { track: string; chunk: string; timestamp: string; payload: string; };
  stream_id: string;
}
export interface TelnyxStreamStop { event: "stop"; sequence_number: string; stream_id: string; }
export interface TelnyxStreamMark { event: "mark"; sequence_number: string; stream_id: string; }
export interface TelnyxStreamDtmf { event: "dtmf"; stream_id: string; dtmf: { digit: string; }; }

export type TelnyxStreamMessage =
  | TelnyxStreamConnected | TelnyxStreamStart | TelnyxStreamMedia
  | TelnyxStreamStop | TelnyxStreamMark | TelnyxStreamDtmf;
```

## Client (`client.ts`)

### TeXML API Helper

All TeXML calls use form-encoded POST to
`https://api.telnyx.com/v2/texml/Accounts/{accountSid}/{path}`.

```typescript
interface TexmlFetchOptions {
  apiKey: string;
  accountSid: string;
  path: string;
  body: URLSearchParams;
  errorPrefix: string;
}

async function texmlFetch(options: TexmlFetchOptions): Promise<unknown> {
  const url = `https://api.telnyx.com/v2/texml/Accounts/${options.accountSid}${options.path}`;
  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${options.apiKey}`,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: options.body.toString(),
  });
  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(`${options.errorPrefix} (${response.status}): ${errorBody}`);
  }
  return response.json();
}
```

### Create Outbound Call

```typescript
export async function createOutboundCall(
  options: CreateOutboundCallOptions
): Promise<TelnyxCallResponse> {
  const body = new URLSearchParams({
    To: options.to,
    From: options.from,
    ApplicationSid: options.applicationSid,
  });

  if (options.url) body.append("Url", options.url);
  if (options.machineDetection) {
    body.append("MachineDetection", options.machineDetection);
    body.append("AsyncAmdStatusCallbackMethod", "POST");
  }
  if (options.asyncAmd) body.append("AsyncAmd", "true");
  if (options.amdStatusCallbackUrl) {
    body.append("AsyncAmdStatusCallback", options.amdStatusCallbackUrl);
  }

  const data = await texmlFetch({
    apiKey: options.apiKey,
    accountSid: options.accountSid,
    path: "/Calls",
    body,
    errorPrefix: "Telnyx TeXML API error",
  });

  assertSidAndStatus(data, "Telnyx TeXML API error");
  return data;
}
```

### Redirect Call (for voicemail routing)

```typescript
export async function redirectCall(options: RedirectCallOptions): Promise<void> {
  await texmlFetch({
    apiKey: options.apiKey,
    accountSid: options.accountSid,
    path: `/Calls/${options.callSid}`,
    body: new URLSearchParams({ Url: options.texmlUrl }),
    errorPrefix: "Telnyx TeXML redirect error",
  });
}
```

### Send SMS

Uses the native Telnyx Messaging API (JSON body, not TeXML):

```typescript
export async function sendSms(options: SendSmsOptions): Promise<TelnyxSendResult> {
  const payload: Record<string, string> = {
    from: options.from, to: options.to, text: options.text,
  };
  if (options.webhookUrl) payload.webhook_url = options.webhookUrl;

  const response = await fetch("https://api.telnyx.com/v2/messages", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${options.apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    const errorBody = await response.text();
    throw new Error(`Telnyx SMS send failed (${response.status}): ${errorBody}`);
  }

  // Parse response and extract id + status from nested structure
  const result = await response.json();
  // result.data.id, result.data.to[0].status
  return { id: result.data.id, status: result.data.to[0].status };
}
```

## TeXML Builders (`texml.ts`)

### Media Stream TeXML

Connects a call to a bidirectional WebSocket for real-time audio:

```typescript
function escapeXml(value: string): string {
  return value
    .replaceAll("&", "&amp;").replaceAll('"', "&quot;")
    .replaceAll("'", "&apos;").replaceAll("<", "&lt;").replaceAll(">", "&gt;");
}

export function buildMediaStreamTexml(options: {
  url: string;
  codec?: "PCMU" | "PCMA" | "G722" | "OPUS" | "AMR-WB" | "L16";
  samplingRate?: 8000 | 16000 | 24000;
}): string {
  const codec = options.codec ?? "L16";
  const samplingRate = options.samplingRate ?? 16_000;

  return [
    '<?xml version="1.0" encoding="UTF-8"?>',
    "<Response>",
    "<Connect>",
    `<Stream url="${escapeXml(options.url)}" bidirectionalMode="rtp" ` +
    `bidirectionalCodec="${codec}" bidirectionalSamplingRate="${samplingRate}" />`,
    "</Connect>",
    "</Response>",
  ].join("");
}
```

### Voicemail TeXML

Plays a message and hangs up:

```typescript
export function buildVoicemailTexml(options: {
  message: string;
  voice?: string;
}): string {
  const voice = options.voice ?? "Polly.Matthew";
  return [
    '<?xml version="1.0" encoding="UTF-8"?>',
    "<Response>",
    '<Pause length="1"/>',
    `<Say voice="${escapeXml(voice)}">${escapeXml(options.message)}</Say>`,
    "<Hangup/>",
    "</Response>",
  ].join("");
}
```

## Webhook Signature Verification (`verify-signature.ts`)

Verifies Ed25519 signatures from Telnyx webhooks using Web Crypto API. No npm
dependencies needed.

Telnyx sends:
- `telnyx-signature-ed25519` header: base64-encoded Ed25519 signature
- `telnyx-timestamp` header: Unix timestamp (seconds)
- Signed payload: `{timestamp}|{rawBody}`

```typescript
const TIMESTAMP_TOLERANCE_MS = 5 * 60 * 1000; // 5 minutes

export async function verifyTelnyxRequest(
  request: Request,
  publicKey: string
): Promise<Response | null> {
  const signature = request.headers.get("telnyx-signature-ed25519");
  const timestamp = request.headers.get("telnyx-timestamp");

  if (!(signature && timestamp)) {
    return Response.json({ error: "Missing Telnyx signature headers" }, { status: 403 });
  }

  // Check timestamp freshness (5-minute tolerance)
  const tsMs = Number(timestamp) * 1000;  // Telnyx uses seconds
  if (Math.abs(Date.now() - tsMs) > TIMESTAMP_TOLERANCE_MS) {
    return Response.json({ error: "Telnyx request timestamp expired" }, { status: 403 });
  }

  // Clone request to preserve body for downstream handlers
  const rawBody = await request.clone().text();
  const payload = `${timestamp}|${rawBody}`;

  try {
    const keyBytes = decodeKey(publicKey);  // Supports hex or base64
    const cryptoKey = await crypto.subtle.importKey(
      "raw", keyBytes.buffer, { name: "Ed25519" }, false, ["verify"]
    );
    const signatureBytes = decodeKey(signature);
    const valid = await crypto.subtle.verify(
      "Ed25519", cryptoKey, signatureBytes.buffer, new TextEncoder().encode(payload)
    );
    if (!valid) {
      return Response.json({ error: "Invalid Telnyx signature" }, { status: 403 });
    }
  } catch {
    return Response.json({ error: "Signature verification failed" }, { status: 403 });
  }

  return null;  // Valid — proceed
}
```

## Gotchas

- **TeXML API URL** — `https://api.telnyx.com/v2/texml/Accounts/{accountSid}/Calls`
  (not `/v2/texml/calls/{id}` or any other variation).
- **Form-encoded bodies** — TeXML API uses `application/x-www-form-urlencoded`, NOT JSON.
  The SMS API uses JSON. Don't mix them up.
- **E.164 encoding in form data** — The `+` in phone numbers gets URL-encoded as `%2B`
  by `URLSearchParams`. This is correct behavior — do not double-encode.
- **AMD requires two flags** — `MachineDetection: "DetectMessageEnd"` AND
  `AsyncAmd: "true"`. Without `AsyncAmd`, the webhook blocks until detection completes.
- **Ed25519 keys** — Telnyx public keys can be hex-encoded or base64-encoded. The
  verification code should handle both formats.
- **Signature payload** — `{timestamp}|{rawBody}` (pipe separator, timestamp in seconds).
- **Request cloning** — Always `request.clone()` before reading the body for signature
  verification, otherwise downstream handlers can't read it.
- **SIP 404 with $0.00 price** — Telnyx rejecting call internally. Check:
  1. Localisation Country set on TeXML app (validates number format)
  2. Max Destination Rate on outbound voice profile
  3. Whitelisted Destinations includes target country
- **Telnyx public key** — Found in the Telnyx portal under Account Settings > Public Key.
  This is different from the API key.
