---
name: agora-voice-agent
description: "Build a voice-chat AI agent for a Next.js web site using Agora Conversational AI + Gemini Live (multimodal voice in/out). Covers architecture decision, Agora SDK setup, Gemini Live config, tool calling with fallback, three-layer cost guardrails, and the wire-format gotchas that cost real time the first time. Use when the user wants to add 'talk to AI' / 'voice chat' / 'voice agent' to a site."
---

# Agora Voice Agent Build

A runbook for adding a voice chat agent to a web site. Built and verified end-to-end on a Next.js 16 portfolio. The goal: a floating mic button → user clicks → opens settings → picks voice → starts call → talks to AI → AI can navigate the site via tool calls.

This skill encodes hard-won learnings from a 6-hour first build. Most of these gotchas are not in any single doc. **Read this top to bottom before writing code** — the bugs you'll hit otherwise cost ~30 minutes each to diagnose.

---

## When to invoke

- User wants to add a voice chat / "talk to AI" widget to their site
- User says: "build a voice agent", "voice chat with my portfolio", "AI assistant that talks"
- User points at a YouTube demo of someone building one
- User wants visitors to converse with their site without typing

**NOT this skill:**
- Text-only chat → use Vercel AI SDK / chat-sdk skill
- Slack / Discord bots → use chat-sdk skill
- Speech-to-text only (transcription) → don't use Convo AI; use Deepgram directly

---

## Step 0 — Read the user's references

If the user hands you a YouTube link, blog post, or doc page, **fetch and read it before recommending an architecture.** Voice agents have multiple valid architectures and the user usually has one in mind from a video they watched.

For YouTube transcripts when scraping services are blocked:
```bash
yt-dlp --write-auto-sub --sub-lang en --skip-download --output "yt.%(ext)s" "<url>"
# then strip VTT timestamps and read
```

---

## Step 1 — Architecture decision (Path A vs Path B)

Two paths. **Default to Path B** unless cost/voice-control matters more than naturalness.

### Path A: Pipeline (older default)
```
Mic → STT (Deepgram) → text → LLM (text) → text → TTS (ElevenLabs) → speaker
```
- Three vendors, three configs, three failure modes
- LLM never hears the user's voice (no tone, no hesitation)
- Cheaper per minute, more vendor flexibility
- Best for: high volume, custom voice cloning, cost-sensitive

### Path B: Multimodal Live (newer default — pick this)
```
Mic → Gemini Live (audio in → audio out + tool calls) → speaker
```
- One vendor, one SDK call
- Native bidirectional audio: model hears tone, speaks with intonation
- ~200–400ms latency vs 600–900ms for pipeline
- Best for: portfolio demos, conversational feel, low maintenance

**Both paths can use Agora as the WebRTC transport.** Agora handles mic capture, channel join, audio routing — just plumbing. The difference is what you put in the middle.

---

## Step 2 — Verify the model identifier

For Gemini Live, the exact ID lives at `https://ai.google.dev/gemini-api/docs/models/{model-id}`, NOT in marketing blog posts.

Current options (verify before using):
- `gemini-3.1-flash-live-preview` — highest quality, but **deprecated proactive audio** (agent waits for user speech first; doesn't auto-greet)
- `gemini-2.5-flash-native-audio-preview-12-2025` — supports proactive greeting; older but more reliable for "AI speaks first" UX

**Important:** the SDK default `gemini-live-2.5-flash` is half-duplex (text out only). It will join the channel, mute its audio, and emit a 500 from the `mllm` module. Do not use it.

---

## Step 3 — Install dependencies

```bash
npm install --legacy-peer-deps \
  agora-agent-server-sdk@^1.3.2 \
  agora-rtc-react@^2.5.1 \
  agora-rtc-sdk-ng@^4.24.3 \
  agora-rtm@^2.2.3 \
  agora-token@^2.0.5 \
  @upstash/redis@^1.37.0 \
  @upstash/ratelimit@^2.0.8
```

`--legacy-peer-deps` is required — `agora-rtm@2.2.x` pins to an older `agora-rtc-sdk-ng` version than the latest. Both Agora packages work fine together at runtime; npm just complains.

---

## Step 4 — Environment variables

See `references/env-vars.md` for full template.

Minimum:
- `NEXT_PUBLIC_AGORA_APP_ID` (public — Agora App IDs are not secrets)
- `NEXT_AGORA_APP_CERTIFICATE` (server-only, never expose to client)
- `NEXT_GEMINI_API_KEY` (from aistudio.google.com/apikey)
- `NEXT_GEMINI_MODEL` (verified ID from Step 2)
- `UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN` (free tier from console.upstash.com)

In the user's Agora project, the **Conversational AI Engine** feature must be explicitly enabled (button in Agora console under the project's Configure page).

