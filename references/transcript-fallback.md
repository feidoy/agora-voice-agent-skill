# Transcript-Pattern Fallback

Gemini Live preview models often **narrate** function calls without invoking them. The agent says "I'll open the X for you" but no `function_call` is emitted, the user sees nothing happen, and they think your app is broken.

This pattern catches it: pattern-match the transcript text and trigger the action client-side as a safety net.

## What works (verified in production)

The matcher must layer SIX behaviors. Each one was added in response to a real failure mode seen in Upstash session data:

1. **Strict commitment phrasing across all tense forms** — fire on present-continuous (`opening`), future (`I'll open`), past (`opened` / `just sent` / `I've opened`), AND auxiliary variants. Bare "open" doesn't count (it appears in offers and questions). The single most common failure mode I missed initially: PAST TENSE. The model frequently narrates "Opened the calendar now" as if the action already happened — without past-tense in the regex, the dispatch silently no-ops.
2. **Order-aware question/offer suppression** — a question/offer phrase only suppresses if it appears BEFORE the commitment phrase. Lets "I'm opening it. Let me know if you want another one." fire while still suppressing "If you want, I'll open it."
3. **Commitment-text scoping for intent search** — slice the lowered text at the first offer phrase and search ALL intents (booking, LinkedIn, project, writeup, live, home) in the commitment prefix only. The pre-offer text is the commitment; what comes after is alternatives. Without this scoping, "I'm opening the calendar... Want me to open the LinkedIn instead?" fires LinkedIn instead of booking — the WRONG action.
4. **Writeup-vs-live disambiguation** — if the agent narrates a domain (e.g. `acme.com`), assume live unless the user explicitly said "project page" / "writeup" — in which case writeup wins even with a domain mention.
5. **Anaphora resolution** — when commitment matches but no project name appears in the current text ("I'm opening it for you now"), fall back to the most-recently-mentioned project from prior turns.
6. **Booking-before-LinkedIn priority** — within the commitment portion, check `BOOKING_INTENT_RE` before `LINKEDIN_INTENT_RE`. Calendar narrations frequently end with "want me to open the LinkedIn instead?" alternatives; even with commitment-text scoping, ordering booking first prevents edge-case mis-routes.

Skip any of these six and you'll see whack-a-mole bugs in production within the first day.

## Production reality check (read this BEFORE designing tests)

- **User transcripts are typically empty.** Gemini Live multimodal handles user audio end-to-end and does NOT emit a separate `user.transcription` event. The user-side regex (e.g. `USER_NAVIGATE_INTENT_RE` for bare imperatives like "open the product") will never fire on the production wire even if QA passes. Document the gap. Keep the user-side code path harmless in case a future model emits these — but don't design around it firing.
- **The model uses pronouns more than you'd expect.** "Tell me about [Product]" → describe → user: "Open it" → agent: "I'm opening it for you now" — no second mention of the product name anywhere in turn 3. Anaphora handling is mandatory, not optional.
- **The model loves trailing offers.** Almost every commitment turn ends with "Let me know if you want another / Want to see something else?" Order-aware suppression is mandatory.

## Safety rules

- **Allowlist-only.** Never derive a destination URL from transcript text directly. Map a known phrase to a known constant.
- **Require navigate-intent for ALL paths, including external links.** Without this, "you can find more on LinkedIn" would yank the user away mid-explanation.
- **Word-boundary regex on short phrases.** Short product/feature names (e.g. "Maven", "Oakhurst") will substring-match inside random sentences. Use `\b...\b`.
- **Track fired turn_ids.** Final transcript chunks can arrive multiple times due to network retries. Don't double-fire.
- **Clamp the fired set on teardown.** Reset on every new call so the same turn_id from a previous call doesn't suppress the new one.
- **Project loop BEFORE generic home check.** Otherwise a product name containing "home" (e.g. "Home Brain") matches the bare "home" word and routes to `/` instead of `/projects/home-brain`. Project-specific matching must beat the generic home intent.

