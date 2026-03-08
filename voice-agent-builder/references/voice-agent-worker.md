# Voice Agent Worker (`apps/voice-agent`)

A Cloudflare Worker that handles outbound phone calls via Telnyx, bridges audio
to Deepgram Agent API for real-time AI conversation, and manages call state via
Durable Objects.

## Package Structure

```
apps/voice-agent/
├── package.json
├── wrangler.jsonc
├── src/
│   ├── index.ts           # Hono HTTP routes + entrypoint
│   ├── call-session.ts    # CallSession Durable Object
│   └── browser-session.ts # (optional) Browser testing DO
└── worker-configuration.d.ts
```

## package.json

```json
{
  "name": "voice-agent",
  "type": "module",
  "scripts": {
    "dev": "lsof -ti:8790 | xargs kill -9 2>/dev/null || true && wrangler dev",
    "build": "tsc -b",
    "deploy": "pnpm run build && wrangler deploy --minify",
    "cf-typegen": "wrangler types --env-interface CloudflareBindings",
    "tunnel": "ngrok http 8790",
    "secrets:bulk": "wrangler secret bulk .dev.vars.production"
  },
  "dependencies": {
    "@workspace/agent": "workspace:*",
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

Key dependency: `agents` (Cloudflare Agents SDK) — provides `Agent` base class,
`getAgentByName`, and `routeAgentRequest` for Durable Object routing.

## wrangler.jsonc

```jsonc
{
  "name": "your-voice-agent",
  "main": "src/index.ts",
  "workers_dev": true,
  "compatibility_date": "2026-01-07",
  "compatibility_flags": ["nodejs_compat"],
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

## HTTP Routes (`index.ts`)

### Route Map

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/health` | None | Health check |
| POST | `/calls/initiate` | Internal key | Start outbound call |
| POST | `/call-connected` | Telnyx signature | Webhook: call answered |
| POST | `/amd-result` | Telnyx signature | Webhook: machine detection result |
| POST | `/voicemail` | Telnyx signature | Webhook: play voicemail |
| ALL | `*` | None | Agents SDK catch-all (WebSocket routing) |

### Call Initiation Flow

```
POST /calls/initiate
  ├─ Validate input (initiateCallSchema)
  ├─ Generate callId (crypto.randomUUID)
  ├─ Store context in CallSession DO via PUT /init
  ├─ Create outbound call via Telnyx TeXML API
  │   ├─ url: /call-connected?callId={callId}
  │   ├─ machineDetection: "DetectMessageEnd"
  │   ├─ asyncAmd: true
  │   └─ amdStatusCallbackUrl: /amd-result?callId={callId}
  └─ Return { callId, providerCallId, status }
```

### Call Connected Webhook

When Telnyx reports the call is answered:

```
POST /call-connected?callId={callId}
  ├─ Verify Telnyx Ed25519 signature
  ├─ Extract CallSid from form body
  ├─ Build WebSocket URL: wss://{host}/agents/call-session/{callId}
  ├─ Return TeXML: <Connect><Stream url="wss://..." /></Connect>
  └─ Telnyx opens WebSocket to CallSession DO
```

### AMD Result Webhook

Answering machine detection result:

```
POST /amd-result?callId={callId}
  ├─ Verify Telnyx signature
  ├─ Extract AnsweredBy from form body
  ├─ If machine: redirect call to /voicemail?callId={callId}
  └─ If human: continue (WebSocket already connected)
```

### Voicemail Webhook

```
POST /voicemail?callId={callId}
  ├─ Verify Telnyx signature
  ├─ Fetch voicemail message from CallSession DO /agent-config
  ├─ Return TeXML: <Pause/><Say>{message}</Say><Hangup/>
```

### Agents SDK Catch-All

The final route delegates to Cloudflare's Agents SDK for WebSocket routing:

```typescript
app.all("*", async (c) => {
  const agentResponse = await routeAgentRequest(c.req.raw, c.env);
  if (agentResponse) return agentResponse;
  return c.text("Not found", 404);
});
```

This routes `wss://.../agents/call-session/{callId}` to the correct CallSession
Durable Object instance.

### Entrypoint Exports

Wrangler requires Durable Object classes to be exported from the entrypoint:

```typescript
export default app;
export { CallSession } from "./call-session.js";
export { BrowserSession } from "./browser-session.js";
```

## CallSession Durable Object (`call-session.ts`)

The core of the voice agent. Each call gets its own DO instance that:
1. Stores entity context and agent config
2. Handles the Telnyx media stream WebSocket
3. Bridges audio to/from Deepgram Agent API
4. Handles barge-in (caller interrupting the agent)

### State

```typescript
interface CallState {
  entityContext: EntityContext | undefined;
  agentConfig: AgentConfigData | undefined;
}
```

### HTTP Handler

The DO exposes two HTTP endpoints:

```
PUT /init     — Store entity context + agent config (called before call connects)
GET /agent-config — Retrieve agent config (used by voicemail handler)
```

### WebSocket Lifecycle

```
1. onConnect     — Telnyx WebSocket opens (called after TeXML <Stream> connects)
2. onMessage     — Telnyx sends media stream events
   ├─ "connected" — Log stream connected
   ├─ "start"     — Extract encoding/sampleRate, connect to Deepgram
   ├─ "media"     — Forward audio to Deepgram bridge
   ├─ "stop"      — Cleanup
   ├─ "mark"      — Ignore (used for mark events)
   └─ "dtmf"      — Ignore (or handle if needed)
3. onError       — Log error (expected on hangup)
4. onClose       — Cleanup bridge + connection
```

### Deepgram Connection

On the `start` event, connect to Deepgram with the entity context:

```typescript
private connectToDeepgram(encoding: string, sampleRate: number) {
  const systemPrompt = buildSystemPrompt(
    this.state.entityContext,
    this.state.agentConfig
  );

  const greeting = buildGreeting(
    this.state.agentConfig,
    this.state.entityContext?.entity.contactFirstName
  );

  const config = buildDeepgramConfig({
    audioEncoding: encoding as "linear16" | "mulaw",
    sampleRate,
    systemPrompt,
    greeting,
    ttsModel: this.state.agentConfig?.ttsModel ?? AGENT_CONFIG_DEFAULTS.ttsModel,
    sttModel: this.state.agentConfig?.sttModel ?? AGENT_CONFIG_DEFAULTS.sttModel,
    llmModel: this.state.agentConfig?.llmModel ?? AGENT_CONFIG_DEFAULTS.llmModel,
    llmTemperature: parseFloat(
      this.state.agentConfig?.llmTemperature ?? AGENT_CONFIG_DEFAULTS.llmTemperature
    ),
    env: {
      CF_ACCOUNT_ID: this.env.CF_ACCOUNT_ID,
      CF_GATEWAY_ID: this.env.CF_GATEWAY_ID,
      CF_AIG_TOKEN: this.env.CF_AIG_TOKEN,
    },
  });

  this.bridge = new DeepgramAgentBridge({
    audioEncoding,
    sampleRate,
    deepgramApiKey: this.env.DEEPGRAM_API_KEY,
    config,
    callbacks: {
      onReady: () => { this.bridgeReady = true; this.flushPreBridgeBuffer(); },
      onAudio: (audio) => { this.forwardAudioToTelnyx(audio); },
      onSpeakingChange: (event) => {
        if (event === "user_started") this.handleBargeIn();
      },
      onTranscript: (role, content) => { logger.info("Conversation", { role, content }); },
      onError: (error) => { logger.error("Deepgram error", { message: error.message }); },
      onClose: () => { this.bridge = undefined; this.bridgeReady = false; },
    },
  });
}
```

### Audio Bridging

```
Telnyx → base64 payload → decode → pre-bridge buffer (if not ready) → Deepgram
Deepgram → Buffer → base64 encode → JSON { event: "media", media: { payload } } → Telnyx
```

Pre-bridge buffer: Audio chunks that arrive before Deepgram is ready are buffered
and flushed once `onReady` fires.

### Barge-In

When the user starts speaking while the agent is talking:

```typescript
private handleBargeIn() {
  this.telnyxConnection.send(JSON.stringify({
    event: "clear",
    stream_id: this.streamId,
  }));
}
```

This tells Telnyx to clear its audio playback buffer, stopping the agent mid-sentence
so the caller can speak.

### Cleanup

On call end (stop event or WebSocket close):

```typescript
private cleanup() {
  this.bridge?.disconnect();  // Close Deepgram WebSocket + clear keepalive
  this.bridge = undefined;
  this.bridgeReady = false;
  this.preBridgeBuffer = [];
  this.telnyxConnection?.close();
  this.telnyxConnection = undefined;
  this.streamId = undefined;
}
```

### Critical: `hibernate: false`

```typescript
static options = { hibernate: false };
```

Voice calls need persistent WebSocket connections. Hibernation would close the
WebSocket and lose the audio stream. SMS sessions CAN use `hibernate: true`
because they're request-based.

### Critical: `shouldSendProtocolMessages`

```typescript
shouldSendProtocolMessages(_connection: Connection, _ctx: ConnectionContext) {
  return false;
}
```

Telnyx doesn't understand Agents SDK protocol messages. Sending them would corrupt
the media stream. This must be disabled.

## BrowserSession Durable Object (Optional — Browser Testing)

For testing the voice agent without making real phone calls. The BrowserSession
connects directly to Deepgram (bypassing Telnyx) and bridges audio from the
browser's microphone.

### How It Works

```
Browser (mic capture via AudioWorklet)
  ↕ WebSocket (binary audio + JSON commands)
BrowserSession DO
  ↕ Deepgram Agent API
  ↕ Claude (via AI Gateway)
```

### Authentication

Uses HMAC tokens (not JWT) — see `references/middleware.md` for details.

```
1. Frontend: POST /voiceTest/createToken → backend signs 60s HMAC token
2. Frontend: wss://voice-agent/agents/browser-session/{id}?token={token}
3. BrowserSession: verifyToken(env.VOICE_AGENT_INTERNAL_KEY, token)
4. If invalid → close(4001, "Unauthorized")
```

### WebSocket Protocol

**Client → Server (text):**
```json
{ "type": "start", "agentConfig": {...}, "customerContext": {...} }
```

**Client → Server (binary):** Raw linear16 PCM audio at 16kHz

**Server → Client (text):**
```typescript
type ServerMessage =
  | { type: "ready" }
  | { type: "transcript"; role: "user" | "agent"; content: string }
  | { type: "agent_speaking"; totalLatency?: number }
  | { type: "user_speaking" }
  | { type: "error"; content: string }
  | { type: "closed" };
```

**Server → Client (binary):** Raw linear16 PCM audio at 16kHz

### Key Differences from CallSession

| | CallSession | BrowserSession |
|---|---|---|
| Audio source | Telnyx media stream (base64 JSON) | Browser mic (raw binary) |
| Audio format | Encoding from Telnyx `start` event | Always linear16 @ 16kHz |
| Auth | None (Telnyx controls access) | HMAC token in URL |
| Context source | Pre-stored via PUT /init | Sent in `start` message |
| Mock data | Never | `buildMockCustomerContext()` if no context sent |
| Barge-in | Sends `clear` to Telnyx | No clear needed (browser handles it) |

### Mock Customer Context

When no `customerContext` is provided in the `start` message, the BrowserSession
uses `buildMockCustomerContext()` which returns realistic test data. This lets
admins test the agent without selecting a real entity.

### Validation Schemas

The `start` message is validated by `startMessageSchema`:

```typescript
const startMessageSchema = z.object({
  type: z.literal("start"),
  agentConfig: agentConfigDataSchema,
  customerContext: customerContextSchema.optional(),
});
```

### Critical Notes

- `shouldSendProtocolMessages` must return `false` — browser doesn't understand
  Agents SDK protocol messages
- `hibernate: false` — needs persistent WebSocket
- Binary audio is sent directly (no base64 wrapping like Telnyx)
- Audio worklet required on browser side (`capture-processor.js`) for mic capture

## Gotchas

- **WebSocket URL routing** — The Agents SDK routes `wss://{host}/agents/{agent-name}/{id}`
  to the correct DO. The agent name must match the class name in kebab-case
  (e.g., `CallSession` → `call-session`).
- **Base64 audio** — Telnyx sends audio as base64-encoded strings in JSON. Decode with
  `Uint8Array.from(atob(payload), (c) => c.charCodeAt(0))`. Encode responses with
  `audio.toString("base64")`.
- **Stream ID** — Telnyx provides `stream_id` in the `start` event. Use this for all
  outbound messages (`media`, `clear`). Without it, Telnyx ignores the messages.
- **PUBLIC_URL env var** — In development, use ngrok (`ngrok http 8790`) and set
  `PUBLIC_URL` to the ngrok URL. Telnyx webhooks need a publicly accessible URL.
- **Internal key auth** — The `/calls/initiate` endpoint should be authenticated with
  an internal key (shared secret between backend and voice agent worker). Don't expose
  it publicly.
- **Error on hangup** — The `onError` handler will fire when the caller hangs up.
  This is expected behavior — the WebSocket closes abruptly. Log it but don't treat
  it as a bug.
- **Telnyx form body parsing** — Webhook payloads use `application/x-www-form-urlencoded`.
  Use `c.req.parseBody()` in Hono to parse them.
