# OpenClaw Agent Factory

A Claude Code skill that creates fully operational [OpenClaw](https://openclaw.ai) AI agents from scratch.

## What it does

Guides you through a 10-phase setup process to create an agent with:

- **Identity** — SOUL.md, USER.md, IDENTITY.md, AGENTS.md (personality, capabilities, rules)
- **Model Auth** — API key configuration for OpenAI, Anthropic, Google, etc.
- **Channels** — Telegram (allowlist) and Slack (pairing) with routing bindings
- **Cortex v2 Memory** — Persistent structured memory with topic files, scoring, and hybrid search
- **Dream Pipeline** — Nightly 12-step consolidation of conversations into long-term memory
- **Self-Improving** — Automatic capture of errors, corrections, and feature requests
- **MCP Integrations** — npm packages + custom MCP servers for any API
- **Browser Automation** — Xvfb + Chromium for web UI operations the API doesn't support

## Install

```bash
claude /plugin install ovictorluiz/openclaw-agent-factory
```

Or manually:

```bash
git clone https://github.com/ovictorluiz/openclaw-agent-factory.git ~/.claude/plugins/openclaw-agent-factory
```

## Usage

```
/openclaw-agent-factory:create-agent
```

Or with an agent name:

```
/openclaw-agent-factory:create-agent archie
```

The skill will guide you interactively through all phases.

## Prerequisites

- Linux server with [OpenClaw](https://openclaw.ai) installed
- Claude Code CLI
- Model API key (OpenAI, Anthropic, etc.)
- Telegram bot token and/or Slack app tokens (for channels)

## What's inside

The skill encodes lessons learned from building production agents:

- 15 critical rules from real setup failures
- Correct auth-profiles.json format (common gotcha: `key` not `apiKey`, `type` not `mode`)
- Telegram allowlist trap (empty `allowFrom` = silent message drop)
- Heartbeat per-agent override (adding to one agent silently disables others)
- Dream cron timezone requirement (without explicit tz, runs in UTC)
- Browser automation stack (Xvfb + headed Chromium + cdpUrl — headless fails for SPAs)
- MCP env var names (always check README — APIs vary)
- API silent field acceptance (some APIs return "success" but don't persist unknown fields)

## License

MIT