## Implementation

```ts
// src/lib/voice/clientTools.ts

// Strict: requires committal verbs, not bare "open".
// NOTE: this regex hardcodes "alex" as the operator's first name (in the
// "here's [name]'s" and "alex's" patterns). When you adopt this skill,
// search-replace `alex` with the operator's first name throughout this
// file, or templatize it via a constant.
const NAVIGATE_INTENT_RE =
  /\b(opening|navigating|taking you|showing you|pulling up|loading up|heading (?:over )?to|sending you|here'?s\s+(?:the|your|his|her|alex'?s?|it|him|that)|i(?:'| wi)?ll (?:open|navigate|take you|show you|pull up|head|load|send you|grab)|let me (?:open|take you|show you|pull up|head|load|grab)|sending you to)\b/i;

// User-side imperatives (rarely fires in production — Gemini Live doesn't
// emit user.transcription — but kept harmless in case a future model does).
const USER_NAVIGATE_INTENT_RE =
  /\b(open|show me|take me|go to|go home|go back|back to|navigate to|head to|pull up|let me see|i want to see|bring up|launch|view|head home|take me home|book|schedule)\b/i;

// Question/offer phrases. Suppresses only when ORDERED before commitment.
const QUESTION_OR_OFFER_RE =
  /\b(would you like|do you want|want me to|should i|if you'?d like|if you want|can i|may i|just say|let me know|tell me if)\b/i;

// "Project page" / "writeup" / "details" — overrides domain-mention live intent.
const WRITEUP_INTENT_RE =
  /\b(project page|writeup|write[- ]up|the (?:page|writeup|details)|details page|portfolio (?:page|entry)|project entry|the card|the project entry|project details)\b/i;

const LIVE_DEMO_INTENT_RE =
  /\b(live(?:\s+(?:site|demo|version|app|build|one|page))?|running(?:\s+app)?|deployed|the actual (?:site|app|thing)|production|website|web ?site|demo|the app|the real (?:thing|app|site)|launching)\b/i;

// Mention of a custom domain or known TLD signals "user wants the live URL"
// even without the word "live" — unless WRITEUP_INTENT_RE overrides.
const DOMAIN_MENTION_RE =
  /\b\w[\w-]*\.(com|ai|org|io|fun|app|space|dev|co|net|me|xyz|tech)\b/i;

// Same operator-name caveat as NAVIGATE_INTENT_RE: replace "alex" with
// the operator's first name when adopting.
const LINKEDIN_INTENT_RE =
  /\b(linkedin|linked in|connect with (?:alex|him|her)|(?:alex'?s?|his|her) profile|work history|career page)\b/i;

const HOME_INTENT_RE =
  /\b(home(?:page)?|main page|back to (?:the )?(?:start|beginning|home)|the bento|the grid)\b/i;

// Map agent phrases to slugs in your app. List ALL the natural-language
// names the model might use when referring to each project, mapped to a
// single canonical slug. Phrases match case-insensitively with word
// boundaries.
const PROJECT_PHRASE_TO_SLUG: Array<{ phrases: readonly string[]; slug: string }> = [
  { phrases: ['acme product', 'acme', 'the acme app', 'acmeproduct.com'], slug: 'acme-product' },
  { phrases: ['widget co', 'widget', 'widget pro'], slug: 'widget-pro' },
  // ... your projects
];

function phraseMatches(text: string, phrase: string): boolean {
  if (/^[a-z0-9 ]+$/.test(phrase)) {
    const re = new RegExp(`\\b${phrase.replace(/\s+/g, '\\s+')}\\b`, 'i');
    return re.test(text);
  }
  return text.includes(phrase);
}

// Used by the client to track the most-recently-mentioned project across
// turns — caller stores the result in a ref and passes it back as
// contextSlug. Stateless on the matcher's side.
export function findMentionedSlug(text: string): string | null {
  const lower = text.toLowerCase();
  for (const { phrases, slug } of PROJECT_PHRASE_TO_SLUG) {
    if (phrases.some((p) => phraseMatches(lower, p))) return slug;
  }
  return null;
}

export function matchAndTriggerNavigation(
  transcriptText: string,
  routerPush: (path: string) => void,
  source: 'assistant' | 'user' = 'assistant',
  contextSlug?: string | null,
): string | null {
  const lower = transcriptText.toLowerCase();

  // (1) Require navigate-intent. Agent narrations need committal verbs;
  // user transcripts can use bare imperatives.
  const navigateRe =
    source === 'user' ? USER_NAVIGATE_INTENT_RE : NAVIGATE_INTENT_RE;
  const navigateMatch = lower.match(navigateRe);
  if (!navigateMatch) return null;

  // (2) Order-aware question/offer suppression: only suppress if the
  // offer phrase appears BEFORE the commitment phrase. This is the
  // critical fix — without it, "Sure, I'm opening it. Let me know if
  // you want another." silently fails because "let me know" is in the
  // suppressor list.
  const offerMatch = lower.match(QUESTION_OR_OFFER_RE);
  if (offerMatch && (offerMatch.index ?? 0) < (navigateMatch.index ?? 0)) {
    return null;
  }

  if (LINKEDIN_INTENT_RE.test(lower)) {
    window.open('https://linkedin.com/in/your-handle', '_blank', 'noopener,noreferrer');
    return 'linkedin';
  }

  // (3) Writeup intent overrides domain/live mentions.
  const wantsWriteup = WRITEUP_INTENT_RE.test(lower);
  const domainMentioned = DOMAIN_MENTION_RE.test(lower);
  const wantsLive =
    !wantsWriteup && (LIVE_DEMO_INTENT_RE.test(lower) || domainMentioned);

  const navigateTo = (slug: string): string => {
    const liveUrl = LIVE_URL_BY_SLUG[slug]; // your map
    if (wantsLive && liveUrl) {
      window.open(liveUrl, '_blank', 'noopener,noreferrer');
      return `${slug} (live)`;
    }
    routerPush(`/projects/${slug}`);
    return slug;
  };

  // (4a) Project loop BEFORE home check. "Home Brain" must beat bare "home".
  for (const { phrases, slug } of PROJECT_PHRASE_TO_SLUG) {
    if (phrases.some((p) => phraseMatches(lower, p))) return navigateTo(slug);
  }

  // (4b) ANAPHORA fallback: navigate-intent matched but no project phrase
  // appeared in the current text ("I'm opening it for you now").
  // Skip when home intent matches so "take me back home" still routes to /.
  if (contextSlug && !HOME_INTENT_RE.test(lower)) {
    return navigateTo(contextSlug);
  }

  if (HOME_INTENT_RE.test(lower)) {
    routerPush('/');
    return 'home';
  }
  return null;
}
```