---

## Step 5 — Critical config decisions

These are the choices that cost time the first time around. Make them right from the start.

| Decision | Pick this | Why |
|---|---|---|
| Schema types in tool definitions | `'object'`, `'string'` (lowercase) | Gemini Live's setup validator rejects proto-style uppercase with `code 500` from the `mllm` module |
| `data_channel` parameter | `'datastream'` (NOT `'rtm'`) | RTM requires the Presence service to be enabled on the Agora project; datastream uses the existing RTC connection. Avoids `error -13001 Presence service not connected`. |
| Token type | `RtcTokenBuilder.buildTokenWithRtm(...)` | Combined RTC+RTM token from one call. Avoids `-10015 NO_RTM_PRIVILEGE` if the client tries to subscribe to RTM later. |
| Stream-message listener placement | BEFORE `client.join()` | Late binding misses early messages including the first tool call. |
| Greeting message field | On the `GeminiLive` vendor, NOT the `Agent` | Double-setting causes engine config conflicts (silent agent on join). |

---

## Step 6 — Build the routes

Four routes, in order of dependency:

1. **`/api/voice-token`** (POST) — issues an RTC+RTM token. Validates channel + uid format. See `references/route-templates.md`.
2. **`/api/invite-agent`** (POST) — starts the Convo AI session with Gemini Live + tool definitions. Applies guardrails before allocating resources. See `references/route-templates.md` and `references/tool-definitions.md`.
3. **`/api/end-agent`** (POST) — releases the single-session lock + refunds unused budget. Critical for mid-session voice-switching UX. Without this, the user hits BUSY for 60s after every end.
4. **`/api/voice-reset`** (GET) — dev-only escape hatch. Gated behind `NODE_ENV === 'production'` check. Clears the Upstash voice:active lock + sweeps voice:ratelimit:* keys.

---

## Step 7 — Build the guardrail layer

Three independent gates, all backed by Upstash Redis. See `references/guardrails.md` for full code.

| Gate | What it prevents | Implementation |
|---|---|---|
| Single-session lock | Two users talking simultaneously | `SET NX EX 60` on `voice:active`. TTL prevents stuck locks if browser crashes. |
| Per-IP rate limit | One user spamming sessions | `@upstash/ratelimit` fixedWindow, default 3/day |
| Daily budget kill switch | Runaway costs from abuse | `INCRBYFLOAT` on `voice:spend:{date}`. Reserve worst-case at session start; refund on clean end. **Clamp refund at 0** — otherwise a malicious caller can drive spend negative and disable the gate entirely. |

