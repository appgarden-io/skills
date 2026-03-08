---
name: voice-agent-builder
description: >
  Build production AI voice and SMS agent systems on Cloudflare Workers using Telnyx
  for telephony, Deepgram Agent API for real-time speech, and Claude for conversation
  intelligence. Use this skill whenever the user wants to build a voice agent, phone bot,
  outbound calling system, SMS chatbot, AI phone assistant, conversational IVR,
  automated calling, or any real-time voice/text AI communication system. Also use it
  when the user mentions Telnyx, Deepgram Agent API, TeXML, or media stream WebSockets
  in the context of building an application. This skill covers the complete architecture
  from PSTN call initiation through real-time audio bridging to AI-powered conversation.
---

# Voice Agent Builder

Build a production AI voice and SMS agent system. This skill guides you through creating
a complete communication platform where an AI agent can make outbound phone calls, handle
real-time voice conversations, detect answering machines, leave voicemails, and conduct
SMS conversations — all deployed on Cloudflare Workers.

## Architecture Overview

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐     ┌───────────┐
│  Backend     │────>│  Voice Agent  │────>│  Telnyx PSTN    │────>│  Phone    │
│  (oRPC)      │     │  (CF Worker)  │     │  (TeXML + Media │     │  Network  │
│              │     │              │     │   Stream WS)    │     │           │
└─────────────┘     └──────┬───────┘     └─────────────────┘     └───────────┘
                           │
                    ┌──────┴───────┐
                    │ Durable Object│
                    │ (CallSession) │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐     ┌───────────┐
                    │  Deepgram     │────>│  Claude    │
                    │  Agent API    │     │  (via CF   │
                    │  (STT+TTS)   │     │  AI GW)    │
                    └──────────────┘     └───────────┘

┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Backend     │────>│  SMS Agent    │────>│  Telnyx SMS API │
│  (oRPC)      │     │  (CF Worker)  │     │                 │
│              │     │              │     └─────────────────┘
└─────────────┘     └──────┬───────┘
                           │
                    ┌──────┴───────┐     ┌───────────┐
                    │ Durable Object│────>│  Claude    │
                    │ (SmsSession)  │     │  (Anthropic│
                    └──────────────┘     │   SDK)     │
                                         └───────────┘
