---
name: agentforce-agent-api
description: >
  Integrates Salesforce Agentforce agents into external applications via the
  Agent API (REST), Mobile SDK (iOS/Android/React Native), and Testing API.
  Covers OAuth Client Credentials auth, session lifecycle, synchronous and
  streaming messaging, agent variables, token caching, the backend-for-frontend
  pattern, Service Agent Markdown stripping, and error diagnosis. Use when
  building chat interfaces with Agentforce, setting up Agent API authentication,
  debugging Agent API errors, connecting external, web, or mobile apps to
  Salesforce agents, passing context variables, or testing agents
  programmatically.
license: MIT
metadata:
  docs:
    - "https://developer.salesforce.com/docs/ai/agentforce/references/agent-api?meta=summary"
    - "https://developer.salesforce.com/docs/ai/agentforce/guide/agent-api-get-started.html"
    - "https://developer.salesforce.com/docs/ai/agentforce/guide/agent-api-troubleshooting.html"
  promptSignals:
    phrases:
      - "agentforce"
      - "agent api"
      - "salesforce agent"
      - "einstein ai-agent"
      - "agentforce agent"
      - "maximum logins exceeded"
      - "chatbot_api"
      - "bypassuser"
      - "external client app"
    allOf:
      - [agentforce, session]
      - [agentforce, chat]
      - [salesforce, agent]
      - [agent, sequenceid]
    anyOf:
      - "agentforce"
      - "agent api"
      - "einstein ai-agent"
      - "chatbot_api"
    noneOf:
      - "github actions agent"
      - "user agent string"
      - "browser agent"
  importPatterns:
    - "@salesforce/react-native-agentforce"
  bashPatterns:
    - '\bsf\s+agent\s+(preview|test|generate|publish|activate)\b'
    - '\bnpm\s+(install|i|add)\s+[^\n]*@salesforce/react-native-agentforce'
  urlPatterns:
    - "api\\.salesforce\\.com/einstein/ai-agent"
    - "/services/oauth2/token"
retrieval:
  aliases:
    - agentforce agent api
    - salesforce agent integration
    - einstein ai-agent api
    - agentforce chat integration
  intents:
    - talk to a salesforce agent from an app
    - authenticate to the agent api
    - build an agentforce chat ui
    - debug agent api errors
    - test an agentforce agent
  entities:
    - Agentforce
    - Agent API
    - External Client App
    - Client Credentials flow
    - Testing API
    - Mobile SDK
---

# Agentforce Agent API Integration

Build interfaces powered by Salesforce Agentforce agents. This skill covers the Agent API, authentication, the backend-for-frontend pattern, Mobile SDKs, the Testing API, and the pitfalls that only show up in production.

This skill is **harness-agnostic** — the guidance and code apply whether the calling app is Next.js, plain Express/Node, Python, a mobile app, or anything else that can make HTTPS requests.

## Quick Start — Minimum Viable Call

Three steps to talk to an agent:

### 1. Get a Token

```
POST https://{MY_DOMAIN}/services/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id={KEY}&client_secret={SECRET}
```

Requires an External Client App with Client Credentials flow. Full setup in [auth-setup.md](auth-setup.md).

### 2. Start a Session

```
POST https://api.salesforce.com/einstein/ai-agent/v1/agents/{AGENT_ID}/sessions
Authorization: Bearer {TOKEN}
Content-Type: application/json
```

```json
{
  "externalSessionKey": "<uuid>",
  "instanceConfig": { "endpoint": "https://{MY_DOMAIN}" },
  "streamingCapabilities": { "chunkTypes": ["Text"] },
  "bypassUser": true
}
```

### 3. Send a Message

```
POST https://api.salesforce.com/einstein/ai-agent/v1/sessions/{SESSION_ID}/messages
Authorization: Bearer {TOKEN}
Content-Type: application/json
```

```json
{
  "message": { "type": "Text", "text": "Hello!", "sequenceId": 1 }
}
```

Response text is in `messages[].message` (not `.text`). Increment `sequenceId` per turn.

---

## Critical Rules

These are the rules that cost the most time to learn the hard way:

