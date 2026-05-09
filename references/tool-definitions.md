# Tool Definitions for Gemini Live

Tool functions go in `additionalParams.tools` on the `GeminiLive` constructor. The SDK forwards them to Gemini Live's `setup.tools` field.

## Critical: lowercase schema types

Gemini Live's setup validator rejects proto-style uppercase (`OBJECT`, `STRING`) with `code 500` from the `mllm` module. The error message is the unhelpful `receive loop error {e}` (Python format string with unfilled placeholder). **Always use lowercase** per JSON Schema / OpenAPI 3 spec.

```ts
// WRONG — will produce silent agent (joins muted)
parameters: {
  type: 'OBJECT',
  properties: {
    slug: { type: 'STRING' },
  },
}

// CORRECT
parameters: {
  type: 'object',
  properties: {
    slug: { type: 'string' },
  },
}
```

## Template

```ts
additionalParams: {
  tools: [
    {
      functionDeclarations: [
        {
          name: 'open_project',
          description:
            "Navigate the user's current tab to a project page. Use when they say 'show me X', 'open X', or 'take me to X'.",
          parameters: {
            type: 'object',
            properties: {
              slug: {
                type: 'string',
                description:
                  'Project slug. Allowed: <comma-separated list of valid slugs>',
              },
            },
            required: ['slug'],
          },
        },
        {
          name: 'open_live_demo',
          description:
            "Open the LIVE deployed app of a project in a new browser tab. Use when user wants the running app: 'open the live site', 'show me the actual app', 'launch the demo'.",
          parameters: {
            type: 'object',
            properties: {
              slug: {
                type: 'string',
                description: 'Project slug.',
              },
            },
            required: ['slug'],
          },
        },
        {
          name: 'open_external_link',
          description: "Open a known external URL (LinkedIn, GitHub, etc) in a new tab.",
          parameters: {
            type: 'object',
            properties: {},
          },
        },
      ],
    },
  ],
},
```

## Tool design principles

1. **Allowlist-driven, not free-form.** Don't accept arbitrary URLs. Hardcode the destinations or look them up by ID/slug.
2. **Description is doctrine.** The model picks tools based on the description text. List trigger phrases the user might say.
3. **Specific > general.** "open_live_demo" works better than a single "navigate" tool with a `mode` parameter, because Gemini Live's tool selector is shape-matching, not semantic.
4. **Empty `properties: {}` is fine** for parameter-less tools. Don't omit `parameters` entirely — some validators reject that.

## Force the model to actually call the tool

Even with good tool definitions, Gemini Live preview models often **narrate** function calls without invoking them. Mitigate at two levels:

**1. Hammer the prompt:**

```
# Tools — CRITICAL BEHAVIOR
THE MOST IMPORTANT RULE: When the user asks to see, open, view, or navigate
to ANYTHING, you MUST INVOKE the matching function. Do NOT just say "I'm
opening X for you" — that does NOTHING. The user sees nothing change unless
you actually call the function.

Correct:
1. User: "Show me the X"
2. You: invoke open_project({ slug: "x" })  ← REQUIRED
3. You speak: "Opening it for you."

Wrong:
1. User: "Show me the X"
2. You speak: "I'm opening X for you now."  ← but no function called, nothing happens
```

**2. Add a transcript-pattern fallback** — see `transcript-fallback.md`.
