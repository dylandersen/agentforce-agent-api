# Agentforce Agent API Skill

A **universal Agent Skill** that teaches an AI coding agent how to integrate Salesforce Agentforce agents into external applications. It covers the Agent API, the backend-for-frontend pattern, Mobile SDKs, Enhanced Chat v2, Testing API workflows, OAuth Client Credentials flow, session management, token caching, Service Agent Markdown handling, and the pitfalls that only surface in production.

The skill follows the portable [Agent Skills](https://code.claude.com/docs/en/skills) format (`SKILL.md` with YAML frontmatter) and works across any harness that reads that format — including Claude Code and Cursor.

## What This Skill Covers

- **Authentication** — OAuth Client Credentials flow, External Client App setup, Run As user requirements, and token caching to avoid rate limits
- **Backend-for-frontend** — keeping the token server-side, session/`sequenceId` bookkeeping, and a minimal `/api/*` route surface
- **Session Management** — creating, messaging, and ending sessions, including the required `x-session-end-reason` header
- **Agent ID Lookup** — finding the API name and 18-character agent ID for both legacy and new Agentforce Builder flows
- **Message Handling** — request/response structure, `sequenceId` placement, and streamed (SSE) response parsing
- **Rendering** — why Service Agents strip Markdown and how to render rich output (headings, buttons, links) client-side
- **Resilience** — silent single retry on transient 502/503/504, `AbortController` for dev double-mount, early token expiry
- **Variables** — context and custom variable handling, including API-settable rules and metadata fields
- **Mobile + Web Chat** — React Native, iOS, Android, Enhanced Chat v2, and inline mode
- **Testing** — Testing API, Agentforce DX, Connect API, and custom evaluation criteria
- **Error Diagnosis** — `Maximum Logins Exceeded`, `login rate exceeded`, 404s, missing `sequenceId`, session-end 400s, and domain mismatches

## Repository Layout

There is **one canonical copy** of the skill; the per-harness directories are symlinks to it, so nothing drifts:

```
skills/agentforce-agent-api/        ← canonical source of truth
├── SKILL.md                        ← entry point (loaded first)
├── api-reference.md                ← endpoints, payloads, variables, citations, agent ID
├── auth-setup.md                   ← OAuth, ECA creation, token caching
├── web-integration.md              ← BFF, Markdown stripping, retries, rich rendering
├── troubleshooting.md              ← error codes and diagnostics
├── mobile-sdk.md                   ← iOS, Android, React Native, Enhanced Chat v2
├── testing-api.md                  ← Testing API, Agentforce DX, CI/CD
└── ecosystem.md                    ← Agentforce tool map and builder comparison

.claude/skills/agentforce-agent-api → ../../skills/agentforce-agent-api   (symlink)
.cursor/skills/agentforce-agent-api → ../../skills/agentforce-agent-api   (symlink)
```

Edit files under `skills/agentforce-agent-api/` — both harnesses pick up the change automatically.

## Installation

### Any repo (copy the canonical folder)

Copy `skills/agentforce-agent-api/` into your project's skills directory:

```bash
# Claude Code
mkdir -p .claude/skills && cp -R skills/agentforce-agent-api .claude/skills/

# Cursor
mkdir -p .cursor/skills && cp -R skills/agentforce-agent-api .cursor/skills/
```

### Via the skills CLI

```bash
npx skills add dylandersen/agentforce-agent-api
```

## Usage

Once installed, the agent can activate this skill automatically when you ask it to:

- Build a chat interface backed by an Agentforce agent
- Set up Agent API authentication
- Find the correct agent API name or agent ID
- Debug Agent API errors (`Maximum Logins Exceeded`, 404s, missing fields, session-end 400s)
- Connect an external, web, or mobile app to a Salesforce agent
- Fix rich formatting that "disappears" in agent replies (Service Agent Markdown stripping)
- Set up or test Enhanced Chat v2, Mobile SDK, or Testing API workflows

### Example Prompts

```
"Set up Agentforce Agent API authentication in my app"
"Create a chat component that talks to my Salesforce agent"
"I'm getting 'Maximum Logins Exceeded' when calling the Agent API — help me fix it"
"My agent's bold text and links are disappearing in the chat UI — why?"
"Add token caching for my Salesforce OAuth flow"
"How do I find the API name and agent ID for my Agentforce agent?"
```

## Prerequisites

1. A Salesforce org with Agentforce enabled and at least one activated agent
2. An External Client App configured with OAuth scopes: `api`, `refresh_token`/`offline_access`, `chatbot_api`, `sfap_api`
3. Client Credentials flow enabled with a valid Run As user that has full API login access (**not** an Einstein Agent service user)
4. Your agent API name and Agent ID
5. The Agent API is **not supported** for "Agentforce (Default)" agents

## Environment Variables

```env
SALESFORCE_MY_DOMAIN=your-org.my.salesforce.com
SALESFORCE_CONSUMER_KEY=your-external-client-app-client-id
SALESFORCE_CONSUMER_SECRET=your-external-client-app-client-secret
SALESFORCE_AGENT_ID=0XxHu000000XXXXKXX
```

Keep the consumer secret and the OAuth token **server-side only** — never ship them in a client bundle.

## License

MIT
