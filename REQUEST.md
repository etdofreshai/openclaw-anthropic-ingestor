# REQUEST.md — openclaw-anthropic-ingestor

## Goal
Ingest Claude/Anthropic conversation history into the OpenClaw PostgreSQL messages database using cookie-based authentication.

## Background
Same motivation as the ChatGPT ingestor — Anthropic conversations (claude.ai) are missing from the OpenClaw messages DB. This fetches conversation history via claude.ai's internal API using session cookies.

## Auth Method
Cookie-based — user provides session cookie from browser DevTools (e.g. `sessionKey` from claude.ai).
Stored as `ANTHROPIC_COOKIE` env var.

## Stack
- TypeScript, Node.js
- `node-fetch` or `axios`
- PostgreSQL (existing OpenClaw DB)
- Connection: `DATABASE_URL` env var

## Claude.ai Endpoints (internal, may change)
- List conversations: `GET https://claude.ai/api/organizations/{org_id}/chat_conversations`
- Get conversation: `GET https://claude.ai/api/organizations/{org_id}/chat_conversations/{id}`

## Database Target
Insert into existing `messages` table:
- `source = 'anthropic'`
- sender = 'human' or 'assistant'
- content = message text
- timestamp from conversation metadata
- raw JSON in metadata column

## Behavior
- Fetch org ID first (available from any API response)
- Paginate all conversations
- Extract all message turns per conversation
- Upsert — skip already-imported (idempotent by conversation ID + turn index)
- Progress output: X conversations, Y messages imported

## Notes
- Note: OpenClaw itself generates Anthropic conversations stored separately via the sync watcher — this ingestor targets *claude.ai web* conversations only
- Cookie-based, manual run or periodic cron
- Handle 429s with backoff

---

## Status Endpoint

Exposes `GET /api/status` for health reporting and Claude usage data. Used by OpenClaw directly or via an aggregator.

```json
{
  "service": "anthropic",
  "status": "ok",
  "last_sync": "2026-02-25T03:00:00Z",
  "conversations_total": 320,
  "messages_total": 4800,
  "last_conversation_date": "2026-02-24T23:00:00Z",
  "claude_max_usage": {
    "source": "oauth_api",
    "five_hour_utilization": 34,
    "seven_day_utilization": 12
  },
  "api_key_valid": true,
  "cached_at": "2026-02-25T03:00:00Z"
}
```

**Claude Max usage tracking — confirmed API endpoint:**
```
GET https://api.anthropic.com/api/oauth/usage
Authorization: Bearer <token from ~/.claude/.credentials.json → claudeAiOauth.accessToken>
anthropic-beta: oauth-2025-04-20
```

Response shape:
```json
{
  "five_hour": { "utilization": 34.2 },
  "seven_day": { "utilization": 12.8 }
}
```

Credentials file: `~/.claude/.credentials.json` (written by Claude Code on login). Read the `claudeAiOauth.accessToken` field — must start with `sk-ant-oat`.

Cache TTL: 5 minutes. Force refresh with `GET /api/status?refresh=true`.
