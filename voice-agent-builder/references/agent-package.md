# Agent Package (`packages/agent`)

The shared agent package contains all AI logic used by both voice and SMS workers.
This is a workspace package — not a standalone app.

## Package Structure

```
packages/agent/
├── package.json
├── src/
│   ├── index.ts                      # Barrel export
│   ├── agent-config.ts               # Config types, defaults, DB fetch
│   ├── entity-context.ts             # Entity data fetch + formatting
│   ├── deepgram-bridge.ts            # Audio bridge to Deepgram Agent API
│   ├── deepgram-config.ts            # Deepgram config builder
│   ├── voice-agent-instructions.ts   # Base voice system prompt
│   └── sms-agent-instructions.ts     # Base SMS system prompt
└── tsconfig.json
```

## package.json

```json
{
  "name": "@workspace/agent",
  "version": "0.0.0",
  "private": true,
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

## Agent Config (`agent-config.ts`)

Defines config types, defaults, and a DB fetch function.

```typescript
import { agentConfig, eq, getDb, industryVertical } from "@workspace/db";

export interface AgentConfigData {
  agentName: string;
  ttsModel: string;
  sttModel: string;
  llmModel: string;
  llmTemperature: string;
  agentInstructions: string;
  greetingTemplate: string;
  voicemailMessage: string;
  smsAgentInstructions: string;
  smsGreetingTemplate: string;
  smsMaxMessagesPerConversation: number;
  industryVerticalInstructions: string;
  industryVerticalName: string;
}

export const AGENT_CONFIG_DEFAULTS: AgentConfigData = {
  agentName: "Agent",  // Customize per domain
  ttsModel: "aura-2-theia-en",
  sttModel: "flux-general-en",
  llmModel: "claude-haiku-4-5",
  llmTemperature: "0.7",
  agentInstructions: "",
  greetingTemplate: "Hello?",
  voicemailMessage: "Hi, we were trying to reach you. Please call us back.",
  smsAgentInstructions: "",
  smsGreetingTemplate: "Hi {contactFirstName}, this is {agentName}.",
  smsMaxMessagesPerConversation: 50,
  industryVerticalInstructions: "",
  industryVerticalName: "",
};

export async function fetchAgentConfig(
  dbUrl: string,
  organizationId: string
): Promise<AgentConfigData> {
  const db = getDb(dbUrl);
  const [row] = await db
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
      smsMaxMessagesPerConversation: agentConfig.smsMaxMessagesPerConversation,
      industryVerticalInstructions: industryVertical.instructions,
      industryVerticalName: industryVertical.name,
    })
    .from(agentConfig)
    .innerJoin(industryVertical, eq(agentConfig.industryVerticalId, industryVertical.id))
    .where(eq(agentConfig.organizationId, organizationId))
    .limit(1);

  return row ?? AGENT_CONFIG_DEFAULTS;
}

export function buildGreeting(
  config: AgentConfigData,
  contactFirstName?: string
): string {
  return config.greetingTemplate
    .replaceAll("{agentName}", config.agentName)
    .replaceAll("{contactFirstName}", contactFirstName ?? "");
}

export function buildSmsGreeting(
  config: AgentConfigData,
  contactFirstName?: string
): string {
  return config.smsGreetingTemplate
    .replaceAll("{agentName}", config.agentName)
    .replaceAll("{contactFirstName}", contactFirstName ?? "");
}
```

## Entity Context (`entity-context.ts`)

Fetches entity data from the database and formats it as a markdown string for the
LLM system prompt. Adapt the fields and formatting to your domain.

```typescript
import { and, eq, getDb /* ...tables... */ } from "@workspace/db";
import type { AgentConfigData } from "./agent-config.js";
import { SMS_AGENT_INSTRUCTIONS } from "./sms-agent-instructions.js";
import { VOICE_AGENT_INSTRUCTIONS } from "./voice-agent-instructions.js";

// Define your context interface — what data the agent needs per call
export interface EntityContext {
  entity: {
    id: string;
    name: string;
    contactFirstName: string;
    contactLastName: string;
    phone: string;
    status: string;
    // ... domain-specific fields
  };
  // ... related data arrays (history, outcomes, etc.)
}

export async function fetchEntityContext(
  dbUrl: string,
  entityId: string,
  organizationId: string
): Promise<EntityContext> {
  const db = getDb(dbUrl);

  // Fetch the entity
  const entityRow = await db.query.entity.findFirst({
    where: and(eq(entity.id, entityId), eq(entity.organizationId, organizationId)),
  });
  if (!entityRow) throw new Error(`Entity ${entityId} not found`);

  // Fetch related data in parallel or sequentially
  // ... adapt to your domain

  return {
    entity: { /* map fields */ },
    // ... related data
  };
}