## Wire-up in the stream-message handler

```ts
const fallbackFiredRef = useRef<Set<string>>(new Set());
const lastProjectSlugRef = useRef<string | null>(null);

// Inside stream-message handler:

// On real function calls: update context too so subsequent turns can resolve "it".
for (const call of calls) {
  executeToolCall({ name: call.name, arguments: call.arguments }, { routerPush: router.push });
  if (call.name === 'open_project' || call.name === 'open_project_new_tab' || call.name === 'open_live_demo') {
    const slug = call.arguments.slug;
    if (typeof slug === 'string') lastProjectSlugRef.current = slug;
  }
}

// On final transcript chunks (turn_status === 1) when no real call fired:
if (
  calls.length === 0 &&
  (parsed.object === 'assistant.transcription' || parsed.object === 'user.transcription') &&
  typeof parsed.text === 'string' &&
  parsed.turn_status === 1
) {
  const fallbackKey = `${parsed.object}-${parsed.turn_id}-${parsed.message_id ?? ''}`;

  // Update context BEFORE running the matcher so same-turn project mentions
  // resolve correctly ("Bazi Oracle is great. Opening it now.").
  const mentioned = findMentionedSlug(parsed.text);
  if (mentioned) lastProjectSlugRef.current = mentioned;

  if (!fallbackFiredRef.current.has(fallbackKey)) {
    const triggered = matchAndTriggerNavigation(
      parsed.text,
      router.push,
      parsed.object === 'user.transcription' ? 'user' : 'assistant',
      lastProjectSlugRef.current,
    );
    if (triggered) fallbackFiredRef.current.add(fallbackKey);
  }
}
```

