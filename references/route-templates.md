# Route Handler Templates

Drop-in skeletons for the four routes. Adapt to your project's import paths.

---

## `/api/voice-token` — issue combined RTC+RTM token

```ts
// src/app/api/voice-token/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { RtcRole, RtcTokenBuilder } from 'agora-token';

const TOKEN_TTL_SECONDS = 60 * 60;

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing required environment variable: ${name}`);
  return value;
}

export async function POST(request: NextRequest) {
  try {
    const body = (await request.json()) as { channel_name?: string; uid?: string };
    const { channel_name, uid } = body;

    if (!channel_name || !uid) {
      return NextResponse.json(
        { error: 'channel_name and uid are required' },
        { status: 400 },
      );
    }

    const appId = requireEnv('NEXT_PUBLIC_AGORA_APP_ID');
    const appCertificate = requireEnv('NEXT_AGORA_APP_CERTIFICATE');

    const numericUid = Number(uid);
    if (!Number.isFinite(numericUid) || numericUid < 0) {
      return NextResponse.json(
        { error: 'uid must be a non-negative numeric string' },
        { status: 400 },
      );
    }

    const expiresAt = Math.floor(Date.now() / 1000) + TOKEN_TTL_SECONDS;

    // CRITICAL: buildTokenWithRtm — combined RTC+RTM, not buildTokenWithUid
    const token = RtcTokenBuilder.buildTokenWithRtm(
      appId,
      appCertificate,
      channel_name,
      String(numericUid),
      RtcRole.PUBLISHER,
      expiresAt,
      expiresAt,
    );

    return NextResponse.json({
      app_id: appId,
      channel_name,
      uid,
      token,
      expires_at: expiresAt,
    });
  } catch (error) {
    console.error('Error issuing voice token:', error);
    return NextResponse.json(
      {
        error:
          error instanceof Error ? error.message : 'Failed to issue token',
      },
      { status: 500 },
    );
  }
}
```

---

## `/api/invite-agent` — start Convo AI session with Gemini Live

```ts
// src/app/api/invite-agent/route.ts
import { NextRequest, NextResponse } from 'next/server';
import {
  AgoraClient,
  Agent,
  Area,
  ExpiresIn,
  GeminiLive,
} from 'agora-agent-server-sdk';
import { checkAndReserveSession, releaseSessionLock } from '@/lib/voice/guardrails';
import { buildSystemPrompt } from '@/lib/voice/promptBuilder';