// Format entity context as markdown for the LLM
function formatEntityContext(ctx: EntityContext): string {
  const lines = [
    "# Entity Context",
    `Name: ${ctx.entity.name}`,
    `Contact: ${ctx.entity.contactFirstName} ${ctx.entity.contactLastName}`,
    `Phone: ${ctx.entity.phone}`,
    `Status: ${ctx.entity.status}`,
    // ... format all relevant data
  ];
  return lines.join("\n");
}
```

### System Prompt Builder

The prompt builder composes the three layers + entity context:

```typescript
const PREAMBLE =
  "You are an AI agent. Your behavior is governed by three layers of instructions " +
  "below, in descending order of precedence: base > industry > customer. " +
  "If instructions conflict, the higher-precedence layer wins.";

function wrapXml(tag: string, content: string): string {
  if (content.length === 0) return `<${tag}></${tag}>`;
  return `<${tag}>\n${content}\n</${tag}>`;
}

export function buildSystemPrompt(
  entityData?: EntityContext,
  configData?: AgentConfigData,
  channel: "voice" | "sms" = "voice"
): string {
  const baseInstructions =
    channel === "sms" ? SMS_AGENT_INSTRUCTIONS : VOICE_AGENT_INSTRUCTIONS;
  const industryInstructions = configData?.industryVerticalInstructions ?? "";
  const customerInstructions =
    channel === "sms"
      ? (configData?.smsAgentInstructions ?? "")
      : (configData?.agentInstructions ?? "");

  const parts = [
    PREAMBLE,
    wrapXml("base", baseInstructions),
    wrapXml("industry", industryInstructions),
    wrapXml("customer", customerInstructions),
  ];

  if (entityData) {
    parts.push(formatEntityContext(entityData));
  }

  return parts.join("\n\n");
}
```

## Deepgram Bridge (`deepgram-bridge.ts`)

The bridge manages the bidirectional WebSocket connection to Deepgram's Agent API.
It handles audio buffering, keepalive, and event routing.

```typescript
import { AgentEvents, type AgentLiveSchema, createClient } from "@deepgram/sdk";

type AudioEncoding = "mulaw" | "linear16";

const BYTES_PER_SAMPLE: Record<AudioEncoding, number> = {
  linear16: 2,
  mulaw: 1,
};

const BUFFER_DURATION_S = 0.2;  // 200ms buffer
const KEEPALIVE_INTERVAL_MS = 8000;  // 8 seconds

export interface DeepgramAgentBridgeCallbacks {
  onAudio: (audio: Buffer) => void;
  onReady: () => void;
  onTranscript: (role: string, content: string) => void;
  onSpeakingChange: (
    event: "user_started" | "agent_started",
    data?: Record<string, unknown>
  ) => void;
  onError: (error: Error) => void;
  onClose: () => void;
}

export interface DeepgramAgentBridgeOptions {
  audioEncoding: AudioEncoding;
  sampleRate: number;
  deepgramApiKey: string;
  config: AgentLiveSchema;
  callbacks: DeepgramAgentBridgeCallbacks;
}

export class DeepgramAgentBridge {
  private readonly audioEncoding: AudioEncoding;
  private readonly sampleRate: number;
  private readonly bufferSize: number;
  private readonly callbacks: DeepgramAgentBridgeCallbacks;

  private connection: ReturnType<ReturnType<typeof createClient>["agent"]> | undefined;
  private audioBuffer = new Uint8Array(0);
  private lastKeepAlive = 0;
  private keepAliveTimer: ReturnType<typeof setInterval> | undefined;

  constructor(options: DeepgramAgentBridgeOptions) {
    this.audioEncoding = options.audioEncoding;
    this.sampleRate = options.sampleRate;
    this.callbacks = options.callbacks;

    // Buffer size = sampleRate * bytesPerSample * duration
    this.bufferSize =
      this.sampleRate * BYTES_PER_SAMPLE[this.audioEncoding] * BUFFER_DURATION_S;

    this.connect(options.deepgramApiKey, options.config);
  }

  sendAudio(data: Uint8Array): void {
    if (!this.connection) return;

    // Append to buffer
    const combined = new Uint8Array(this.audioBuffer.length + data.length);
    combined.set(this.audioBuffer);
    combined.set(data, this.audioBuffer.length);
    this.audioBuffer = combined;

    // Flush full chunks to Deepgram
    while (this.audioBuffer.length >= this.bufferSize) {
      const chunk = this.audioBuffer.slice(0, this.bufferSize);
      this.audioBuffer = this.audioBuffer.slice(this.bufferSize);
      this.connection.send(
        chunk.buffer.slice(chunk.byteOffset, chunk.byteOffset + chunk.byteLength)
      );
    }

    this.maybeSendKeepAlive();
  }