Don't forget to clear both refs on call teardown:
```ts
const teardown = useCallback(async () => {
  fallbackFiredRef.current.clear();
  lastProjectSlugRef.current = null;
  // ... rest of cleanup
}, []);
```

## Drift-resistant QA harness

Build a standalone Node script (`scripts/qa-voice.mjs`) that **extracts the regex literals and slug mappings from the source file via lightweight text parsing**, instead of duplicating them. Test cases stay in the script; patterns stay in source — no drift.

```js
function extractRegex(name) {
  const re = new RegExp(`const\\s+${name}\\s*=\\s*\\n?\\s*(/[^\\n]+/[gimsuy]*)\\s*;`, 'm');
  const m = source.match(re);
  if (!m) throw new Error(`Could not extract ${name}`);
  return Function(`return ${m[1]};`)();
}

const NAVIGATE_INTENT_RE = extractRegex('NAVIGATE_INTENT_RE');
// ... extract all regexes
```

Test cases (group → transcript → expected). Replace `Acme Product` /
`acme-product` with your actual product names + slugs:

- **Project pages (writeup):** `"Sure, opening the Acme Product for you."` → `/projects/acme-product`
- **Live demos:** `"Opening acme.com for you in a new tab."` → `https://acme.com`
- **LinkedIn:** `"Sure, opening [operator]'s LinkedIn profile now."` → LinkedIn URL
- **Go home:** `"Taking you back to the homepage."` → `/`
- **Must NOT navigate:** `"Tell me about Acme Product — what would you like to know?"` → no fire
- **Writeup overrides live:** `"Opening the Widget Pro project page for you."` → `/projects/widget-pro` (not the live URL, even when a domain mention exists)
- **User-side imperatives:** `"Open the Widget Pro project page."` with `source: 'user'` → `/projects/widget-pro` (note: rarely fires in production — see "Production reality check" above)
- **Anaphora:** `"I'm opening it for you now. Let me know if you want to check out another one."` with `contextSlug: 'acme-product'` → `/projects/acme-product`
- **Anaphora doesn't override explicit:** `"Pulling up the Acme Product for you now."` with `contextSlug: 'widget-pro'` → `/projects/acme-product` (explicit beats context)
- **Anaphora doesn't beat home intent:** `"Taking you back to the homepage."` with `contextSlug: 'acme-product'` → `/`
- **Order-aware: commit then offer fires:** `"Sure, I'm opening it. Let me know if you want another."` → fires
- **Order-aware: offer then commit suppresses:** `"If you want, I'll open it."` → no fire
- **Past-tense narrations:** `"Opened the calendar now — pick a time."` → fires booking
- **Calendar mis-route guard:** `"I'm opening the calendar now... Want me to open the LinkedIn instead?"` → fires booking, NOT LinkedIn (commitment-text scoping)

Wire it into a `preship` script: `npm run preship` = `npm run build && npm run qa:voice`.

## Why this matters

- Without anaphora handling alone, ~40% of "show me X" requests fail in real conversations because the model uses pronouns
- Without order-aware suppression, every commit-then-offer turn (which is most of them) silently fails
- With all four behaviors layered + a tuned prompt, success rate is high but not 100% — preview models lie sometimes in ways no regex can catch. Read the analytics; iterate.

## When to retire this

Workaround for preview-model unreliability. When Gemini Live becomes GA with reliable function calling (or when you swap to a more reliable model), you can remove the fallback. Or leave it as a safety net — it's cheap and harmless.
