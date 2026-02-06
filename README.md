# Agentforce Agent API Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that teaches Claude how to integrate Salesforce Agentforce Agent API into web applications. Covers OAuth Client Credentials flow, session management, synchronous messaging, token caching, and common debugging pitfalls.

## What This Skill Does

When installed, Claude Code gains deep knowledge of the Agentforce Agent API, including:

- **Authentication** — OAuth Client Credentials flow with Connected App setup, Run As user requirements, and token caching to avoid rate limits
- **Session Management** — Creating, messaging, and ending Agent API sessions with correct endpoint formats (`api.salesforce.com`, not your My Domain)
- **Message Handling** — Proper request/response body structure, including the `sequenceId` placement that trips up most developers
- **Token Caching** — `globalThis`-based caching pattern for Next.js dev mode (HMR-safe), with deduplication of concurrent requests
- **React Strict Mode** — AbortController pattern to handle double-mount without deadlocks
- **Error Diagnosis** — Quick reference for common errors like `Maximum Logins Exceeded`, `login rate exceeded`, 404s, and missing `sequenceId`

## Installation

```bash
npx skills add dylandersen/agentforce-agent-api
```

This installs the skill into your project's `.claude/skills/` directory.

## Usage

Once installed, Claude Code automatically activates this skill when you ask it to:

- Build a chat interface with an Agentforce agent
- Set up Agent API authentication
- Debug Agent API errors (`Maximum Logins Exceeded`, 404s, missing fields)
- Connect an external app (Next.js, React, etc.) to a Salesforce agent

### Example Prompts

```
"Set up Agentforce Agent API authentication in my Next.js app"

"Create a chat component that talks to my Salesforce agent"

"I'm getting 'Maximum Logins Exceeded' when calling the Agent API — help me fix it"

"Add token caching for my Salesforce OAuth flow"
```

## Prerequisites

Before using the Agent API, you'll need:

1. A Salesforce org with Agentforce enabled
2. An External Client App (Connected App) configured with OAuth scopes: `api`, `offline_access`, `chatbot_api`, `sfap_api`
3. Client Credentials flow enabled with a valid Run As user (must have full API login access — **not** an Einstein Agent service user)
4. Your Agent ID (starts with `0Xx`)

## Environment Variables

The skill will guide Claude to set up these variables in your project:

```env
SALESFORCE_MY_DOMAIN=your-org.my.salesforce.com
SALESFORCE_CONSUMER_KEY=your-connected-app-client-id
SALESFORCE_CONSUMER_SECRET=your-connected-app-client-secret
SALESFORCE_AGENT_ID=0XxHu000000XXXXKXX
```

## License

MIT