  disconnect(): void {
    if (this.keepAliveTimer) {
      clearInterval(this.keepAliveTimer);
      this.keepAliveTimer = undefined;
    }
    if (this.connection) {
      this.connection.disconnect();
      this.connection = undefined;
    }
    this.audioBuffer = new Uint8Array(0);
  }

  private connect(apiKey: string, config: AgentLiveSchema): void {
    const client = createClient(apiKey);
    let connection: ReturnType<ReturnType<typeof createClient>["agent"]>;

    try {
      connection = client.agent();
    } catch (err) {
      this.callbacks.onError(err instanceof Error ? err : new Error(String(err)));
      return;
    }
    this.connection = connection;

    connection.on(AgentEvents.Open, () => {
      connection.configure(config);
    });

    connection.on(AgentEvents.SettingsApplied, () => {
      this.lastKeepAlive = Date.now();
      this.startKeepAlive();
      this.callbacks.onReady();
    });

    connection.on(AgentEvents.Audio, (data: Buffer) => {
      this.callbacks.onAudio(data);
    });

    connection.on(AgentEvents.UserStartedSpeaking, () => {
      this.callbacks.onSpeakingChange("user_started");
    });

    connection.on(AgentEvents.AgentStartedSpeaking, (data) => {
      this.callbacks.onSpeakingChange("agent_started", {
        totalLatency: data.total_latency,
        ttsLatency: data.tts_latency,
        tttLatency: data.ttt_latency,
      });
    });

    connection.on(AgentEvents.ConversationText, (data) => {
      this.callbacks.onTranscript(data.role, data.content);
    });

    connection.on(AgentEvents.Error, (error) => {
      this.callbacks.onError(
        error instanceof Error ? error : new Error(String(error.message ?? error))
      );
    });

    connection.on(AgentEvents.Close, () => {
      if (this.keepAliveTimer) {
        clearInterval(this.keepAliveTimer);
        this.keepAliveTimer = undefined;
      }
      this.connection = undefined;
      this.callbacks.onClose();
    });
  }

  private startKeepAlive(): void {
    this.keepAliveTimer = setInterval(() => {
      this.connection?.keepAlive();
      this.lastKeepAlive = Date.now();
    }, KEEPALIVE_INTERVAL_MS);
  }

  private maybeSendKeepAlive(): void {
    if (Date.now() - this.lastKeepAlive < KEEPALIVE_INTERVAL_MS) return;
    this.lastKeepAlive = Date.now();
    this.connection?.keepAlive();
  }
}
```

## Deepgram Config Builder (`deepgram-config.ts`)

Builds the configuration object sent to Deepgram Agent API on connection.

```typescript
import type { AgentLiveSchema } from "@deepgram/sdk";

export interface DeepgramConfigOptions {
  audioEncoding: "mulaw" | "linear16";
  sampleRate: number;
  systemPrompt: string;
  greeting: string;
  ttsModel: string;
  sttModel: string;
  llmModel: string;
  llmTemperature: number;
  env: {
    CF_ACCOUNT_ID: string;
    CF_GATEWAY_ID: string;
    CF_AIG_TOKEN: string;
  };
}

export function buildDeepgramConfig(options: DeepgramConfigOptions): AgentLiveSchema {
  return {
    audio: {
      input: { encoding: options.audioEncoding, sample_rate: options.sampleRate },
      output: {
        encoding: options.audioEncoding,
        sample_rate: options.sampleRate,
        container: "none" as const,
      },
    },
    agent: {
      speak: {
        provider: { type: "deepgram" as const, model: options.ttsModel },
      },
      listen: {
        provider: {
          type: "deepgram" as const,
          // Flux models require the v2 API
          ...(options.sttModel.startsWith("flux-") && { version: "v2" }),
          model: options.sttModel,
        },
      },
      think: {
        provider: {
          type: "anthropic" as const,
          model: options.llmModel,
          temperature: options.llmTemperature,
        },
        endpoint: {
          url: `https://gateway.ai.cloudflare.com/v1/${options.env.CF_ACCOUNT_ID}/${options.env.CF_GATEWAY_ID}/anthropic/v1/messages`,
          headers: {
            "cf-aig-authorization": `Bearer ${options.env.CF_AIG_TOKEN}`,
          },
        },
        prompt: options.systemPrompt,
      },
      greeting: options.greeting,
    },
  };
}
```

## Validation Schemas (`schemas.ts`)

All payloads exchanged between the backend, workers, and Durable Objects should be
validated with these Zod schemas. Adapt the entity fields to your domain.

```typescript
import { z } from "zod/v4";

// Date helpers for serialized dates from JSON
const dateString = z.string().transform((s) => new Date(s));
const nullableDateString = z.string().nullable()
  .transform((s) => (s ? new Date(s) : null));

