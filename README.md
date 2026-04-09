# Agentforce Agent API Skill

A dual-harness skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and Cursor that teaches how to integrate Salesforce Agentforce agents into external applications. It covers the Agent API, Mobile SDKs, Enhanced Chat v2, Testing API workflows, OAuth Client Credentials flow, session management, token caching, and common debugging pitfalls.

## What This Skill Does

When installed, the skill gives the agent deep knowledge of the Agentforce ecosystem, including:

- **Authentication** — OAuth Client Credentials flow, External Client App setup, Run As user requirements, and token caching to avoid rate limits
- **Session Management** — Creating, messaging, and ending Agent API sessions with the correct endpoint formats
- **Agent ID Lookup** — How to find the API name and 18-character agent ID for both legacy and new Agentforce Builder flows
- **Message Handling** — Proper request/response body structure, including `sequenceId` placement and streamed response parsing
- **Variables** — Context and custom variable handling, including API-settable rules and metadata fields
- **Mobile + Web Chat** — React Native, iOS, Android, Enhanced Chat v2, and inline mode setup
- **Testing** — Testing API, Agentforce DX, Connect API, and custom evaluation criteria
- **Token Caching** — `globalThis`-based caching pattern for Next.js dev mode, with deduplication of concurrent requests
- **React Strict Mode** — AbortController pattern to handle double-mount without deadlocks
- **Error Diagnosis** — Common errors like `Maximum Logins Exceeded`, `login rate exceeded`, 404s, missing `sequenceId`, and domain mismatches

## Installation

```bash
npx skills add dylandersen/agentforce-agent-api
```

This installs the skill for Claude Code. The repo also mirrors the same skill into `.cursor/skills/agentforce-agent-api/` so Cursor can discover it too.

## Repository Layout

The skill is intentionally split into a compact core plus reference files:

- `.claude/skills/agentforce-agent-api/SKILL.md` — primary skill entry point for Claude Code
- `.claude/skills/agentforce-agent-api/*.md` — deep reference material
- `.cursor/skills/agentforce-agent-api/*.md` — mirrored skill files for Cursor

## Usage

Once installed, Claude Code or Cursor can automatically activate this skill when you ask it to:

- Build a chat interface with an Agentforce agent
- Set up Agent API authentication
- Find the correct agent API name or agent ID
- Debug Agent API errors (`Maximum Logins Exceeded`, 404s, missing fields)
- Connect an external app, web app, or mobile app to a Salesforce agent
- Set up or test Enhanced Chat v2, Mobile SDK, or Testing API workflows

### Example Prompts

```
"Set up Agentforce Agent API authentication in my Next.js app"

"Create a chat component that talks to my Salesforce agent"

"I'm getting 'Maximum Logins Exceeded' when calling the Agent API — help me fix it"

"Add token caching for my Salesforce OAuth flow"

"How do I find the API name and agent ID for my Agentforce agent?"
```

## Prerequisites

Before using the Agent API, you'll need:

1. A Salesforce org with Agentforce enabled
2. An External Client App (Connected App) configured with OAuth scopes: `api`, `offline_access`, `chatbot_api`, `sfap_api`
3. Client Credentials flow enabled with a valid Run As user (must have full API login access - **not** an Einstein Agent service user)
4. Your agent API name and Agent ID

## Environment Variables

The skill will guide Claude to set up these variables in your project:

```env
SALESFORCE_MY_DOMAIN=your-org.my.salesforce.com
SALESFORCE_CONSUMER_KEY=your-connected-app-client-id
SALESFORCE_CONSUMER_SECRET=your-connected-app-client-secret
SALESFORCE_AGENT_ID=0XxHu000000XXXXKXX
```

## Reference Docs

The deep reference files live beside the skill and cover the detailed flows:

- `api-reference.md` — Agent API endpoints, session lifecycle, variables, citations, agent ID lookup
- `auth-setup.md` — External Client App setup, token request, caching, and env var handling
- `troubleshooting.md` — Common errors and diagnostic guidance
- `mobile-sdk.md` — React Native, iOS, Android, Enhanced Chat v2, and inline mode
- `testing-api.md` — Testing API, Connect API, custom evaluations, and CLI workflows
- `ecosystem.md` — Agentforce tool map and builder comparison

## License

MIT
