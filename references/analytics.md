# Voice Agent Analytics & Diagnostics

Once the voice agent is in production you NEED a way to read what visitors actually said and which tools fired. Heuristic guessing about why "it didn't work" is whack-a-mole; reading one real session usually tells you exactly what's wrong.

This is private analytics — the operator reviews their own data. No public surface.

## Storage shape

Reuse the existing Upstash client config (no new env vars beyond `UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN`).

```
voice:session:<id>           HASH   { ip, started_at, ended_at, voice, duration_seconds, end_reason, agent_id, channel, user_agent }
voice:transcript:<id>        LIST   each entry = JSON {ts, role, text, turn_id}
voice:tool:<id>              LIST   each entry = JSON {ts, name, args, triggered_by}
voice:sessions:index         ZSET   chronological browsing (score = ts, member = session_id)
```

All keys auto-expire after `ANALYTICS_TTL_DAYS` (default 30).

## Privacy posture

- Logs go to PRIVATE Upstash. Reviewable only via the Upstash console (or a future authenticated admin endpoint).
- **Anonymize the IP at write time** — strip the last octet of IPv4 (`192.168.1.42` → `192.168.1.0`). Keeps coarse geolocation for diagnostics, kills the tracking-tool optics.
- Clamp transcript text to a max length (e.g. 2000 chars) so a malformed huge payload can't blow your storage budget.
- Keep a `NEXT_VOICE_ANALYTICS_ENABLED` env var that defaults to ON in production and OFF in dev — so local sessions don't pollute the production store.
- All log functions **fail open**. Analytics never blocks a session, even if Upstash is down.

## Free-tier headroom

A 3-min session writes ~13 commands (1 session_start + 10 turns + 2 tool_calls + 1 session_end). Upstash free tier gives 10K commands/day → ~770 sessions/day ceiling. Storage at ~2KB/session × 30-day TTL × 100 sessions/day = ~6MB. Well under the 256MB free cap.

## Tag tool calls by trigger source

Critical for debugging: tag every tool log entry with `triggered_by: 'function_call'` or `'transcript_fallback'`. This tells you HOW often the model is lying about tool calls vs invoking them.

In production with preview-tier Gemini Live, expect the fallback to carry 60–90% of tool calls. If it's 100%, the model isn't invoking at all — re-check your tool definitions (schema casing, etc).

## The zremrangebyrank gotcha

"Trim the index to last 1000 sessions" is a one-liner that lies to you:

```ts
// WRONG — silently deletes the only member when N <= 1000
await redis.zremrangebyrank(INDEX_KEY, 0, -1001);
```

Redis clamps `-1001` when there are fewer than 1001 members. With N=1, the range collapses to `(0, 0)` and removes the only element. You'll see your sessions index stay empty even though session HASH and LIST keys are populating.

Gate the trim on actual overflow:

```ts
// CORRECT
const count = await redis.zcard(INDEX_KEY);
if (count > 1000) {
  await redis.zremrangebyrank(INDEX_KEY, 0, count - 1001);
}
```

## Wire-up summary

- Server-side: log `session_start` from `/api/invite-agent` after the agent starts; log `session_end` from `/api/end-agent` with duration + end reason.
- Client-side: log every final transcript turn AND every tool call (both function_call and transcript_fallback paths) via a fire-and-forget POST to `/api/voice-log` with `keepalive: true` so it survives navigation.
- Server `/api/voice-log` validates body shape, calls `logTurn` / `logToolCall`. Wrap in try/catch with `console.error` for production debugging (Vercel surfaces these in runtime logs).

## Diagnostic CLI workflow

Save this as a snippet for the moment a user reports failure:

```bash
# From project root, .env.local present
set -a && source .env.local && set +a

# What sessions exist?
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/keys/voice:session:*" | jq

# Most recent session id (when index works)
SID=$(curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/zrevrange/voice:sessions:index/0/0" | jq -r '.result[0]')

# Or pick from KEYS output if index is broken
SID="A46AN97JR75DN46KH84LN24HH26CA39P"

# Session metadata
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/hgetall/voice:session:$SID" | jq

# Full transcript
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/lrange/voice:transcript:$SID/0/-1" | \
  jq -r '.result[]' | jq -s

# All tool calls (look at triggered_by ratio)
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/lrange/voice:tool:$SID/0/-1" | \
  jq -r '.result[]' | jq -s
```

## Two production gotchas that only show up in the data

1. **User transcripts are typically empty.** Gemini Live multimodal handles user audio end-to-end and does NOT emit `user.transcription` events. You'll see only assistant-side turns. Visitor intent has to be inferred from agent replies and tool calls. Document this in the analytics module so future-you doesn't chase it.

2. **Pronoun-only commits.** Watch for turns like "I'm opening it for you now" with no project name. If the next user turn is a complaint, your fallback's anaphora resolution either isn't installed or isn't tracking context correctly. See `transcript-fallback.md`.

## Tool failure visibility

Reading the transcript will show cases where the agent narrates "I've opened the calendar" but the tool actually returned `ok: false` (env var unset, slug invalid, etc.). The model doesn't reliably surface tool failures in its speech.

Pair the analytics with a **client-side toast** — when `executeToolCall(...).ok === false`, render a 6-second toast surfacing the actual error message above the in-session UI. Visible truth beats narrated lies. This is the only fix that doesn't depend on the model behaving.
