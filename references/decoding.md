# Wire Format and Stream Message Decoding

The single biggest gotcha after model setup is reading what Agora actually sends. This was figured out from runtime logs because no Agora doc states the wire format clearly.

## Format

Each `stream-message` event delivers a `Uint8Array` payload. Decode as UTF-8 to get a text string with this structure:

```
{messageId}|{seq}|{total}|{base64-encoded-json-body}
```

- `messageId` — opaque string, used to reassemble multi-part messages
- `seq` — 1-indexed part number
- `total` — total parts for this message
- body — base64-encoded JSON (the part-data; concatenate all parts before base64 decoding)

Most messages fit in a single part (`seq=1, total=1`). Multi-part is rare but does happen with very long transcript chunks.

## Common decoded message types

```jsonc
// Transcript chunk (the firehose — silence this in your console.log filter)
{
  "object": "assistant.transcription",
  "text": "I'll open The Bazi Oracle for you now.",
  "data_type": "transcribe",
  "turn_status": 1,           // 1 = final chunk, 0 = streaming partial
  "turn_id": 4,
  "message_id": "abc123"
}

// Agent state changes (idle, listening, speaking, silent)
{
  "object": "message.state",
  "state": "listening",
  "turn_id": 0,
  "data_type": "message"
}

// Errors from the MLLM bridge — code 500 = Gemini Live setup rejected
{
  "object": "message.error",
  "module": "mllm",
  "message": "receive loop error {e}",   // unfilled Python format placeholder
  "code": 500
}

// Tool call — shape varies by SDK version, parser handles 4+ shapes
{
  "type": "tool_call",
  "name": "open_project",
  "arguments": { "slug": "your-project-slug" }
}
```

## Parser

```ts
// Inside client.on('stream-message', ...)
const partBuffers = new Map<string, string[]>();

client.on('stream-message', (uid, payload) => {
  try {
    const wire = new TextDecoder().decode(payload);
    const parts = wire.split('|');
    if (parts.length < 4) return;

    const messageId = parts[0];
    const seq = Number(parts[1]);
    const total = Number(parts[2]);
    const part = parts.slice(3).join('|'); // body may contain pipes
    if (!Number.isFinite(seq) || !Number.isFinite(total)) return;

    let buf = partBuffers.get(messageId);
    if (!buf) {
      buf = new Array(total).fill('');
      partBuffers.set(messageId, buf);
    }
    buf[seq - 1] = part;
    if (buf.some((s) => !s)) return; // not all parts in yet
    partBuffers.delete(messageId);

    const base64 = buf.join('');
    let json: string;
    try {
      json = atob(base64);
    } catch {
      return;
    }

    let parsed: Record<string, unknown>;
    try {
      parsed = JSON.parse(json);
    } catch {
      return;
    }

    // Skip the firehose of transcript chunks unless you need them
    const dataType = parsed.data_type as string | undefined;
    const objectType = parsed.object as string | undefined;
    if (dataType !== 'transcribe' && objectType !== 'assistant.transcription') {
      console.log('[voice] decoded msg:', parsed);
    }

    // --- Find tool/function calls ---
    const calls: Array<{ name: string; arguments: Record<string, unknown> }> = [];
    const pushCall = (name: unknown, rawArgs: unknown) => {
      if (typeof name !== 'string') return;
      let args: Record<string, unknown> = {};
      if (rawArgs && typeof rawArgs === 'object') {
        args = rawArgs as Record<string, unknown>;
      } else if (typeof rawArgs === 'string') {
        try { args = JSON.parse(rawArgs); } catch { /* leave empty */ }
      }
      calls.push({ name, arguments: args });
    };

    // Shape 1: { type: 'tool_call', name, arguments }
    if (parsed.type === 'tool_call' || dataType === 'tool_call') {
      pushCall(parsed.name, parsed.arguments);
    }
    // Shape 2: { tool_calls: [{ name, arguments }] } or [{ function: {...} }]
    if (Array.isArray(parsed.tool_calls)) {
      for (const tc of parsed.tool_calls as Array<Record<string, unknown>>) {
        const fn = tc.function as Record<string, unknown> | undefined;
        if (fn) pushCall(fn.name, fn.arguments);
        else pushCall(tc.name, tc.arguments);
      }
    }
    // Shape 3: { function_call: { name, args } } — Gemini Live raw
    const fc = parsed.function_call as
      | { name?: string; args?: unknown; arguments?: unknown }
      | undefined;
    if (fc) pushCall(fc.name, fc.args ?? fc.arguments);
    // Shape 4: { toolCall: { functionCalls: [...] } } — Gemini Live serverContent
    const toolCall = parsed.toolCall as
      | { functionCalls?: Array<{ name?: string; args?: unknown }> }
      | undefined;
    if (toolCall && Array.isArray(toolCall.functionCalls)) {
      for (const fcItem of toolCall.functionCalls) {
        pushCall(fcItem.name, fcItem.args);
      }
    }

    for (const call of calls) {
      executeYourTool(call.name, call.arguments);
    }
  } catch (err) {
    console.warn('[voice] stream-message handler error:', err);
  }
});
```

## Debugging tips

1. **Always log the raw wire string first.** Decode it manually in browser DevTools (`atob(base64part)`) to see what Agora actually sent.
2. **Filter out `assistant.transcription`** in your console — they fire constantly and drown out signal.
3. **`message.error` with `module: 'mllm'`** is the most useful diagnostic. Its `message` field tells you which side rejected the setup. The `{e}` Python format placeholder is an Agora bug — there's no way to get the original error without checking Agora's server logs (which you don't have access to).
4. **Listener registration order matters.** `client.on('stream-message', ...)` must be called BEFORE `client.join()`. Late binding misses early messages including the agent's first state transition.
