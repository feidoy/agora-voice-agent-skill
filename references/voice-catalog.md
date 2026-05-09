# Gemini Live Voice Catalog

30 named voices, sourced from `https://ai.google.dev/gemini-api/docs/speech-generation`. Google ships single-word descriptors. No published gender/accent metadata — test each to find the one that fits your brand.

## Catalog

| Voice | Descriptor |
|---|---|
| Aoede | Breezy |
| Kore | Firm |
| Charon | Informative |
| Puck | Upbeat |
| Fenrir | Excitable |
| Zephyr | Bright |
| Leda | Youthful |
| Orus | Firm |
| Callirrhoe | Easy-going |
| Autonoe | Bright |
| Enceladus | Breathy |
| Iapetus | Clear |
| Umbriel | Easy-going |
| Algieba | Smooth |
| Despina | Smooth |
| Erinome | Clear |
| Algenib | Gravelly |
| Rasalgethi | Informative |
| Laomedeia | Upbeat |
| Achernar | Soft |
| Alnilam | Firm |
| Schedar | Even |
| Gacrux | Mature |
| Pulcherrima | Forward |
| Achird | Friendly |
| Zubenelgenubi | Casual |
| Vindemiatrix | Gentle |
| Sadachbia | Lively |
| Sadaltager | Knowledgeable |
| Sulafat | Warm |

## Allowlist validator

Always validate user-supplied voice strings against this catalog server-side. Don't trust the client.

```ts
// src/lib/voice/voiceCatalog.ts
export const GEMINI_VOICES = [
  { name: 'Aoede', descriptor: 'Breezy' },
  { name: 'Kore', descriptor: 'Firm' },
  { name: 'Charon', descriptor: 'Informative' },
  { name: 'Puck', descriptor: 'Upbeat' },
  { name: 'Fenrir', descriptor: 'Excitable' },
  { name: 'Zephyr', descriptor: 'Bright' },
  { name: 'Leda', descriptor: 'Youthful' },
  { name: 'Orus', descriptor: 'Firm' },
  { name: 'Callirrhoe', descriptor: 'Easy-going' },
  { name: 'Autonoe', descriptor: 'Bright' },
  { name: 'Enceladus', descriptor: 'Breathy' },
  { name: 'Iapetus', descriptor: 'Clear' },
  { name: 'Umbriel', descriptor: 'Easy-going' },
  { name: 'Algieba', descriptor: 'Smooth' },
  { name: 'Despina', descriptor: 'Smooth' },
  { name: 'Erinome', descriptor: 'Clear' },
  { name: 'Algenib', descriptor: 'Gravelly' },
  { name: 'Rasalgethi', descriptor: 'Informative' },
  { name: 'Laomedeia', descriptor: 'Upbeat' },
  { name: 'Achernar', descriptor: 'Soft' },
  { name: 'Alnilam', descriptor: 'Firm' },
  { name: 'Schedar', descriptor: 'Even' },
  { name: 'Gacrux', descriptor: 'Mature' },
  { name: 'Pulcherrima', descriptor: 'Forward' },
  { name: 'Achird', descriptor: 'Friendly' },
  { name: 'Zubenelgenubi', descriptor: 'Casual' },
  { name: 'Vindemiatrix', descriptor: 'Gentle' },
  { name: 'Sadachbia', descriptor: 'Lively' },
  { name: 'Sadaltager', descriptor: 'Knowledgeable' },
  { name: 'Sulafat', descriptor: 'Warm' },
] as const;

const VOICE_NAMES = new Set(GEMINI_VOICES.map((v) => v.name));

export function isValidVoice(name: unknown): name is string {
  return typeof name === 'string' && VOICE_NAMES.has(name);
}

export const DEFAULT_VOICE = 'Kore';
```

## Picking voices for context

Tested defaults that worked well in our build:
- **Kore (Firm)** — production default at alexlee.space. Authoritative without being cold.
- **Aoede (Breezy)** — softer, more conversational. Good for casual/playful brands.
- **Charon (Informative)** — male-presenting, deeper. Good for serious/technical.
- **Sulafat (Warm)** — explicitly warm. Good for hospitality/wellness brands.

Avoid: Fenrir (Excitable) and Pulcherrima (Forward) for professional contexts — too theatrical.

## UI pattern that worked

A picker showing all 30 in a 2-column grid. Click a voice = highlight selected (don't auto-start). Bottom CTA "Initiate Call" commits the choice and starts the session. Voice persists in `localStorage` under `voiceAgent.voice` so returning visitors hear the last voice they liked.