```

## When to Use This Skill

Use when building any system that needs:
- AI-powered outbound phone calls
- Real-time voice conversation with an AI agent
- Answering machine detection and voicemail
- AI-driven SMS conversations with human takeover
- Telnyx TeXML integration for call control
- Deepgram Agent API for speech-to-text + text-to-speech
- Cloudflare Workers + Durable Objects for stateful call sessions

## Prerequisites

This skill assumes the user already has:
- A pnpm monorepo with workspace packages
- A database package using Drizzle ORM + Neon Postgres
- An API layer (e.g., oRPC, tRPC, or Hono routes)
- Authentication (e.g., Better Auth)
- A Telnyx account with: API key, account SID, TeXML app, phone number(s)
- A Deepgram account with API key
- A Cloudflare account with AI Gateway configured

If any of these are missing, help the user set them up before proceeding with the
voice agent build.

## Required Packages (Quick Reference)

Read `references/packages.md` for full details including exact versions, workspace
package structure, environment files, and service binding configuration.

### npm Dependencies

| Package | Version | Where | Purpose |
|---------|---------|-------|---------|
| `hono` | `^4.11.3` | All workers | HTTP framework |
| `zod` | `^4.3.5` | All packages | Runtime validation |
| `agents` | `0.5.0` | Voice + SMS workers | Cloudflare Agents SDK (Durable Objects) |
| `@deepgram/sdk` | `^4.11.3` | Agent package | TypeScript types for Agent API |
| `@anthropic-ai/sdk` | `^0.71.2` | SMS worker only | Direct Claude API calls |
| `drizzle-orm` | `^0.45.1` | DB package | Database ORM |
| `@neondatabase/serverless` | `^1.0.0` | DB package | Neon Postgres driver |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `wrangler` | `^4.57.0` | Cloudflare Workers CLI |
| `@cloudflare/workers-types` | `^4.20250823.0` | Worker type definitions |
| `typescript` | `^5.9.2` | TypeScript compiler |
| `vitest` | `^3.2.4` | Testing framework |
| `drizzle-kit` | `^0.31.9` | Migration generation |

### Environment Variables

Each agent worker needs a `.dev.vars` file (local) and production secrets (via
`wrangler secret bulk`). See `references/deployment.md` for the complete list.

**Voice Agent** (11 vars): `TELNYX_API_KEY`, `TELNYX_PUBLIC_KEY`,
`TELNYX_ACCOUNT_SID`, `TELNYX_TEXML_APP_SID`, `TELNYX_PHONE_NUMBER`,
`DEEPGRAM_API_KEY`, `CF_ACCOUNT_ID`, `CF_GATEWAY_ID`, `CF_AIG_TOKEN`,
`VOICE_AGENT_INTERNAL_KEY`, `PUBLIC_URL`

**SMS Agent** (all of the above + 2): `DB` (Neon connection string),
`SMS_AGENT_INTERNAL_KEY`

**Backend** (4 vars): `VOICE_AGENT_INTERNAL_KEY`, `SMS_AGENT_INTERNAL_KEY` +
service bindings `VOICE_AGENT`, `SMS_AGENT` (configured in wrangler.jsonc)

## Conversation Guide

Before writing any code, gather this information from the user. The answers shape
every component — from the system prompt to the database schema.

### 1. Domain & Identity

- **What domain/industry is this for?** (e.g., healthcare appointment reminders,
  collections, sales follow-up, customer support, survey)
- **What is the agent's name?** What company does it represent?
- **What is the agent's personality/tone?** (e.g., warm and empathetic, professional
  and efficient, casual and friendly)

### 2. Conversation Objectives

- **What is the primary goal of each call/SMS?** (e.g., collect payment, confirm
  appointment, gather feedback, qualify lead)
- **What are the resolution options?** List them in priority order — the agent will
  work through these sequentially. (e.g., for collections: full payment → payment
  plan → escalate; for appointments: confirm → reschedule → cancel)
- **When should the agent escalate to a human?** Define the triggers (e.g., customer
  disputes, emotional distress, explicit request, unable to resolve)

### 3. Compliance & Rules

- **What regulations apply?** (e.g., contact hour restrictions, frequency limits,
  opt-out handling, privacy rules, industry-specific regulations)
- **What should the agent never say or do?** (e.g., never provide medical/legal/
  financial advice, never threaten, never disclose to third parties)

### 4. Data Model

- **What entity does the agent call about?** (e.g., customer with outstanding debt,
  patient with appointment, lead with inquiry). What fields describe this entity?
- **What context does the agent need during a call?** (e.g., outstanding invoices,
  appointment details, order history, previous call outcomes)
- **What outcomes should be recorded after a call?** (e.g., payment agreed,
  appointment confirmed, callback scheduled, escalated)

### 5. Channel Configuration

- **Which channels?** Voice only, SMS only, or both?
- **Voicemail behavior?** What message to leave if a machine answers?
- **SMS greeting template?** What's the first message?
- **Voice greeting?** What does the agent say when the call connects?

### 6. Infrastructure

- **Telnyx setup:** Do they have a TeXML app, outbound voice profile, and phone
  number(s) configured? What country/region?
- **Deepgram preferences:** Which TTS voice? Which STT model?
- **LLM model:** Which Claude model? (default: claude-haiku-4-5 for low latency)

## Build Sequence

Follow this order. Each step has a detailed reference file.

### Step 1: Database Schema
Add tables for the domain entity, communication records, outcomes, and agent config.
Read `references/database-schema.md` for the Drizzle schema patterns.

### Step 2: Shared Agent Package
Create the `packages/agent` workspace package with:
- System prompt builder (layered: base → industry → customer)
- Entity context formatter (fetches and formats data for the LLM)
- Deepgram bridge (bidirectional audio over WebSocket)
- Deepgram config builder
- Agent config types and defaults
Read `references/agent-package.md` for implementation details.

### Step 3: Telnyx Package
Create the `packages/telnyx` workspace package with:
- Outbound call creation (TeXML API)
- Call redirect (for voicemail routing)
- SMS send
- Webhook signature verification (Ed25519)
- TeXML builders (media stream, voicemail)
- Type definitions (schemas, stream messages)
Read `references/telnyx-package.md` for implementation details.

### Step 4: Voice Agent Worker
Create the `apps/voice-agent` Cloudflare Worker with:
- HTTP routes (call initiation, webhooks, voicemail)
- CallSession Durable Object (WebSocket ↔ Deepgram bridge)
- Wrangler configuration
Read `references/voice-agent-worker.md` for implementation details.

### Step 5: SMS Agent Worker (if needed)
Create the `apps/sms-agent` Cloudflare Worker with:
- HTTP routes (initiate, inbound webhook, status callback, human takeover/release)
- SmsSession Durable Object (conversation state, AI response generation)
- Wrangler configuration
Read `references/sms-agent-worker.md` for implementation details.

### Step 6: Backend API Routes
Add routes to the existing API layer for:
- Call initiation (fetches context, proxies to voice agent)
- SMS initiation (fetches context, proxies to SMS agent)
- Call/conversation history queries
Read `references/backend-routes.md` for implementation details.

### Step 7: Environment & Deployment
Configure environment variables, secrets, and deployment.
Read `references/deployment.md` for the complete setup.

## System Prompt Architecture

The prompt system uses three layers of precedence (base > industry > customer):

```xml
<!-- Preamble: sets the layering contract -->
You are an AI agent. Your behavior is governed by three layers of instructions
below, in descending order of precedence: base > industry > customer.
If instructions conflict, the higher-precedence layer wins.

