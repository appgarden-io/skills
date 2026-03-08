# SMS Agent Worker (`apps/sms-agent`)

A Cloudflare Worker that handles AI-powered SMS conversations via Telnyx.
Uses Durable Objects for conversation state and the Anthropic SDK directly
(no Deepgram — there's no audio in SMS).

## Package Structure

```
apps/sms-agent/
├── package.json
├── wrangler.jsonc
├── src/
│   ├── index.ts              # Hono HTTP routes + entrypoint
│   ├── sms-session.ts        # SmsSession Durable Object
│   ├── sms-handlers.ts       # Route handler implementations
│   └── sms-test-handler.ts   # (optional) Test message handler
└── worker-configuration.d.ts
```

## package.json

```json
{
  "name": "sms-agent",
  "type": "module",
  "scripts": {
    "dev": "lsof -ti:8791 | xargs kill -9 2>/dev/null || true && wrangler dev",
    "deploy": "pnpm run build && wrangler deploy --minify",
    "cf-typegen": "wrangler types --env-interface CloudflareBindings",
    "tunnel": "ngrok http 8791"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "latest",
    "@workspace/agent": "workspace:*",
    "@workspace/db": "workspace:*",
    "@workspace/logger": "workspace:*",
    "@workspace/middleware": "workspace:*",
    "@workspace/telnyx": "workspace:*",
    "agents": "0.5.0",
    "hono": "catalog:",
    "zod": "catalog:"
  }
}
```

Note: SMS agent uses `@anthropic-ai/sdk` directly because there's no audio
processing — it just needs text-in/text-out LLM calls.

## wrangler.jsonc

```jsonc
{
  "name": "your-sms-agent",
  "main": "src/index.ts",
  "workers_dev": true,
  "compatibility_flags": ["nodejs_compat"],
  "dev": { "port": 8791 },
  "durable_objects": {
    "bindings": [
      { "name": "SMS_SESSION", "class_name": "SmsSession" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["SmsSession"] }
  ]
}
```

## HTTP Routes

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/health` | None | Health check |
| POST | `/sms/initiate` | Internal key | Start outbound SMS conversation |
| POST | `/sms/inbound` | Telnyx signature | Webhook: customer replied |
| POST | `/sms/status` | Telnyx signature | Webhook: delivery status |
| POST | `/sms/do-takeover` | Internal key | Human takes over conversation |
| POST | `/sms/do-release` | Internal key | Release back to AI |
| POST | `/sms/human-send` | Internal key | Human sends message manually |

## SMS Initiation Flow

```
POST /sms/initiate
  ├─ Validate input (initiateSmsSchema)
  ├─ Fetch entity context + agent config from DB
  ├─ Create sms_conversation record in DB
  ├─ Initialize SmsSession DO (store context + config)
  ├─ Build greeting from template
  ├─ Send greeting via Telnyx SMS API
  ├─ Store outbound message in sms_message table
  └─ Return { conversationId, status: "initiated" }
```

## Inbound SMS Flow

```
POST /sms/inbound (Telnyx webhook)
  ├─ Verify Telnyx signature
  ├─ Extract from/to/text from webhook payload
  ├─ Find active conversation by phone numbers
  │   (match customerPhoneNumber + providerPhoneNumber, status IN active/human_taken)
  ├─ Store inbound message in sms_message table
  ├─ Update conversation (messageCount++, lastMessageAt)
  ├─ Forward to SmsSession DO /customer-message
  │   ├─ If human-controlled: return null (no AI response)
  │   └─ If AI-controlled: generate response via Claude
  ├─ If AI response: send via Telnyx + store in DB
  └─ Return 200
```

## SmsSession Durable Object

### State

```typescript
interface SessionState {
  conversationId: string;
  customerId: string;
  organizationId: string;
  customerContext: EntityContext;
  agentConfig: AgentConfigData;
  isHumanControlled: boolean;
  humanAgentId: string | null;
}
```

### HTTP Endpoints

```
PUT  /init              — Initialize session with context + config
POST /customer-message  — Process incoming customer message
POST /human-takeover    — Switch to human control
POST /human-release     — Switch back to AI control
```

### AI Response Generation

The SMS agent builds the Claude conversation from the message history in the database:

```typescript
private async generateAiResponse(
  state: SessionState,
  latestCustomerMessage: string
): Promise<string> {
  const db = getDb(this.env.DB);

  // Load message history from DB (last 30 messages)
  const messages = await db
    .select({ sender: smsMessage.sender, content: smsMessage.content })
    .from(smsMessage)
    .where(eq(smsMessage.conversationId, state.conversationId))
    .orderBy(asc(smsMessage.createdAt))
    .limit(30);

  // Build system prompt (same layered architecture as voice)
  const systemPrompt = buildSystemPrompt(
    state.customerContext,
    state.agentConfig,
    "sms"  // Uses SMS-specific base instructions
  );

  // Map message history to Anthropic format
  const anthropicMessages: Anthropic.MessageParam[] = [];
  for (const msg of messages) {
    if (msg.sender === "customer") {
      anthropicMessages.push({ role: "user", content: msg.content });
    } else {
      anthropicMessages.push({ role: "assistant", content: msg.content });
    }
  }

  // Call Claude via AI Gateway
  const client = new Anthropic({
    baseURL: `https://gateway.ai.cloudflare.com/v1/${this.env.CF_ACCOUNT_ID}/${this.env.CF_GATEWAY_ID}/anthropic`,
    defaultHeaders: {
      Authorization: `Bearer ${this.env.CF_AIG_TOKEN}`,
    },
  });

  const response = await client.messages.create({
    model: state.agentConfig.llmModel || "claude-haiku-4-5",
    max_tokens: 300,
    system: systemPrompt,
    messages: anthropicMessages,
  });

  const textBlock = response.content.find((block) => block.type === "text");
  return textBlock?.text ?? "I'm having trouble responding. A team member will follow up.";
}
```

### Human Takeover/Release

Human agents can take control of a conversation:

```typescript
// Takeover: set isHumanControlled = true, store humanAgentId
// When isHumanControlled, the DO returns null for AI response
// Human sends messages via /sms/human-send (bypasses DO entirely)

// Release: set isHumanControlled = false, clear humanAgentId
// AI resumes responding to customer messages
```

### Critical: `hibernate: true`

```typescript
static options = { hibernate: true };
```

SMS sessions are request-based (no persistent WebSocket), so hibernation is safe
and saves resources between messages.

## Status Callback Handler

Maps Telnyx delivery statuses to internal statuses:

```typescript
const statusMap = {
  queued: "pending",
  sending: "pending",
  sent: "sent",
  delivered: "delivered",
  delivery_unconfirmed: "delivered",
  failed: "failed",
  sending_failed: "failed",
  delivery_failed: "failed",
};
```

## SMS Test Handler (Optional)

For admin testing without creating real conversations. Receives agent config and
conversation history, returns a Claude response:

```typescript
const testMessageSchema = z.object({
  agentConfig: agentConfigDataSchema,
  customerContext: customerContextSchema.optional(),
  messages: z.array(z.object({
    role: z.enum(["user", "assistant"]),
    content: z.string(),
  })),
  latestMessage: z.string().nullable(),
});

export async function handleTestMessage(
  request: Request,
  env: CloudflareBindings
): Promise<Response> {
  const parsed = testMessageSchema.safeParse(await request.json());
  if (!parsed.success) return Response.json({ error: "Invalid input" }, { status: 400 });

  const { agentConfig, messages, latestMessage } = parsed.data;
  const customerContext = parsed.data.customerContext ?? buildMockEntityContext();

  // If no messages yet, return the greeting
  if (latestMessage === null && messages.length === 0) {
    const greeting = buildSmsGreeting(agentConfig, customerContext.entity.contactFirstName);
    return Response.json({ aiResponse: greeting });
  }

  // Build system prompt and call Claude
  const systemPrompt = buildSystemPrompt(customerContext, agentConfig, "sms");
  const anthropicMessages = [...messages];
  if (latestMessage) anthropicMessages.push({ role: "user", content: latestMessage });

  const client = new Anthropic({
    baseURL: `https://gateway.ai.cloudflare.com/v1/${env.CF_ACCOUNT_ID}/${env.CF_GATEWAY_ID}/anthropic`,
    defaultHeaders: { Authorization: `Bearer ${env.CF_AIG_TOKEN}` },
  });

  const response = await client.messages.create({
    model: agentConfig.llmModel || "claude-haiku-4-5",
    max_tokens: 300,
    system: systemPrompt,
    messages: anthropicMessages,
  });

  const textBlock = response.content.find((block) => block.type === "text");
  return Response.json({
    aiResponse: textBlock?.text ?? "Unable to respond. A team member will follow up.",
  });
}
```

Register in `index.ts`:
```typescript
app.post("/sms/test-message", internalKeyAuth("SMS_AGENT_INTERNAL_KEY"), (c) =>
  handleTestMessage(c.req.raw, c.env)
);
```

## Gotchas

- **Anthropic SDK `baseURL`** — For AI Gateway, set `baseURL` to
  `https://gateway.ai.cloudflare.com/v1/{ACCOUNT_ID}/{GATEWAY_ID}/anthropic`
  (no `/v1/messages` suffix — the SDK appends that). Use `Authorization` in
  `defaultHeaders` with the AI Gateway token.
- **Message history order** — Always `orderBy(asc(createdAt))` to maintain
  conversation chronology. Reversed order confuses the LLM.
- **`max_tokens: 300`** — SMS messages should be concise. 300 tokens is ~1-2
  short text messages. Adjust based on your domain's needs.
- **Both sender types map to assistant** — Both `"ai"` and `"human"` senders
  map to the `assistant` role in the Anthropic conversation. This maintains
  conversation continuity even after human takeover.
- **Conversation lookup by phone numbers** — When an inbound SMS arrives,
  find the conversation by matching both `customerPhoneNumber` AND
  `providerPhoneNumber`. Don't match on just one — a number could be in
  multiple conversations.
- **Telnyx inbound webhook payload** — Deeply nested structure:
  `body.data.payload.from.phone_number`. Validate the structure exists before
  accessing nested fields.
- **DB connection string** — The SMS agent needs a `DB` env var pointing to the
  Neon database connection string (with pooling). This is different from the
  voice agent which passes context via Durable Object state.
