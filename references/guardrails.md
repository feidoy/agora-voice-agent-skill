# Three-Layer Guardrails

Upstash Redis-backed cost + abuse controls. All three gates run on every session start.

## The pattern

| Gate | Prevents | Implementation |
|---|---|---|
| Single-session lock | Two users talking simultaneously | `SET NX EX 60` — atomic acquire with auto-expiry |
| Per-IP rate limit | One user spamming sessions | `@upstash/ratelimit` fixedWindow, 3/day default |
| Daily budget kill switch | Runaway cost from abuse or bug | `INCRBYFLOAT` with reservation + refund. **Clamp at 0.** |

## Critical bug to avoid: negative budget bypass

If `refundBudget()` is called with a small `actualSeconds`, the refund is large. If a malicious caller hits `/api/end-agent` repeatedly with `session_seconds: 1`, each call refunds ~$0.50. After 10 calls, `voice:spend:today` is `-$5`, which is below the threshold forever. **The kill switch is now disabled.**

Fix: always clamp `voice:spend:{date}` to `>= 0` after refund.

## Full implementation

```ts
// src/lib/voice/guardrails.ts
import { Redis } from '@upstash/redis';
import { Ratelimit } from '@upstash/ratelimit';

const isProduction = process.env.NODE_ENV === 'production';

function getRedis(): Redis | null {
  const url = process.env.UPSTASH_REDIS_REST_URL;
  const token = process.env.UPSTASH_REDIS_REST_TOKEN;
  if (!url || !token) {
    if (isProduction) {
      throw new Error('UPSTASH credentials required in production');
    }
    return null;
  }
  return new Redis({ url, token });
}

const MAX_COST_PER_SESSION_USD = 0.5;
const LOCK_TTL_SECONDS = 60;
const RATE_LIMIT_DEFAULT = Number(process.env.NEXT_RATELIMIT_PER_IP_PER_DAY ?? 3);
const DAILY_BUDGET_USD = Number(process.env.NEXT_DAILY_BUDGET_USD ?? 5);

export type GuardrailDecision =
  | { allowed: true }
  | {
      allowed: false;
      code: 'BUSY' | 'RATE_LIMITED' | 'BUDGET_EXCEEDED';
      message: string;
    };

export async function checkAndReserveSession(ip: string): Promise<GuardrailDecision> {
  const redis = getRedis();
  if (!redis) return { allowed: true };

  // Gate 1: single-session lock — atomic SET NX EX
  const lockKey = 'voice:active';
  const lockAcquired = await redis.set(lockKey, ip, {
    nx: true,
    ex: LOCK_TTL_SECONDS,
  });
  if (lockAcquired !== 'OK') {
    return {
      allowed: false,
      code: 'BUSY',
      message:
        "The line is busy — someone else is talking to the agent. Try again in a minute.",
    };
  }

  // Skip Gates 2 + 3 in dev (single-session lock still applies so you don't
  // accidentally double-launch when iterating)
  if (!isProduction) return { allowed: true };

  // Gate 2: per-IP rate limit
  try {
    const ratelimit = new Ratelimit({
      redis,
      limiter: Ratelimit.fixedWindow(RATE_LIMIT_DEFAULT, '1d'),
      prefix: 'voice:ratelimit',
    });
    const { success } = await ratelimit.limit(ip);
    if (!success) {
      await redis.del(lockKey); // release lock
      return {
        allowed: false,
        code: 'RATE_LIMITED',
        message: `You've already used your ${RATE_LIMIT_DEFAULT} conversations for today. Come back tomorrow.`,
      };
    }
  } catch (error) {
    await redis.del(lockKey);
    throw error;
  }

  // Gate 3: daily budget — wrapped in try/catch so any Upstash hiccup
  // mid-incrbyfloat releases the lock instead of stranding it for the TTL
  try {
    const today = new Date().toISOString().slice(0, 10);
    const spendKey = `voice:spend:${today}`;
    const currentSpend = Number((await redis.get(spendKey)) ?? 0);

    if (currentSpend >= DAILY_BUDGET_USD) {
      await redis.del(lockKey);
      return {
        allowed: false,
        code: 'BUDGET_EXCEEDED',
        message: "Today's voice agent budget has been used up. Try again tomorrow.",
      };
    }

    const newSpend = await redis.incrbyfloat(spendKey, MAX_COST_PER_SESSION_USD);
    await redis.expire(spendKey, 86400); // 24h TTL

    if (newSpend > DAILY_BUDGET_USD * 1.05) {
      await redis.del(lockKey);
      await redis.incrbyfloat(spendKey, -MAX_COST_PER_SESSION_USD);
      return {
        allowed: false,
        code: 'BUDGET_EXCEEDED',
        message: "Today's voice agent budget has been used up. Try again tomorrow.",
      };
    }

    return { allowed: true };
  } catch (error) {
    await redis.del(lockKey).catch(() => undefined);
    throw error;
  }
}

/** Release the single-session lock. Call from /api/end-agent + on errors. */
export async function releaseSessionLock(): Promise<void> {
  const redis = getRedis();
  if (!redis) return;
  await redis.del('voice:active');
}

/** Refund unused budget. Clamp at 0 to prevent negative-bypass attack. */
export async function refundBudget(actualSeconds: number): Promise<void> {
  const redis = getRedis();
  if (!redis) return;

  const costPerSec = Number(process.env.NEXT_VOICE_COST_PER_SECOND ?? 0.0025);

  // Clamp inputs — malicious caller might pass 0 or negative for max refund
  if (!Number.isFinite(actualSeconds) || actualSeconds <= 0) return;
  const cappedSeconds = Math.min(actualSeconds, 600);
  const actualCost = cappedSeconds * costPerSec;
  const refund = MAX_COST_PER_SESSION_USD - actualCost;
  if (refund <= 0) return;

  const today = new Date().toISOString().slice(0, 10);
  const key = `voice:spend:${today}`;
  const after = await redis.incrbyfloat(key, -refund);

  // CRITICAL: clamp to >= 0. Without this, repeated end-agent calls drive the
  // running spend negative and disable the budget gate entirely.
  if (Number(after) < 0) {
    await redis.set(key, 0, { ex: 86400 });
  }
}
```

## Tuning

- `MAX_COST_PER_SESSION_USD = 0.5` — assumes ~3-min sessions at $0.10/min. Adjust if your model is more expensive.
- `LOCK_TTL_SECONDS = 60` — short enough that a crashed browser doesn't stick the lock for long; long enough to cover a normal session start. The end-agent route releases it cleanly anyway.
- `RATE_LIMIT_DEFAULT = 3/day` — appropriate for a personal portfolio. Higher for a SaaS demo.
- `DAILY_BUDGET_USD = 5` — kill switch ceiling.
- `NEXT_VOICE_COST_PER_SECOND = 0.0025` — refund cost model. Tune once you have real usage data.