// Agent config — matches the AgentConfigData interface
export const agentConfigDataSchema = z.object({
  agentName: z.string(),
  ttsModel: z.string(),
  sttModel: z.string(),
  llmModel: z.string(),
  llmTemperature: z.string(),
  agentInstructions: z.string(),
  greetingTemplate: z.string(),
  voicemailMessage: z.string(),
  smsAgentInstructions: z.string(),
  smsGreetingTemplate: z.string(),
  smsMaxMessagesPerConversation: z.number(),
  industryVerticalInstructions: z.string().default(""),
  industryVerticalName: z.string().default(""),
});

// Entity context — adapt to your domain's data model
export const entityContextSchema = z.object({
  entity: z.object({
    id: z.string(),
    name: z.string(),
    contactFirstName: z.string(),
    contactLastName: z.string(),
    phone: z.string(),
    status: z.string(),
    // ... domain-specific fields
  }),
  // ... related data arrays with their own schemas
});

// Start message — sent by browser to BrowserSession DO
export const startMessageSchema = z.object({
  type: z.literal("start"),
  agentConfig: agentConfigDataSchema,
  customerContext: entityContextSchema.optional(),
});

// Init body — sent to CallSession/SmsSession DOs before connection
export const initBodySchema = z.object({
  customerContext: entityContextSchema,
  agentConfig: agentConfigDataSchema,
});
```

These schemas serve double duty: runtime validation AND type inference. The
`AgentConfigData` interface should match `z.infer<typeof agentConfigDataSchema>`.

## Mock Entity Context (for Browser Testing)

Used by BrowserSession when no real entity is provided in the `start` message.
Create realistic test data that exercises the main conversation flows.

```typescript
export function buildMockEntityContext(): EntityContext {
  return {
    entity: {
      id: "mock-001",
      name: "Test Entity",
      contactFirstName: "Sarah",
      contactLastName: "Mitchell",
      phone: "+61412345678",
      status: "active",
      // ... realistic domain-specific test data
    },
    // ... related data that covers typical conversation paths
  };
}
```

Design mock data to exercise the agent's main decision branches — e.g., include
some history that triggers different resolution paths, a failed previous interaction,
and an active item that the agent should discuss.

## Package Exports (`index.ts`)

The barrel export should include everything needed by both workers:

```typescript
// Agent config
export { AGENT_CONFIG_DEFAULTS, type AgentConfigData, buildGreeting, buildSmsGreeting, fetchAgentConfig } from "./agent-config.js";

// Entity context + system prompt
export { buildSystemPrompt, type EntityContext, fetchEntityContext } from "./entity-context.js";

// Deepgram
export { DeepgramAgentBridge, type DeepgramAgentBridgeCallbacks, type DeepgramAgentBridgeOptions } from "./deepgram-bridge.js";
export { buildDeepgramConfig, type DeepgramConfigOptions } from "./deepgram-config.js";

// Schemas (for validation at trust boundaries)
export { agentConfigDataSchema, entityContextSchema, initBodySchema, startMessageSchema } from "./schemas.js";

// Testing utilities
export { buildMockEntityContext } from "./mock-entity-context.js";

// Base instructions (for reference, not typically imported directly)
export { VOICE_AGENT_INSTRUCTIONS } from "./voice-agent-instructions.js";
export { SMS_AGENT_INSTRUCTIONS } from "./sms-agent-instructions.js";
```

## Gotchas

- **Deepgram SDK `AgentLiveSchema` type** — The listen provider type is open
  (`{ type: string } & Record<string, unknown>`), so extra fields like `version: "v2"`
  must be spread in. There's no typed `version` field.
- **Flux STT models** — Deepgram's newer Flux models (e.g., `flux-general-en`) require
  `version: "v2"` on the listen provider. Standard models (e.g., `nova-2`) don't.
- **Audio encoding** — Telnyx sends the encoding in the `start` event. Match it exactly
  when configuring Deepgram input/output. Mismatched encoding = garbled audio.
- **`container: "none"`** — Critical for output audio config. Without this, Deepgram
  wraps audio in a container format that Telnyx can't play.
- **Buffer size calculation** — `sampleRate * bytesPerSample * bufferDuration`. For
  linear16 @ 16kHz with 200ms buffer: `16000 * 2 * 0.2 = 6400 bytes`.
- **Keepalive interval** — 8 seconds is Deepgram's recommended interval. Too long and
  the connection times out during silent pauses in conversation.
- **AI Gateway URL format** — Must be
  `https://gateway.ai.cloudflare.com/v1/{ACCOUNT_ID}/{GATEWAY_ID}/anthropic/v1/messages`
  with `cf-aig-authorization` header (not standard `Authorization`).
