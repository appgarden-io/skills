# Deployment & Environment

## Environment Variables

### Voice Agent Worker

| Variable | Description | Example |
|----------|-------------|---------|
| `TELNYX_API_KEY` | Telnyx API key (Bearer auth) | `KEY_xxx` |
| `TELNYX_PUBLIC_KEY` | Telnyx Ed25519 public key (webhook verification) | Hex or base64 string |
| `TELNYX_ACCOUNT_SID` | Telnyx account SID | `a1b63975-...` |
| `TELNYX_TEXML_APP_SID` | TeXML application SID | `2901654610...` |
| `TELNYX_PHONE_NUMBER` | Outbound caller ID (E.164) | `+61240727243` |
| `DEEPGRAM_API_KEY` | Deepgram API key | `dg-xxx` |
| `CF_ACCOUNT_ID` | Cloudflare account ID | `9a7d04f1a...` |
| `CF_GATEWAY_ID` | Cloudflare AI Gateway ID | `my-gateway` |
| `CF_AIG_TOKEN` | Cloudflare AI Gateway token | `aig-xxx` |
| `VOICE_AGENT_INTERNAL_KEY` | Shared secret for backend auth | Random UUID |
| `PUBLIC_URL` | Public URL for webhooks | `https://voice-agent.example.com` |

### SMS Agent Worker

All of the above, plus:

| Variable | Description | Example |
|----------|-------------|---------|
| `DB` | Neon database connection string (pooled) | `postgresql://...` |
| `SMS_AGENT_INTERNAL_KEY` | Shared secret for backend auth | Random UUID |

### Backend

| Variable | Description |
|----------|-------------|
| `VOICE_AGENT_URL` | Voice agent worker URL |
| `VOICE_AGENT_INTERNAL_KEY` | Matches voice agent's internal key |
| `SMS_AGENT_URL` | SMS agent worker URL |
| `SMS_AGENT_INTERNAL_KEY` | Matches SMS agent's internal key |

## Telnyx Setup

### 1. Create a TeXML Application

In the Telnyx portal:
1. Go to Voice > TeXML Applications
2. Create new application
3. Set Voice URL to `{PUBLIC_URL}/call-connected` (updated per deployment)
4. **Set Localisation Country** to the target country (e.g., Australia)
   - Without this, Telnyx validates numbers as US format and rejects non-US calls
5. Note the Application SID → `TELNYX_TEXML_APP_SID`

### 2. Create an Outbound Voice Profile

1. Go to Voice > Outbound Voice Profiles
2. Create new profile
3. **Check Whitelisted Destinations** — must include target country codes
4. **Check Max Destination Rate** — set appropriately or disable
5. Associate your phone number(s) with this profile
6. Note the Profile ID (for debugging, not needed in env)

### 3. Phone Number Configuration

1. Go to Numbers > My Numbers
2. Assign your number to the TeXML application (for voice)
3. For SMS: assign to a Messaging Profile with webhook URL `{PUBLIC_URL}/sms/inbound`
4. Ensure the number supports the channels you need (voice, SMS, or both)

### 4. Get Public Key

1. Go to Account > Public Key
2. Copy the Ed25519 public key → `TELNYX_PUBLIC_KEY`

## Deepgram Setup

1. Create account at console.deepgram.com
2. Create an API key with Agent API access
3. Note the key → `DEEPGRAM_API_KEY`

### Available Models

**TTS (text-to-speech):**
- `aura-2-theia-en` — Natural female voice (recommended)
- `aura-2-andromeda-en` — Natural male voice
- See Deepgram docs for full list

**STT (speech-to-text):**
- `flux-general-en` — Latest general model (requires `version: "v2"`)
- `nova-2` — Previous generation, stable

## Cloudflare AI Gateway Setup

1. Go to Cloudflare dashboard > AI > AI Gateway
2. Create a gateway
3. Note Account ID → `CF_ACCOUNT_ID`
4. Note Gateway ID → `CF_GATEWAY_ID`
5. Create a token → `CF_AIG_TOKEN`

The gateway URL format:
```
https://gateway.ai.cloudflare.com/v1/{CF_ACCOUNT_ID}/{CF_GATEWAY_ID}/anthropic/v1/messages
```

## Development Workflow

### Local Development

```bash
# Terminal 1: Start voice agent
pnpm --filter voice-agent dev    # Runs on :8790

# Terminal 2: Start SMS agent
pnpm --filter sms-agent dev      # Runs on :8791

# Terminal 3: Start backend
pnpm --filter backend dev        # Runs on :8787

# Terminal 4: ngrok tunnels for Telnyx webhooks
# Voice agent:
ngrok http 8790
# Then update PUBLIC_URL in voice-agent .dev.vars

# SMS agent (separate ngrok instance or ngrok config):
ngrok http 8791
# Then update PUBLIC_URL in sms-agent .dev.vars
```

### .dev.vars Format

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

### Deploying

```bash
# Deploy voice agent
pnpm --filter voice-agent deploy

# Deploy SMS agent
pnpm --filter sms-agent deploy

# Push secrets (from .dev.vars.production)
pnpm --filter voice-agent secrets:bulk
pnpm --filter sms-agent secrets:bulk
```

After deploying, update the TeXML app Voice URL in Telnyx portal to point to
the production Worker URL.

## Gotchas

- **ngrok free tier** — URL changes every restart. Update both `PUBLIC_URL` env var
  AND the Telnyx TeXML app Voice URL each time.
- **Telnyx Localisation Country** — If not set on the TeXML app, Telnyx validates
  phone numbers as US format and silently rejects non-US calls with SIP 404.
- **Outbound Voice Profile whitelisting** — Must include destination country codes
  or calls are rejected at the Telnyx level (SIP 404, price $0.00).
- **Separate ngrok tunnels** — Voice agent and SMS agent need separate public URLs
  if running on different ports. Use `ngrok config` for multiple tunnels.
- **Cloudflare Workers dev port conflicts** — Each worker needs its own port. Use
  `dev.port` in wrangler.jsonc to avoid conflicts.
- **Secrets vs .dev.vars** — `.dev.vars` is for local development only. For production,
  use `wrangler secret bulk` or set secrets individually via `wrangler secret put`.
- **AI Gateway token** — Uses `cf-aig-authorization` header (voice, via Deepgram config)
  or `Authorization` header (SMS, via Anthropic SDK `defaultHeaders`). Both work but
  through different code paths.
- **Anchorsite** — In Telnyx, set the anchorsite to the nearest region to your
  Cloudflare Worker region for lowest latency (e.g., Sydney for AU calls).
