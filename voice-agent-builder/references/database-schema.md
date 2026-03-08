# Database Schema Patterns

This reference covers the Drizzle ORM schema patterns for the voice/SMS agent system.
Adapt the table names, enums, and fields to your domain.

## Overview

The schema needs these categories of tables:

1. **Entity table** — The thing the agent communicates about (customer, patient, lead)
2. **Communication records** — Calls and SMS conversations
3. **Outcomes** — What happened on each call/conversation
4. **Agent configuration** — Per-organization AI agent settings
5. **Industry verticals** — Shared lookup table for industry-specific instructions
6. **Supporting tables** — Domain-specific (e.g., transactions, appointments, plans)

## Enums

Define enums for statuses and outcomes. These are domain-specific — here's the pattern:

```typescript
import { pgEnum } from "drizzle-orm/pg-core";

// Entity status — reflects where they are in the communication lifecycle
export const entityStatusEnum = pgEnum("entity_status", [
  "pending",      // Not yet contacted
  "attempted",    // Contact attempted, no resolution
  // ... domain-specific statuses
  "escalated",    // Handed off to human
]);

// Call outcome types — what happened on the call
export const callOutcomeEnum = pgEnum("call_outcome_type", [
  // ... domain-specific outcomes
  "escalated",
  "no_answer",
  "call_back_scheduled",
]);

// SMS conversation status
export const smsConversationStatusEnum = pgEnum("sms_conversation_status", [
  "active",
  "paused",
  "human_taken",
  "completed",
  "escalated",
]);

// SMS message metadata
export const smsMessageSenderEnum = pgEnum("sms_message_sender", [
  "ai",
  "human",
  "customer",
]);

export const smsMessageStatusEnum = pgEnum("sms_message_status", [
  "pending",
  "sent",
  "delivered",
  "failed",
  "received",
]);

// Escalation reasons — domain-specific
export const escalationReasonEnum = pgEnum("escalation_reason", [
  "customer_request",
  // ... domain-specific reasons
]);
```

## Industry Verticals (Global Lookup)

This table stores industry-specific instructions shared across all organizations.
No RLS — all orgs can read from it.

```typescript
export const industryVertical = pgTable("industry_vertical", {
  id: uuid("id").defaultRandom().primaryKey(),
  slug: text("slug").unique().notNull(),
  name: text("name").notNull(),
  instructions: text("instructions").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow()
    .$onUpdate(() => new Date()).notNull(),
});
```

## Entity Table

The core entity the agent communicates with/about. Adapt fields to your domain.

```typescript
export const entity = pgTable(
  "entity",  // rename: customer, patient, lead, etc.
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),

    // Identity fields — adapt to domain
    name: text("name").notNull(),
    contactFirstName: text("contact_first_name").notNull(),
    contactLastName: text("contact_last_name").notNull(),
    email: text("email").notNull(),
    phone: text("phone").notNull(),  // E.164 format

    // Status tracking
    status: entityStatusEnum("status").default("pending").notNull(),
    callAttemptCount: integer("call_attempt_count").default(0).notNull(),
    lastContactedAt: timestamp("last_contacted_at", { withTimezone: true }),

    // Domain-specific fields go here

    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow()
      .$onUpdate(() => new Date()).notNull(),
  },
  (table) => [
    // RLS policy — entities belong to an organization
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("entity_org_id_idx").on(table.organizationId),
    index("entity_status_idx").on(table.status),
  ]
);
```

## Call Table

Records each phone call attempt.

```typescript
export const call = pgTable(
  "call",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    entityId: uuid("entity_id").notNull()  // rename to match your entity
      .references(() => entity.id, { onDelete: "cascade" }),
    callDatetime: timestamp("call_datetime", { withTimezone: true }).notNull(),
    durationSeconds: integer("duration_seconds"),
    transcript: text("transcript"),
    providerCallId: text("provider_call_id").unique(),  // Telnyx CallSid
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("call_org_id_idx").on(table.organizationId),
    index("call_entity_id_idx").on(table.entityId),
    index("call_datetime_idx").on(table.callDatetime),
  ]
);
```

## Call Outcome Table

Records the result of each call. Fields are domain-specific.

```typescript
export const callOutcome = pgTable(
  "call_outcome",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    callId: uuid("call_id").notNull()
      .references(() => call.id, { onDelete: "cascade" }),
    entityId: uuid("entity_id").notNull()
      .references(() => entity.id, { onDelete: "cascade" }),
    outcome: callOutcomeEnum("outcome").notNull(),
    summary: text("summary").notNull(),

    // Domain-specific outcome fields go here
    // e.g., for collections: ddDate, ddAmount, repaymentPlanDetails
    // e.g., for appointments: confirmedDate, rescheduledDate
    // e.g., for sales: qualificationScore, nextSteps

    escalationReason: escalationReasonEnum("escalation_reason"),
    callbackDate: timestamp("callback_date", { withTimezone: true }),
    nextAction: text("next_action"),
    nextActionDate: timestamp("next_action_date", { withTimezone: true }),

    // Approval workflow (optional — for outcomes that need human sign-off)
    approved: boolean("approved").default(false).notNull(),
    approvedBy: text("approved_by").references(() => user.id, { onDelete: "set null" }),
    approvedAt: timestamp("approved_at", { withTimezone: true }),

    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("call_outcome_org_id_idx").on(table.organizationId),
    index("call_outcome_call_id_idx").on(table.callId),
    index("call_outcome_entity_id_idx").on(table.entityId),
  ]
);
```