1. **API host**: Always `api.salesforce.com/einstein/ai-agent/v1` — My Domain goes in the request **body** only (`instanceConfig.endpoint`), never in the URL.
2. **sequenceId**: Must be **inside** the `message` object, not at the root.
3. **Response field**: Text lives in `messages[].message`, not `messages[].text`.
4. **Run As user**: Must have a full Salesforce license — never an Einstein Agent service user (they hit `Maximum Logins Exceeded`).
5. **Token caching**: Mandatory. Salesforce enforces 3,600 logins/user/hour. Never retry on rate limit.
6. **Env var trimming**: Always `.trim()` — hosting platforms can append a trailing `\n`.
7. **Agent type**: The Agent API is **not supported** for "Agentforce (Default)" agents.
8. **Timeout**: 120-second timeout per call; HTTP 500 on timeout.
9. **Session end needs a reason header**: `DELETE /sessions/{id}` returns **400** unless you send `x-session-end-reason` (e.g. `UserRequest`). See [api-reference.md](api-reference.md#end-session).
10. **Service Agents strip Markdown**: LLM-generated text comes back as **plain text** — bold, headings, and links are removed by the platform. Do rich formatting client-side. See [web-integration.md](web-integration.md#service-agents-strip-markdown).
11. **Transient 5xx happen**: The Agent API occasionally returns a one-off 502/503/504. Retry **once** with a short backoff before surfacing an error. See [web-integration.md](web-integration.md#transient-5xx-retry).

## Never Expose Tokens to the Browser

The OAuth token grants agent access for the whole org. Put a thin **backend-for-frontend (BFF)** between the browser and Salesforce: the server holds the client secret, caches the token, owns sessions, and exposes only narrow `/api/*` routes. The client never sees a Salesforce credential.

```
Browser ──► Your BFF (holds secret, caches token, owns session) ──► Agent API
```

Full pattern, including session mapping and the concierge-style chat route, in [web-integration.md](web-integration.md).

## Session Lifecycle

```
Start Session → Send Messages (sync or stream) → End Session (with reason header)
                     ↓
              Submit Feedback (optional)
```

- **Synchronous**: Full response in one HTTP response.
- **Streaming**: SSE events — `ProgressIndicator` → `TextChunk` → `Inform` → `EndOfTurn`.
- **End session**: `DELETE /sessions/{SESSION_ID}` **with** `x-session-end-reason` header.
- **Feedback**: `POST /sessions/{SESSION_ID}/feedback` with `feedbackId` from the Inform message.

Full endpoint reference in [api-reference.md](api-reference.md).

## Agent Variables

Pass context and custom variables on session start or with messages:

```json
{
  "variables": [
    { "name": "$Context.EndUserLanguage", "value": "en_US" },
    { "name": "team_descriptor", "value": "West Region" }
  ]
}
```

Key rules:
- Context variables (`$Context.*`) are read-only after session start (except `EndUserLanguage`).
- Custom field variables: omit the `__c` suffix (`Conversation_Key__c` → `$Context.Conversation_Key`).
- Must check "Allow value to be set by API" in Builder; `visibility: external` in metadata.

Full variable reference in [api-reference.md](api-reference.md#agent-variables).

## Token Caching

Salesforce enforces a **3,600 logins/user/hour** rolling limit. Cache the token, expire it 60 seconds early, dedupe concurrent requests behind one in-flight Promise, and **never** retry on `login rate exceeded`.

In frameworks with hot-module reload (e.g. Next.js dev), module-scoped variables reset on every save — cache on `globalThis` so the token survives reloads:

```typescript
const g = globalThis as typeof globalThis & {
  __sfToken?: { token: string; expiresAt: number };
};
if (!g.__sfToken) g.__sfToken = { token: '', expiresAt: 0 };
```

Full caching patterns in [auth-setup.md](auth-setup.md#token-caching).

## Quick Error Reference

| Error | Fix |
|-------|-----|
| `Maximum Logins Exceeded` | Change Run As to admin / Integration User (full license) |
| `login rate exceeded` | Wait 15–30 min; add token caching; stop retrying |
| 404 `URL No Longer Exists` | Use `api.salesforce.com`, not My Domain, as the API host |
| `Missing required creator property 'sequenceId'` | Move `sequenceId` inside the `message` object |
| `Bad force-config endpoint` | `.trim()` all env vars |
| No text in response | Read `msg.message`, not `msg.text` |
| `DELETE` session returns 400 | Add the `x-session-end-reason` header |
| Bold/links missing in replies | Service Agents strip Markdown — format client-side |
| Occasional 502/503/504 | Retry once with backoff before surfacing an error |
| Session stuck "Connecting..." | Use the `AbortController` pattern (dev double-mount) |
| `EngineConfigLookupException` | Use My Domain URL, not the Lightning URL |

Full diagnostic checklist in [troubleshooting.md](troubleshooting.md).

## Getting the Agent ID

**Legacy Builder**: 18-char ID from the URL in Setup → Agentforce Agents.

**New Builder**: Query `SELECT Id FROM BotDefinition WHERE DeveloperName = 'Agent_Name'`.

Details in [api-reference.md](api-reference.md#getting-the-agent-id).

## Deep References

| Topic | File |
|-------|------|
| Full API endpoints, payloads, variables, citations, agent ID | [api-reference.md](api-reference.md) |
| OAuth setup, ECA creation, token caching patterns | [auth-setup.md](auth-setup.md) |
| Backend-for-frontend, Markdown stripping, retries, rich rendering | [web-integration.md](web-integration.md) |
| Error codes, diagnostics, common integration bugs | [troubleshooting.md](troubleshooting.md) |
| iOS, Android, React Native Mobile SDKs, Enhanced Chat v2 | [mobile-sdk.md](mobile-sdk.md) |
| Testing API, Agentforce DX testing, CI/CD | [testing-api.md](testing-api.md) |
| All Agentforce tools, SDKs, and builder comparison | [ecosystem.md](ecosystem.md) |