In dev mode, skip rate-limit + budget gates (still keep the single-session lock so accidental double-clicks don't double-launch).

---

## Step 8 — Build the client component

One React component, `'use client'`, owns the full state machine. See `references/client-component.md`.

State: `idle | connecting | active | ending | error`

Lifecycle on click-to-start:
1. POST `/api/voice-token` → get RTC token
2. Lazy-import `agora-rtc-sdk-ng` (~800KB, keep out of main bundle)
3. Create RTC client, register ALL listeners (`user-joined`, `user-published`, `stream-message`)
4. `client.join(...)`
5. Get mic, `client.publish(mic)`
6. POST `/api/invite-agent` → server brings agent into channel
7. Subscribe to agent's audio via `user-published` event
8. Listen for tool calls via `stream-message` event

Lifecycle on end:
1. Stop session timer
2. Close mic track
3. `client.leave()`
4. POST `/api/end-agent` to release server-side lock immediately

---

## Step 9 — Wire format and tool calls

This is the gotcha that ate the most time. Agora sends agent events as data-stream messages with this format:

```
{messageId}|{seq}|{total}|{base64-encoded-json-body}
```

- Most messages fit in one chunk (`seq=1, total=1`)
- Multi-part messages need reassembly by `messageId`
- Body is base64; decode → JSON.parse
- Look for tool calls in 4+ shapes (`function_call`, `tool_calls[]`, `toolCall.functionCalls[]`, etc.)
- Most messages are `assistant.transcription` chunks (the firehose of agent's spoken text)

See `references/decoding.md` for the parser.

**Critical fallback pattern:** Gemini Live preview models often **narrate** function calls without invoking them ("I'll open X for you" — but no tool fires). Always pair real tool-call detection with a transcript-pattern matcher that catches navigation intent in the transcribed text and triggers the action client-side as a safety net. See `references/transcript-fallback.md`.

---

## Step 10 — Voice picker

Gemini Live has 30 named voices. Each has a one-word descriptor (e.g. Aoede=breezy, Kore=firm, Charon=informative). Use an allowlist for server-side validation (don't accept arbitrary client input). See `references/voice-catalog.md`.

UX pattern that worked well:
- Click the floating chat icon → opens settings panel with voice picker
- User picks a voice (highlights the card; doesn't start call yet)
- Bottom CTA "Initiate Call" starts the session with that voice
- Voice persists in `localStorage` for next time
- Mid-call: voice chip in the active panel opens the picker; "Switch & Restart" ends and re-starts the call with the new voice

---

## Step 11 — Verify before push

Run code review + security review in parallel. The pre-push security review on this build caught a HIGH severity bug (negative-budget bypass) that would have disabled the daily kill switch in production. Worth the 90 seconds.

```bash
# Run /qa for browser smoke test
# Then in parallel:
#   - code-reviewer agent
#   - security-reviewer agent
# Pass them the list of new files + concrete questions about correctness, env exposure, regex tightness, and any known-fragile heuristics.
```

Also build a **drift-resistant QA harness** before shipping. The transcript fallback is regex-heavy and you will keep evolving it. A standalone Node script that **extracts the regex literals and slug mappings from the source file** (instead of duplicating them) catches cases where you tighten one regex and forget to update tests. See `references/transcript-fallback.md` for the harness pattern. Wire it into a `preship` script alongside the build.

---

## Step 12 — Post-launch diagnostics (read-the-data > guess-the-bug)

After ship, you WILL get reports of "it didn't work." The first 3 reports are nearly always the same root cause but feel like 3 different bugs. Resist the urge to add more regex patterns until you've read at least one real session.

Build private analytics from day one — see `references/analytics.md`. Storage shape:

```
voice:session:<id>           HASH   metadata (ip last-octet stripped, voice, duration, end_reason)
voice:transcript:<id>        LIST   JSON entries {ts, role, text, turn_id}
voice:tool:<id>              LIST   JSON entries {ts, name, args, triggered_by}
voice:sessions:index         ZSET   chronological browse, score=ts member=session_id
```

**Diagnostic workflow when a user reports a failure:**

```bash
# From the project root, with .env.local loaded
SID=$(curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/zrevrange/voice:sessions:index/0/0" | jq -r '.result[0]')

# Read the session's transcript and tool calls
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/lrange/voice:transcript:$SID/0/-1"
curl -s -X POST -H "Authorization: Bearer $UPSTASH_REDIS_REST_TOKEN" \
  "$UPSTASH_REDIS_REST_URL/lrange/voice:tool:$SID/0/-1"
```

Tagging tool calls with `triggered_by: "function_call"` vs `"transcript_fallback"` is critical — it tells you how often the model is lying about tool calls vs invoking them. If 90% of your tool calls came via the fallback, that's a signal the model isn't reliable enough and the prompt may need rework.

**Two production gotchas you'll only see in the data, not in QA:**

1. **User transcripts are typically empty.** Gemini Live multimodal handles user audio end-to-end and does NOT emit a separate `user.transcription` event. Your `user-side regex` (e.g. for bare imperatives like "open the product") will never fire on the production wire even if QA passes. Don't chase it; document the gap and rely on the assistant side. The wire might emit user transcriptions on a future model — keep the code path harmless.
2. **`zremrangebyrank(KEY, 0, -1001)` deletes the only member when N≤1000.** Redis clamps the negative index, collapsing the range to (0, 0). This silently kills your sessions index. Gate the trim on `ZCARD > 1000`.

---

---

## Common pitfalls — quick reference

These are the actual bugs we hit, ranked by time-to-diagnose.

| Symptom | Cause | Fix |
|---|---|---|
| Agent joins channel but is muted on audio | `outputModalities: ['audio']` (lowercase) — Gemini Live wants `['AUDIO']`, OR you're on the half-duplex `gemini-live-2.5-flash` model | Drop the modalities override; switch to `gemini-3.1-flash-live-preview` |
| `code: 500` from `module: mllm`, `message: receive loop error {e}` | Schema types are uppercase (`OBJECT`/`STRING`) | Switch to lowercase per JSON Schema |
| `error -13001 Presence service not connected` on RTM subscribe | Using `data_channel: 'rtm'` without enabling Presence in Agora console | Switch to `data_channel: 'datastream'` |
| `error -10015 NO_RTM_PRIVILEGE` | Token doesn't have RTM privilege | Use `RtcTokenBuilder.buildTokenWithRtm` instead of `buildTokenWithUid` |
| BUSY error when voice-switching mid-call | Single-session lock TTL hasn't expired | Add `/api/end-agent` route that releases the lock; client awaits it before calling `/api/invite-agent` again |
| Tool call console.log shows `assistant.transcription` only, never `function_call` | Gemini Live preview is narrating without invoking | Add transcript-pattern fallback. Match navigation phrases against a project allowlist; trigger client-side. |
| Agent narrates "I'm opening it for you now" but page doesn't change, even with the fallback installed | **Anaphora.** The agent uses a pronoun ("it", "that one") referring to a project mentioned earlier. The fallback can't find a slug in the current text. | Track last-mentioned project across turns in `lastProjectSlugRef`. Pass it to the matcher as `contextSlug` — if navigate-intent matches but no slug is found in the current text, fall back to context. See `references/transcript-fallback.md`. |
| Agent narrates "Opened the calendar now" / "Just sent" / "I've opened" — past tense — and zero tools fire | **Past-tense narration.** The model reasons about the action being completed and narrates that way. NAVIGATE_INTENT_RE captured present-continuous ("opening") and future ("I'll open") but missed past tense entirely. | Expand the verb set across all tense forms in NAVIGATE_INTENT_RE: `open(?:ing\|ed)`, `(?:taking\|took) you`, `(?:showing\|showed) you`, `book(?:ing\|ed)`, plus `I've opened` / `I have opened` / `Just opened` auxiliary variants. Add a prompt-rule asking the model to use present-continuous as belt-and-suspenders. |
| Agent says "I'm opening his calendar... Want me to open his LinkedIn instead?" — and LinkedIn fires instead of calendar | **Search window includes the alternative offer.** The matcher was checking intents across the WHOLE text, so LinkedIn-mentioned-in-the-offer matched even though calendar was the actual commitment. | Slice the text at the first offer phrase (`commitText = lower.slice(0, offerMatch.index)`) and only search the commitment portion for intent dispatch. The pre-offer text is the commitment; what comes after is alternatives. |
| Agent's natural "let me know if you want another one" tagline kills navigation that already committed | The question/offer regex matches the tagline and suppresses fire | Make the suppressor **order-aware**. Only suppress if the offer phrase appears BEFORE the commitment phrase. Lets "I'm opening it. Let me know if you want another." fire while still suppressing "If you want, I'll open it." |
| Production user-side regex (`USER_NAVIGATE_INTENT_RE`) never fires despite passing QA | Gemini Live multimodal doesn't emit `user.transcription` events at all | Don't try to fix it. Document the gap in the analytics module and the QA doc. The user-side regex stays harmless (doesn't break anything if the event never arrives) but plan around its absence: the agent side carries 100% of the navigation signal. |
| `voice:sessions:index` ZSET stays empty even though session HASH and transcript LIST are populating correctly | `zremrangebyrank(key, 0, -1001)` deletes the only member when N≤1000. Redis clamps `-1001` to `0`, collapsing the range to `(0, 0)`. | Gate the trim on `ZCARD > 1000` — only trim when there's actual overflow. |
| Agent narrates "I've opened the calendar" confidently, but the booking tool returned `ok:false` (env var unset) | Tool result isn't being fed back to the model, OR is fed back but the model ignores it | Two fixes layered: (1) prompt rule "if open_booking returns ok:false, share `<your-contact-email>` instead" — best-effort, model may ignore; (2) **client-side toast** that surfaces tool-failure messages visibly above the in-session UI for 6s. The toast is the truth-telling layer that doesn't depend on the model. |
| Agent tries to collect name + email by voice ("What's your email?... Spell that?") | Default Cal.com / scheduling guidance assumes voice-driven booking | Hard rule in the prompt: "NEVER collect name/email by voice — the calendar form does that." Pair with a hybrid `open_booking` tool that just opens the link; let the calendar's own form handle data entry. Voice for the warm-up, click for the high-stakes typing. |
| Agent probes visitor about their employment ("Are you a recruiter?") | Default LLM-friendly persona templates encourage adaptive segmentation | Hard rule in the prompt: "NEVER probe employment status, affiliation, or whether they're a recruiter. Adapt to what they volunteer." Visitors find the segmentation prompts intrusive, and agents that classify by employment status often end up in awkward conversational corners. |
| Hydration mismatch error after JSX shape change | Stale server module in `.next/` cache | `lsof -ti:3000 \| xargs kill && rm -rf .next/ && npm run dev` + hard refresh |
| `properties: mllm.url not found` (400) | SDK type marks `url` as optional but Agora's engine requires it | Pass `url: 'wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent'` to GeminiLive constructor |
| Cookie banner blocks Playwright screenshot capture | Default capture happens before banner dismissal | `page.getByRole('button', { name: /essential only/i }).click()` then re-screenshot |

---

## References

- `references/env-vars.md` — full env var template with sourcing instructions
- `references/route-templates.md` — drop-in Next.js route handler skeletons
- `references/tool-definitions.md` — Gemini Live tool function declaration format with examples
- `references/guardrails.md` — Upstash-backed three-layer guardrail code
- `references/decoding.md` — Agora wire format parser with all known shapes
- `references/transcript-fallback.md` — pattern matcher for unreliable tool calling, **including anaphora handling and order-aware suppression**
- `references/voice-catalog.md` — all 30 Gemini voices with descriptors
- `references/client-component.md` — React state machine + RTC lifecycle
- `references/analytics.md` — private session logging to Upstash + diagnostic CLI workflow + the zremrangebyrank gotcha
- `references/prompt-rules.md` — system-prompt rules learned the hard way (don't probe employment, don't narrate failures, don't collect name/email by voice, the docent-not-teleporter UX, the hybrid-booking pattern)
- `references/kb-grounding.md` — three KB grounding strategies (full inject / topic-routed / RAG), how to scope tools per use case, and the support-specific patterns (confidence + escalation, auth-gated tools, per-tenant context, ticket creation as universal escalation path)

---

## Cost estimates

Per minute, all-in:
- Agora Convo AI: ~$0.99 / 1k minutes (~$0.001/min)
- Gemini Live: ~$0.10–0.20/min depending on input/output ratio
- Upstash REST: negligible at portfolio traffic levels

Realistic budget for a personal portfolio: $5/day with a 3-min session cap and 3 sessions/IP/day = max ~$2.70/IP/day worst case.
