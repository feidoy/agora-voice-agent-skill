# Environment Variables

Copy this template into `.env.local.example` and adapt for the project.

```bash
# =============================================================================
# Voice Agent — environment variables
# =============================================================================
# Copy this file to `.env.local` (same directory) and fill in real values.
# `.env.local` is gitignored. NEVER commit it.
#
# Architecture: Agora Conversational AI Engine handles WebRTC transport.
# Gemini Live (multimodal) handles audio-in, reasoning, and audio-out natively.
# =============================================================================


# -----------------------------------------------------------------------------
# Agora — required
# -----------------------------------------------------------------------------
# 1. Sign up / sign in at https://console.agora.io
# 2. Create a new project. Copy the App ID and the Primary Certificate.
# 3. CRITICAL: open the project's Configure page and click "Engage Conversational AI"
#    to enable the Convo AI Engine for this App ID.
#
# App IDs are NOT secrets (Agora's docs explicitly state this). They appear in
# every client-side SDK call. The certificate IS a secret — server-only.
# -----------------------------------------------------------------------------

NEXT_PUBLIC_AGORA_APP_ID=
NEXT_AGORA_APP_CERTIFICATE=

# Optional — UID the AI agent uses inside the RTC channel. Default 123456.
NEXT_PUBLIC_AGENT_UID=123456


# -----------------------------------------------------------------------------
# Gemini — required (handles STT + LLM + TTS in one)
# -----------------------------------------------------------------------------
# Get an API key at https://aistudio.google.com/apikey
#
# Verify the model identifier in Google's docs:
# https://ai.google.dev/gemini-api/docs/models/{model-id}
# Marketing blog posts often omit the API name. Always cross-reference.
# -----------------------------------------------------------------------------

NEXT_GEMINI_API_KEY=
# Latest stable Live models (verify before using):
# - gemini-3.1-flash-live-preview            (best quality; deprecated proactive audio)
# - gemini-2.5-flash-native-audio-preview-12-2025  (older; supports auto-greet)
# DO NOT USE gemini-live-2.5-flash — half-duplex; produces silent agent.
NEXT_GEMINI_MODEL=gemini-3.1-flash-live-preview

# Voice selection. 30 options — see references/voice-catalog.md.
NEXT_GEMINI_VOICE=Kore

# Google's Gemini Live WebSocket endpoint. REQUIRED — Agora's engine validates
# it even though the SDK type marks it optional. Override only if Google
# moves the URL or you want a regional endpoint.
NEXT_GEMINI_URL=wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent


# -----------------------------------------------------------------------------
# Greeting (optional)
# -----------------------------------------------------------------------------
# First thing the agent says when a user joins. Set the persona's tone here.
# Note: 3.x Live models DON'T speak this proactively — user must talk first.
NEXT_AGENT_GREETING="Hey, welcome — what brings you here today?"


# -----------------------------------------------------------------------------
# Upstash Redis — required for production (guardrails)
# -----------------------------------------------------------------------------
# Create a free database at https://console.upstash.com — Global or Regional.
# Copy the REST URL + REST Token (NOT the regular Redis credentials).
#
# Used for:
#   voice:active            — single-session lock (60s TTL)
#   voice:ratelimit:*       — per-IP rate limit counters
#   voice:spend:{date}      — daily budget tracker
# -----------------------------------------------------------------------------

UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

# Hard cap on a single conversation in seconds (auto-disconnect).
NEXT_MAX_SESSION_SECONDS=180

# Max conversations per IP per day (production only — disabled in dev).
NEXT_RATELIMIT_PER_IP_PER_DAY=3

# Daily $USD budget kill switch. Cumulative spend > this = reject new calls.
NEXT_DAILY_BUDGET_USD=5

# Cost model: $/sec used for the budget refund calculation.
# Default 0.0025 = ~$0.15/min, conservative for Gemini Live + Agora.
# NEXT_VOICE_COST_PER_SECOND=0.0025
```

## Vercel deployment

Add the same vars in Vercel → Project Settings → Environment Variables. Keep `Production`, `Preview`, `Development` in sync.

`NODE_ENV` is set automatically by Vercel — don't set it manually. Preview deployments run with `NODE_ENV=production`, which means the `voice-reset` dev-only gate works correctly there.

## Local gitignore

Verify `.gitignore` catches both files:
```
.env*
```

This intentionally also gitignores `.env.local.example`. If you want the example committed for documentation, add an exception:
```
!.env.local.example
```