const MAX_SESSION_SECONDS = Number(process.env.NEXT_MAX_SESSION_SECONDS ?? 180);
const GREETING = process.env.NEXT_AGENT_GREETING ?? 'Hey, what can I help you with?';
const AGENT_UID = process.env.NEXT_PUBLIC_AGENT_UID ?? '123456';

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing required environment variable: ${name}`);
  return value;
}

function getClientIp(request: NextRequest): string {
  return (
    request.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ??
    request.headers.get('x-real-ip') ??
    'unknown'
  );
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { requester_id, channel_name, voice: requestedVoice } = body;

    if (!channel_name || !requester_id) {
      return NextResponse.json(
        { error: 'channel_name and requester_id are required' },
        { status: 400 },
      );
    }

    // Guardrails first — fail fast if rate-limited or budget exceeded
    const ip = getClientIp(request);
    const decision = await checkAndReserveSession(ip);
    if (!decision.allowed) {
      return NextResponse.json(
        { error: decision.message, code: decision.code },
        { status: 429 },
      );
    }

    const appId = requireEnv('NEXT_PUBLIC_AGORA_APP_ID');
    const appCertificate = requireEnv('NEXT_AGORA_APP_CERTIFICATE');
    const geminiApiKey = requireEnv('NEXT_GEMINI_API_KEY');
    const geminiModel = process.env.NEXT_GEMINI_MODEL ?? 'gemini-3.1-flash-live-preview';
    const geminiVoice = requestedVoice ?? process.env.NEXT_GEMINI_VOICE ?? 'Kore';
    const geminiUrl =
      process.env.NEXT_GEMINI_URL ??
      'wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent';

    const systemPrompt = buildSystemPrompt();

    const client = new AgoraClient({ area: Area.US, appId, appCertificate });

    // CRITICAL: instructions/greeting on GeminiLive vendor, NOT on Agent.
    // Setting both causes the agent to join muted.
    const agent = new Agent({
      name: `voice-agent-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
      failureMessage: 'Hold on, give me a moment.',
      maxHistory: 50,
      advancedFeatures: { enable_rtm: false, enable_tools: true },
      // datastream NOT rtm — RTM requires Presence service
      parameters: { data_channel: 'datastream', enable_error_message: true },
    }).withMllm(
      new GeminiLive({
        apiKey: geminiApiKey,
        model: geminiModel,
        url: geminiUrl, // REQUIRED even though SDK type says optional
        instructions: systemPrompt,
        voice: geminiVoice,
        greetingMessage: GREETING,
        failureMessage: 'Hold on, give me a moment.',
        maxHistory: 50,
        // No inputModalities/outputModalities — let SDK pick defaults.
        // Overriding has produced silent agents in testing.
        additionalParams: {
          // Tool definitions go here. See references/tool-definitions.md
          tools: [
            // ... your function declarations
          ],
        },
      }),
    );

    const session = agent.createSession(client, {
      channel: channel_name,
      agentUid: AGENT_UID,
      remoteUids: [String(requester_id)],
      idleTimeout: 30,
      expiresIn: ExpiresIn.seconds(MAX_SESSION_SECONDS),
      debug: false,
    });

    const agentId = await session.start();

    return NextResponse.json({
      agent_id: agentId,
      create_ts: Math.floor(Date.now() / 1000),
      state: 'RUNNING',
    });
  } catch (error) {
    // Release lock on any failure
    await releaseSessionLock().catch(() => undefined);
    console.error('Error starting voice session:', error);
    return NextResponse.json(
      {
        error:
          error instanceof Error ? error.message : 'Failed to start conversation',
      },
      { status: 500 },
    );
  }
}
```

---

## `/api/end-agent` — release lock, refund budget

**This endpoint is critical.** Without it the lock TTL is 60s and users hit BUSY when voice-switching mid-call.

```ts
// src/app/api/end-agent/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { refundBudget, releaseSessionLock } from '@/lib/voice/guardrails';

export async function POST(request: NextRequest) {
  let sessionSeconds = 0;
  try {
    const body = (await request.json()) as { session_seconds?: number };
    if (typeof body?.session_seconds === 'number' && body.session_seconds > 0) {
      sessionSeconds = body.session_seconds;
    }
  } catch {
    /* empty body OK */
  }

  await releaseSessionLock().catch(() => undefined);
  if (sessionSeconds > 0) {
    await refundBudget(sessionSeconds).catch(() => undefined);
  }

  return NextResponse.json({ ok: true });
}
```

---

## `/api/voice-reset` — dev-only escape hatch

Useful when the lock gets stuck during development. **Always keep behind the `NODE_ENV` gate** so it's a 404 in production.

```ts
// src/app/api/voice-reset/route.ts
import { NextResponse } from 'next/server';
import { Redis } from '@upstash/redis';
import { releaseSessionLock } from '@/lib/voice/guardrails';

export async function GET() {
  if (process.env.NODE_ENV === 'production') {
    return NextResponse.json(
      { error: 'Not available in production' },
      { status: 404 },
    );
  }

  await releaseSessionLock();

  const url = process.env.UPSTASH_REDIS_REST_URL;
  const token = process.env.UPSTASH_REDIS_REST_TOKEN;
  let counters_cleared = 0;

  if (url && token) {
    const redis = new Redis({ url, token });
    const today = new Date().toISOString().slice(0, 10);

    const keys = [`voice:spend:${today}`];
    for (const k of keys) {
      counters_cleared += (await redis.del(k)) ?? 0;
    }

    let cursor: string = '0';
    do {
      const [next, batch] = (await redis.scan(cursor, {
        match: 'voice:ratelimit:*',
        count: 100,
      })) as [string, string[]];
      if (batch.length > 0) {
        counters_cleared += (await redis.del(...batch)) ?? 0;
      }
      cursor = next;
    } while (cursor !== '0');
  }

  return NextResponse.json({
    ok: true,
    message: 'Voice agent lock + rate limits cleared.',
    counters_cleared,
  });
}
```

Use it from terminal during dev:
```bash
curl -sS http://localhost:3000/api/voice-reset
```
