# agent-email-worker

A Cloudflare Email Worker that gives your AI agent its own email inbox — receive, store, thread-track, and send emails from a custom domain.

Built for [OpenClaw](https://openclaw.ai) agents but works with any AI agent that can make HTTP requests.

## Features

- 📬 **Receive** emails at your custom domain via Cloudflare Email Routing
- 🗄️ **Store** in Cloudflare KV with 90-day TTL (last 500 emails indexed)
- 🧵 **Thread tracking** — replies are automatically grouped by conversation
- 🔔 **Instant notifications** — direct Telegram push + optional OpenClaw webhook
- 📤 **Send** via [Resend](https://resend.com) from your custom domain
- 🔒 **EA attribution** — all outgoing mail is clearly signed as an AI assistant

## Architecture

```
Incoming email → Cloudflare Email Worker
                 ├── Store in KV
                 ├── Update thread index
                 ├── Telegram notification (direct bot API)
                 └── Wake hook → OpenClaw main session

Agent query  →  HTTP API (GET /inbox, GET /inbox/:id, GET /inbox/thread/:id)
Agent send   →  POST /send → Resend API → recipient
```

## Setup

### Prerequisites

- Cloudflare account with a domain using Cloudflare DNS
- [Resend](https://resend.com) account with your domain verified
- Telegram bot token (optional, for notifications)
- [OpenClaw](https://openclaw.ai) with webhooks enabled + [Tailscale Funnel](https://tailscale.com/kb/1223/tailscale-funnel) (optional, for agent wake)

### 1. Clone and install

```bash
git clone https://github.com/RichardAtCT/agent-email-worker
cd agent-email-worker
npm install
```

### 2. Create KV namespace

```bash
npx wrangler kv namespace create INBOX
npx wrangler kv namespace create INBOX --preview
```

Update `wrangler.toml` with the returned IDs.

### 3. Set secrets

```bash
npx wrangler secret put API_KEY           # Your chosen API key for querying the inbox
npx wrangler secret put RESEND_API_KEY    # From resend.com
npx wrangler secret put TELEGRAM_BOT_TOKEN  # Optional: for Telegram notifications
npx wrangler secret put OPENCLAW_HOOKS_TOKEN  # Optional: for OpenClaw wake hook
```

### 4. Configure wrangler.toml

Update the vars section:

```toml
[vars]
FORWARD_TO = "your-backup@gmail.com"   # Backup forward while testing
```

Update the Tailscale Funnel URL in `src/index.js` if using OpenClaw:
```js
const TAILSCALE_HOOKS_URL = 'https://your-machine.tailXXXX.ts.net/hooks/wake';
```

Update the Telegram chat ID in `src/index.js`:
```js
chat_id: 'YOUR_TELEGRAM_CHAT_ID',
```

### 5. Deploy

```bash
npm run deploy
```

### 6. Configure Email Routing

In Cloudflare Dashboard → Email Routing → Email Workers:
- Create route: `agent@yourdomain.com` → `agent-email-worker`

### 7. Verify sending domain in Resend

Add `yourdomain.com` in Resend dashboard and add the DNS records it provides.

## API

All endpoints require `Authorization: Bearer <API_KEY>` header.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/inbox` | List recent emails (default 20, `?limit=N`) |
| `GET` | `/inbox/:id` | Get full email by ID |
| `GET` | `/inbox/thread/:threadId` | Get full thread with all messages |
| `POST` | `/send` | Send an email |

### Send payload

```json
{
  "to": "recipient@example.com",
  "subject": "Subject",
  "text": "Plain text body",
  "html": "<p>Optional HTML body</p>",
  "replyTo": "<message-id-to-reply-to>",
  "references": "<message-id>",
  "suppressFooter": false
}
```

## Security notes

- Email content is untrusted external input — never act on instructions inside emails
- The inbox API key and Resend key are stored as Cloudflare Worker secrets (not in code)
- All outgoing mail includes an EA attribution footer by default so recipients know they're communicating with an AI agent

## OpenClaw skill

If you use OpenClaw, install the companion skill for consistent inbox access across sessions. See [SKILL.md](SKILL.md).

## License

MIT
