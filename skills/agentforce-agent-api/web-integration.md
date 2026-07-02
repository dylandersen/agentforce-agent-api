# Web Integration — Backend-for-Frontend, Rendering, Resilience

How to wire an Agentforce agent into a web or mobile front end safely. These patterns are language-agnostic; the examples use Node/Express and TypeScript, but the shape applies to any server runtime.

> Prerequisite: OAuth + token caching from [auth-setup.md](auth-setup.md), endpoints from [api-reference.md](api-reference.md).

## Why a backend-for-frontend (BFF)

The OAuth access token grants agent access for the whole org. It must **never** reach the browser. Put a thin server between the client and Salesforce:

```
Browser ──►  Your BFF  ──►  Agent API (api.salesforce.com)
             • holds client secret
             • caches the OAuth token
             • owns the agent session
             • exposes only narrow /api/* routes
```

The client sends plain user text to your own routes; the BFF attaches auth, manages the session, and returns only the agent's reply. Benefits: no secret leakage, one shared token cache, a single place to add retries, rate limiting, logging, and a site gate.

### Minimal route surface

| Route | Purpose |
|-------|---------|
| `POST /api/chat` | Accepts `{ text }`, ensures a session, sends the message, returns `{ reply }` |
| `POST /api/chat/end` | Ends the session (`x-session-end-reason` header) |

Keep the Salesforce session id server-side, keyed by your own browser session/cookie — do not return it to the client.

## Session and sequenceId bookkeeping

`sequenceId` must increment per message **within a session**, and it lives inside the `message` object. Track it server-side alongside the session id:

```typescript
// Keyed by your app's session, not exposed to the client
const agentSessions = new Map<string, string>();   // appSession -> salesforceSessionId
const agentSeq = new Map<string, number>();         // salesforceSessionId -> last sequenceId

async function sendAgentMessage(sfSessionId: string, text: string) {
  const seq = (agentSeq.get(sfSessionId) ?? 0) + 1;
  const body = { message: { type: 'Text', text, sequenceId: seq } };

  const { status, json } = await agentFetchWithRetry(
    'POST',
    `/sessions/${sfSessionId}/messages`,
    body,
  );
  if (status !== 200) {
    throw new Error(`Agent message failed (${status})`);
  }
  // Only commit the sequenceId after success, so a retry can safely reuse it.
  agentSeq.set(sfSessionId, seq);

  const reply = extractReply(json) ?? "I'm here — could you try rephrasing that?";
  return reply;
}

function extractReply(json: any): string | undefined {
  // Text is in messages[].message — NOT messages[].text
  const informs = (json.messages ?? []).filter((m: any) => m.message);
  return informs.map((m: any) => m.message).join('\n\n') || undefined;
}
```

## Transient 5xx retry

The Agent API occasionally returns a one-off `502` / `503` / `504` from an upstream blip. A single silent retry with a short backoff keeps a hiccup from becoming a scary error. It is safe because the `sequenceId` above is not committed until the call succeeds — retrying reuses the same value.

```typescript
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function agentFetchWithRetry(method: string, path: string, body: unknown) {
  let status = 0;
  let json: any;
  for (let attempt = 0; attempt < 2; attempt++) {
    ({ status, json } = await agentFetch(method, path, body));
    if (status !== 502 && status !== 503 && status !== 504) break;
    if (attempt === 0) await sleep(600);   // brief backoff, retry once
  }
  return { status, json };
}
```

Do **not** apply this retry to OAuth token requests — retrying `login rate exceeded` burns more quota (see [auth-setup.md](auth-setup.md#token-caching)).

## Service Agents strip Markdown

**The single most surprising behavior.** The Agentforce **Service Agent** platform sanitizes Markdown out of every LLM-generated message before it reaches the Agent API. Even if the agent's instructions say to emit `**bold**`, `## headings`, or `[links](url)`, the client receives **plain text** with the formatting removed.

Consequences and how to handle them:

1. **Do not rely on the agent for rich formatting.** Instructing the agent to "use Markdown" does not work for Service Agents. Format on the client instead.
2. **Emit stable, parseable plain-text structure from the agent**, then parse it client-side. Blank-line-separated blocks, a `Name — meta` convention per line, and bare URLs survive sanitization and are easy to render richly.
3. **Render bare URLs as buttons/links client-side.** Because `[text](url)` is stripped, have actions output a bare `https://…` URL and let the client detect it and render a real button.

```typescript
// Client-side: turn the agent's sanitized plain text into rich elements.
// Split on blank lines into blocks; within a block, treat lines as list items.
function renderAgentText(text: string): Block[] {
  return text
    .split(/\n{2,}/)
    .map((block) => {
      const lines = block.split('\n').filter((l) => l.replace(/\u00A0/g, '').trim());
      // A short line with no list markers reads as a heading.
      if (lines.length === 1 && lines[0].length < 48 && !/https?:\/\//.test(lines[0])) {
        return { type: 'heading', text: lines[0] };
      }
      return { type: 'list', items: lines.map(parseNameMeta) };
    });
}

// "Godzilla Tribute — Fri Jul 10 · 7:00 PM · Main Hall · Buy tickets: https://…"
function parseNameMeta(line: string) {
  const url = line.match(/https?:\/\/\S+/)?.[0];
  const [name, ...rest] = line.split(' — ');
  return { name, meta: rest.join(' — '), url };
}
```

The exact parser depends on the structure your agent emits — the point is: **decide a plain-text contract, keep the agent emitting it consistently, and do the styling in the client.**

> Employee Agents and the Mobile SDK's native UI do not have this exact constraint; this is specifically a Service Agent + custom-UI concern.

## Client-side session bootstrap (dev double-mount)

Frameworks that double-invoke effects in development (e.g. React Strict Mode) will fire your "start session" call twice. Use an `AbortController` to cancel the duplicate rather than a `useRef` guard, which can deadlock the UI in "Connecting…":

```typescript
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/chat/start', { method: 'POST', signal: controller.signal })
    .then((r) => r.json())
    .then((data) => setReady(true))
    .catch((err) => {
      if (err.name === 'AbortError') return;   // expected on the duplicate mount
      // handle real errors
    });
  return () => controller.abort();
}, []);
```

## Ending sessions cleanly

End the session when the user closes the chat or the browser session expires, and send the required reason header (a missing header returns 400):

```typescript
async function endAgentSession(sfSessionId: string) {
  await agentFetch('DELETE', `/sessions/${sfSessionId}`, undefined, {
    'x-session-end-reason': 'UserRequest',
  });
  agentSeq.delete(sfSessionId);
}
```

## Deployment checklist

- [ ] Client secret and token live only on the server; browser sees neither.
- [ ] Env vars `.trim()`-ed; My Domain normalized to include `https://`.
- [ ] Token cached (expire 60s early), concurrent requests deduped, no retry on rate limit.
- [ ] Transient 5xx retried once with backoff on message sends only.
- [ ] `sequenceId` incremented per message, committed only after success.
- [ ] Rich formatting done client-side (Service Agents strip Markdown).
- [ ] Session ended with `x-session-end-reason` on close.