## Agent Config Table

One row per organization. Stores AI agent settings.

```typescript
export const agentConfig = pgTable(
  "agent_config",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull().unique()
      .references(() => organization.id, { onDelete: "cascade" }),
    industryVerticalId: uuid("industry_vertical_id").notNull()
      .references(() => industryVertical.id, { onDelete: "restrict" }),

    // Agent identity
    agentName: text("agent_name").default("Agent").notNull(),

    // Model configuration
    ttsModel: text("tts_model").default("aura-2-theia-en").notNull(),
    sttModel: text("stt_model").default("flux-general-en").notNull(),
    llmModel: text("llm_model").default("claude-haiku-4-5").notNull(),
    llmTemperature: numeric("llm_temperature", { precision: 2, scale: 1 })
      .default("0.7").notNull(),

    // Voice channel
    agentInstructions: text("agent_instructions").notNull(),
    greetingTemplate: text("greeting_template").notNull(),
    voicemailMessage: text("voicemail_message").notNull(),

    // SMS channel
    smsAgentInstructions: text("sms_agent_instructions").default("").notNull(),
    smsGreetingTemplate: text("sms_greeting_template").notNull(),
    smsMaxMessagesPerConversation: integer("sms_max_messages_per_conversation")
      .default(50).notNull(),

    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow()
      .$onUpdate(() => new Date()).notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("agent_config_org_id_idx").on(table.organizationId),
  ]
);
```

## SMS Tables

```typescript
export const smsConversation = pgTable(
  "sms_conversation",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    entityId: uuid("entity_id").notNull()
      .references(() => entity.id, { onDelete: "cascade" }),
    providerPhoneNumber: text("provider_phone_number").notNull(),
    customerPhoneNumber: text("customer_phone_number").notNull(),
    status: smsConversationStatusEnum("status").default("active").notNull(),
    humanAgentId: text("human_agent_id")
      .references(() => user.id, { onDelete: "set null" }),
    humanTakenAt: timestamp("human_taken_at", { withTimezone: true }),
    lastMessageAt: timestamp("last_message_at", { withTimezone: true })
      .defaultNow().notNull(),
    messageCount: integer("message_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow()
      .$onUpdate(() => new Date()).notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("sms_conversation_org_id_idx").on(table.organizationId),
    index("sms_conversation_entity_id_idx").on(table.entityId),
    index("sms_conversation_status_idx").on(table.status),
    index("sms_conversation_phone_idx").on(
      table.providerPhoneNumber,
      table.customerPhoneNumber
    ),
  ]
);

export const smsMessage = pgTable(
  "sms_message",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    conversationId: uuid("conversation_id").notNull()
      .references(() => smsConversation.id, { onDelete: "cascade" }),
    sender: smsMessageSenderEnum("sender").notNull(),
    content: text("content").notNull(),
    providerMessageId: text("provider_message_id").unique(),
    status: smsMessageStatusEnum("status").default("pending").notNull(),
    sentByUserId: text("sent_by_user_id")
      .references(() => user.id, { onDelete: "set null" }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("sms_message_org_id_idx").on(table.organizationId),
    index("sms_message_conversation_id_idx").on(table.conversationId),
    index("sms_message_provider_id_idx").on(table.providerMessageId),
  ]
);
```

## Escalation Table

Tracks cases handed off to human agents.

```typescript
export const escalation = pgTable(
  "escalation",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    organizationId: text("organization_id").notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    entityId: uuid("entity_id").notNull()
      .references(() => entity.id, { onDelete: "cascade" }),
    callId: uuid("call_id")
      .references(() => call.id, { onDelete: "set null" }),
    escalatedAt: timestamp("escalated_at", { withTimezone: true })
      .defaultNow().notNull(),
    reason: escalationReasonEnum("reason").notNull(),
    assignedTo: text("assigned_to")
      .references(() => user.id, { onDelete: "set null" }),
    status: escalationStatusEnum("status").default("open").notNull(),
    resolutionNotes: text("resolution_notes"),
    resolvedAt: timestamp("resolved_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    ...crudPolicy({
      role: appUser,
      read: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
      modify: sql`${table.organizationId} = current_setting('app.current_org_id', true)`,
    }).filter((p): p is NonNullable<typeof p> => p != null),
    index("escalation_org_id_idx").on(table.organizationId),
    index("escalation_entity_id_idx").on(table.entityId),
    index("escalation_status_idx").on(table.status),
    index("escalation_assigned_to_idx").on(table.assignedTo),
  ]
);
```

## After Creating the Schema

1. Run `pnpm --filter @workspace/db db:generate` to generate migrations
2. Run `pnpm --filter @workspace/db db:migrate` to apply them
3. Run `pnpm --filter @workspace/db db:rls-setup` to grant permissions to the app_user role
4. Update seed data if needed

## Gotchas

- **RLS policies use `current_setting('app.current_org_id', true)`** — the second
  arg `true` means return NULL instead of error if not set. This is critical for
  the app_user role to work correctly.
- **`crudPolicy` returns nullable items** — always filter with `.filter((p) => p != null)`
- **The `industryVertical` table has NO RLS** — it's a global lookup. Grant access
  to app_user via the rls-setup script.
- **Numeric fields for money use `precision: 12, scale: 2`** — don't use float/real.
- **All timestamps use `withTimezone: true`** — critical for correct time handling
  across regions.
