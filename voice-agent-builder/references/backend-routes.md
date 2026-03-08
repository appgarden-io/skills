# Backend API Routes

The backend (existing API layer) acts as a proxy between the frontend and the
voice/SMS agent workers. It fetches entity context and agent config from the
database, then forwards to the appropriate worker.

## Worker Proxy Pattern

Both voice and SMS use the same proxy helper pattern. These use **Cloudflare
Service Bindings** (`env.VOICE_AGENT`, `env.SMS_AGENT`) — not HTTP URLs. The
hostname in the URL is arbitrary (only the path matters).

```typescript
function proxyToVoiceAgent(
  env: CloudflareBindings,
  path: string,
  body: Record<string, unknown>
): Promise<Response> {
  return env.VOICE_AGENT.fetch(
    new Request(`https://voice-agent${path}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Internal-Key": env.VOICE_AGENT_INTERNAL_KEY,
      },
      body: JSON.stringify(body),
    })
  );
}

function proxyToSmsAgent(
  env: CloudflareBindings,
  path: string,
  body: Record<string, unknown>
): Promise<Response> {
  return env.SMS_AGENT.fetch(
    new Request(`https://sms-agent${path}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Internal-Key": env.SMS_AGENT_INTERNAL_KEY,
      },
      body: JSON.stringify(body),
    })
  );
}
```

## Call Initiation Route

```typescript
const initiateCall = protectedProcedure
  .input(z.object({
    customerId: z.string().uuid(),
    to: z.string().regex(/^\+[1-9]\d{1,14}$/, "Invalid E.164 phone number"),
  }))
  .handler(async ({ input, context }) => {
    const organizationId = getOrganizationId(context);

    // Fetch agent config + entity context in a SINGLE withRls call
    const { agentConfigData, entityContext } = await context.withRls(async (tx) => {
      // Query agent config joined with industry vertical
      const [row] = await tx
        .select({
          agentName: agentConfig.agentName,
          ttsModel: agentConfig.ttsModel,
          sttModel: agentConfig.sttModel,
          llmModel: agentConfig.llmModel,
          llmTemperature: agentConfig.llmTemperature,
          agentInstructions: agentConfig.agentInstructions,
          greetingTemplate: agentConfig.greetingTemplate,
          voicemailMessage: agentConfig.voicemailMessage,
          smsAgentInstructions: agentConfig.smsAgentInstructions,
          smsGreetingTemplate: agentConfig.smsGreetingTemplate,
          industryVerticalInstructions: industryVertical.instructions,
          industryVerticalName: industryVertical.name,
        })
        .from(agentConfig)
        .innerJoin(industryVertical, eq(agentConfig.industryVerticalId, industryVertical.id))
        .where(eq(agentConfig.organizationId, organizationId))
        .limit(1);

      const ctx = await buildEntityContext(tx, input.customerId);
      const config = row ?? AGENT_CONFIG_DEFAULTS;
      return { agentConfigData: config, entityContext: ctx };
    });

    if (!entityContext) {
      throw new Error(`Entity ${input.customerId} not found`);
    }

    // Proxy to voice agent via service binding
    const response = await proxyToVoiceAgent(context.env, "/calls/initiate", {
      to: input.to,
      customerId: input.customerId,
      agentConfig: agentConfigData,
      customerContext: entityContext,
    });

    if (!response.ok) {
      throw new Error(`Voice agent error: ${await response.text()}`);
    }

    return response.json();
  });
```

## SMS Initiation Route

```typescript
const initiateSms = protectedProcedure
  .input(z.object({
    customerId: z.string().uuid(),
    to: z.string().regex(/^\+[1-9]\d{1,14}$/),
  }))
  .handler(async ({ input, context }) => {
    const organizationId = getOrganizationId(context);

    // SMS agent fetches its own context from DB — just proxy the identifiers
    const response = await proxyToSmsAgent(context.env, "/sms/initiate", {
      to: input.to,
      customerId: input.customerId,
      organizationId,
    });

    if (!response.ok) throw new Error(`SMS agent error: ${await response.text()}`);
    return response.json();
  });
```

## Call History Route

```typescript
const listCalls = protectedProcedure
  .input(z.object({
    customerId: z.string().uuid(),
  }))
  .handler(async ({ input, context }) => {
    const rows = await context.withRls((tx) =>
      tx
        .select({
          id: call.id,
          callDatetime: call.callDatetime,
          durationSeconds: call.durationSeconds,
          transcript: call.transcript,
          outcomeType: callOutcome.outcome,
          outcomeSummary: callOutcome.summary,
        })
        .from(call)
        .leftJoin(callOutcome, eq(call.id, callOutcome.callId))
        .where(eq(call.customerId, input.customerId))
        .orderBy(desc(call.callDatetime))
    );

    return { calls: rows };
  });
```

## SMS Conversation Routes

```typescript
// List conversations with pagination
const listConversations = protectedProcedure
  .input(z.object({
    status: z.enum(["active", "human_taken", "closed"]).optional(),
    customerId: z.string().uuid().optional(),
    page: z.number().min(1).default(1),
    pageSize: z.number().min(1).max(100).default(20),
  }))
  .handler(async ({ input, context }) => {
    const organizationId = getOrganizationId(context);

    const conditions = [eq(smsConversation.organizationId, organizationId)];
    if (input.status) conditions.push(eq(smsConversation.status, input.status));
    if (input.customerId) conditions.push(eq(smsConversation.customerId, input.customerId));

    const rows = await context.withRls((tx) =>
      tx.select({ /* conversation fields */ })
        .from(smsConversation)
        .innerJoin(customer, eq(smsConversation.customerId, customer.id))
        .where(and(...conditions))
        .orderBy(desc(smsConversation.lastMessageAt))
        .limit(input.pageSize)
        .offset((input.page - 1) * input.pageSize)
    );

    return { conversations: rows };
  });

// Get conversation thread — SINGLE withRls call for both queries
const getThread = protectedProcedure
  .input(z.object({ conversationId: z.string().uuid() }))
  .handler(async ({ input, context }) => {
    const organizationId = getOrganizationId(context);

    const { conversation, messages } = await context.withRls(async (tx) => {
      const [conv] = await tx
        .select({ /* conversation fields */ })
        .from(smsConversation)
        .innerJoin(customer, eq(smsConversation.customerId, customer.id))
        .where(and(
          eq(smsConversation.id, input.conversationId),
          eq(smsConversation.organizationId, organizationId)
        ))
        .limit(1);

      if (!conv) throw new Error("Conversation not found");

      const msgs = await tx
        .select({ /* message fields */ })
        .from(smsMessage)
        .where(and(
          eq(smsMessage.conversationId, input.conversationId),
          eq(smsMessage.organizationId, organizationId)
        ))
        .orderBy(asc(smsMessage.createdAt));

      return { conversation: conv, messages: msgs };
    });

    return { conversation, messages };
  });

// Human takeover — DB update is source of truth, DO notification is best-effort
const takeover = protectedProcedure
  .input(z.object({ conversationId: z.string().uuid() }))
  .handler(async ({ input, context }) => {
    await context.withRls(async (tx) => {
      await tx.update(smsConversation).set({
        status: "human_taken",
        humanAgentId: context.user.id,
        humanTakenAt: new Date(),
      }).where(eq(smsConversation.id, input.conversationId));
    });

    // Fire-and-forget: notify DO (best-effort — reconciles on next inbound)
    await proxyToSmsAgent(context.env, "/sms/do-takeover", {
      conversationId: input.conversationId,
      humanAgentId: context.user.id,
    });

    return { success: true };
  });

// Human release back to AI
const release = protectedProcedure
  .input(z.object({ conversationId: z.string().uuid() }))
  .handler(async ({ input, context }) => {
    await context.withRls(async (tx) => {
      await tx.update(smsConversation).set({
        status: "active",
        humanAgentId: null,
        humanTakenAt: null,
      }).where(eq(smsConversation.id, input.conversationId));
    });

    await proxyToSmsAgent(context.env, "/sms/do-release", {
      conversationId: input.conversationId,
    });

    return { success: true };
  });

// Human sends a message
const sendMessage = protectedProcedure
  .input(z.object({
    conversationId: z.string().uuid(),
    content: z.string().min(1).max(1600),
  }))
  .handler(async ({ input, context }) => {
    // Verify conversation exists and this user has takeover
    const conversation = await context.withRls(async (tx) => {
      const [conv] = await tx.select().from(smsConversation)
        .where(eq(smsConversation.id, input.conversationId)).limit(1);
      return conv ?? null;
    });

    if (!conversation) throw new Error("Conversation not found");
    if (conversation.status !== "human_taken") throw new Error("Not human-controlled");
    if (conversation.humanAgentId !== context.user.id) throw new Error("Not your takeover");

    const response = await proxyToSmsAgent(context.env, "/sms/human-send", {
      conversationId: input.conversationId,
      content: input.content,
      sentByUserId: context.user.id,
    });

    if (!response.ok) throw new Error("Failed to send SMS");
    return { success: true };
  });
```

## Testing Routes

### Voice Test Token

Creates a short-lived HMAC token for browser-based voice testing:

```typescript
const createVoiceTestToken = protectedProcedure
  .handler(async ({ context }) => {
    const token = await signToken(context.env.VOICE_AGENT_INTERNAL_KEY, {
      exp: Date.now() + 60_000, // 60 second TTL
    });
    return { token };
  });
```

The frontend uses this token to open a WebSocket directly to the BrowserSession
DO: `wss://voice-agent/agents/browser-session/{sessionId}?token={token}`

### SMS Test Message

Sends a test message through the SMS agent without creating a real conversation:

```typescript
const sendSmsTestMessage = protectedProcedure
  .input(z.object({
    agentConfig: agentConfigDataSchema,
    customerContext: customerContextSchema.optional(),
    messages: z.array(z.object({
      role: z.enum(["user", "assistant"]),
      content: z.string(),
    })),
    latestMessage: z.string().nullable(),
  }))
  .handler(async ({ input, context }) => {
    const response = await context.env.SMS_AGENT.fetch(
      new Request("https://sms-agent/sms/test-message", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Internal-Key": context.env.SMS_AGENT_INTERNAL_KEY,
        },
        body: JSON.stringify(input),
      })
    );

    const data = await response.json();
    if (!response.ok) throw new Error(data.error ?? "Test message failed");
    return { aiResponse: data.aiResponse };
  });
```

## Gotchas

- **NEVER call `withRls()` multiple times in a single handler.** Each call creates
  a new Neon WebSocket pool with ~2-3s overhead for TCP/TLS handshake. Batch all
  queries in a single `withRls((tx) => { ... })` call.
- **Service bindings, not URLs** — In production, backend-to-worker communication
  uses `env.VOICE_AGENT.fetch(...)` (Cloudflare Service Binding), not `fetch(url)`.
  The URL hostname is arbitrary — only the path matters. Configure service bindings
  in the backend's `wrangler.jsonc`.
- **DB source of truth for takeover state** — The DB update happens first, then the
  DO is notified as best-effort. If the DO notification fails, it reconciles on the
  next inbound message by checking DB state. This prevents split-brain issues.
- **Entity context is sent in the request body** — The backend fetches context
  from the DB and sends it to the agent worker. The agent worker does NOT query
  the database for entity context — it receives it pre-fetched. This keeps the
  agent worker stateless and fast.
- **SMS message length** — Telnyx supports up to 1600 characters per SMS. Longer
  messages are split into segments (each billed separately).
- **HMAC token for voice testing** — Uses the same `VOICE_AGENT_INTERNAL_KEY` as
  the internal key auth, but as an HMAC signing secret. The token expires in 60
  seconds — the frontend must connect the WebSocket within that window.