<base>
  <!-- Core behavior, compliance, escalation rules, conversation flow -->
  <!-- This is the domain-specific instruction set -->
</base>

<industry>
  <!-- Industry vertical tweaks (from industryVertical table) -->
  <!-- e.g., healthcare-specific language, financial regulations -->
</industry>

<customer>
  <!-- Organization-specific customization (from agentConfig table) -->
  <!-- e.g., company-specific policies, branding, product details -->
</customer>

<!-- Live context: entity data formatted as markdown -->
# Entity Context
Name: ...
Details: ...
History: ...
```

When building the base prompt for a new domain, use this structure:

```
<rules> — Explain the three-layer system
<identity> — Who the agent is, personality, tone
<tone> — Communication style guidelines
<agent_context> — What data the agent receives per call
<objective> — Primary goal + resolution options in priority order
<identity_verification> — How to verify the caller (voice only)
<opening> — How to start the conversation
<negotiation_guidelines> — How to handle back-and-forth
<escalate_to_human> — When and how to escalate
<compliance> — Regulatory requirements
<conversation_close> — How to end successfully
```

For SMS, add: `<opening_message>`, `<opt_out>`, `<non_response>`, `<delivery_failure>`.

## Key Patterns & Gotchas

### Audio Bridging
- Buffer 200ms of audio before forwarding to Deepgram (balances latency vs reliability)
- Send keepalive to Deepgram every 8 seconds to prevent timeout on silent calls
- Handle barge-in by sending `{ event: "clear", stream_id }` to Telnyx

### Telnyx TeXML
- Use `<Connect><Stream>` for bidirectional media stream WebSocket
- AMD (answering machine detection) requires both `machineDetection: "DetectMessageEnd"`
  AND `asyncAmd: true` for immediate callback
- Phone numbers must be E.164 format (`+[country code][number]`)
- Webhook signature verification uses Ed25519 (not HMAC)

### Durable Objects
- Each call/conversation gets its own DO instance (keyed by callId/conversationId)
- Store entity context + agent config in DO state before the call connects
- Voice: `hibernate: false` (needs persistent WebSocket). SMS: `hibernate: true` (request-based)

### Cloudflare AI Gateway
- Route Claude API calls through AI Gateway for logging, caching, rate limiting
- Voice uses Deepgram's built-in LLM provider pointing at the gateway URL
- SMS uses the Anthropic SDK directly with gateway as `baseURL`

### Database
- Never call `withRls()` multiple times in a single handler — each call creates a new
  connection pool with ~2-3s overhead. Batch all queries in a single transaction.

## Reference Files

| File | When to read |
|------|-------------|
| `references/packages.md` | Package versions, workspace structure, env files, service bindings |
| `references/database-schema.md` | Setting up tables for the domain |
| `references/agent-package.md` | Building the shared agent logic (prompts, Deepgram, config) |
| `references/telnyx-package.md` | Building the Telnyx integration (calls, SMS, webhooks) |
| `references/middleware.md` | Internal key auth, HMAC tokens, request logger, error handler |
| `references/voice-agent-worker.md` | Building the voice agent CF Worker + BrowserSession testing |
| `references/sms-agent-worker.md` | Building the SMS agent CF Worker + test handler |
| `references/backend-routes.md` | API routes, proxy helpers, testing routes |
| `references/deployment.md` | Environment variables, Telnyx/Deepgram setup, deployment |
