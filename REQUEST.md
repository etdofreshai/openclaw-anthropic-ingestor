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
