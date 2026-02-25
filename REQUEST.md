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
    "source": "local_jsonl",
    "tokens_used_4h_window": 48200,
    "tokens_used_7d_window": 312000,
    "jsonl_path": "~/.claude/projects/",
    "note": "Percentages available once Anthropic usage endpoint is reverse-engineered"
  },
  "api_key_valid": true,
  "cached_at": "2026-02-25T03:00:00Z"
}
```

**Claude Max usage tracking:** Reads local `~/.claude/projects/**/*.jsonl` files written by Claude Code — same approach used by `ccusage` and `Claude-Code-Usage-Monitor`. No Anthropic API access needed for usage stats.

Once the claude.ai settings page usage endpoint is reverse-engineered (network tab in DevTools), add the 4-hour and 7-day percentage fields.

Cache TTL: 5 minutes. Force refresh with `GET /api/status?refresh=true`.
