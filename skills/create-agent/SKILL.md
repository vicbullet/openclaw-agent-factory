---
description: Create a new OpenClaw agent end-to-end on any machine — guided setup for identity, model, auth, channels, Cortex v2 memory, Dream Pipeline, and browser. Use when someone wants to add a new agent to their OpenClaw gateway.
allowed-tools:
  - Bash(openclaw *)
  - Bash(systemctl --user *)
  - Bash(journalctl --user *)
  - Bash(python3 *)
  - Bash(chmod *)
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(cat *)
  - Bash(grep *)
  - Bash(curl *)
  - Bash(cp *)
  - Bash(df *)
  - Bash(hostname *)
  - Bash(date *)
  - Bash(tail *)
  - Bash(node *)
  - Bash(npm *)
  - Bash(npx *)
  - Bash(snap *)
  - Bash(apt *)
  - Bash(kill *)
  - Bash(pkill *)
  - Bash(rm /tmp/*)
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - WebFetch
---

# /create-openclaw-agent — OpenClaw Agent Factory

Create a fully operational OpenClaw AI agent from scratch. This skill walks through every step — from prerequisites to a working agent with persistent memory, messaging channels, scheduled Dream consolidation, and browser access.

**Works on any Linux server with OpenClaw installed.**

Arguments: `$ARGUMENTS`

---

## HOW THIS WORKS

This is a guided, step-by-step process. Each phase has verification steps — we don't advance until the current phase is confirmed working.

**Phases:**
0. Prerequisites Check
1. Gather Requirements
2. Create Agent + Model Auth
3. Channel Setup (Telegram + Slack)
4. Workspace Identity Files
5. Cortex v2 Memory System
6. Memory Search (Voyage AI) + MCP Integrations
7. Browser Setup (optional)
8. Dream Cron
9. Test & Verify
10. Seed Content (optional)

---

## CRITICAL RULES

1. **MERGE, never overwrite** — when editing openclaw.json, ADD/UPDATE keys without deleting existing content. NEVER remove `gateway.mode`.
2. **Absolute paths** in JSON configs — use full paths like `/home/user/.openclaw/...`, not `~/.openclaw/...`.
3. **auth-profiles.json format** — `{"version":1,"profiles":{"<provider>":{"type":"api_key","provider":"<provider>","key":"<key>"}}}`. Must use `type` (not `mode`) and `key` (not `apiKey`).
4. **Replace default workspace files** — OpenClaw creates generic SOUL.md, USER.md, IDENTITY.md, AGENTS.md on first boot. These are useless placeholders. ALWAYS replace with actual content.
5. **MCP env var names** — ALWAYS check package README. Common traps: `LINEAR_ACCESS_TOKEN` (not API_KEY), `SLITE_API_KEY` (not API_TOKEN).
6. **Test after each phase** — confirm success before proceeding.
7. **Back up before editing config** — always `cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak`.
8. **Validate after config edits** — run `openclaw doctor` BEFORE restarting gateway.
9. **systemd user service** — gateway runs as user unit. Use `systemctl --user` (never `sudo systemctl`).
10. **Heartbeat per-agent overrides global** — Adding `heartbeat` to ANY agent in `agents.list` disables the global default for ALL agents. When enabling heartbeat for a new agent, add explicit `heartbeat` to every existing agent too.
11. **Dream cron timezone is NOT inherited** — Each cron needs its own timezone. Without it, runs in UTC.
12. **Telegram allowFrom is REQUIRED** — `dmPolicy: "allowlist"` with empty `allowFrom` silently drops ALL messages. Always include user's numeric Telegram ID.
13. **Gateway restart command** — use `openclaw daemon restart`. If that fails, use `systemctl --user restart openclaw-gateway`.
14. **File permissions** — openclaw.json: 600, auth-profiles.json: 600, directories: 700.
15. **External APIs may accept unknown fields silently** — always verify with Swagger/docs that a field actually works before building a tool around it.

---

## PHASE 0: Prerequisites Check

### 0A: Confirm you're on the server

```bash
hostname
```

Must be the OpenClaw server, NOT your laptop.

### 0B: OpenClaw installed?

```bash
openclaw --version
```

If not: follow https://docs.openclaw.ai/getting-started first.

### 0C: Gateway running?

```bash
systemctl --user status openclaw-gateway
```

Expected: `active (running)`. If not:
- Service doesn't exist → `openclaw onboard --install-daemon`
- Service stopped → `systemctl --user start openclaw-gateway`

### 0D: Config valid?

```bash
openclaw doctor
```

Fix critical errors before proceeding.

### 0E: Existing agents and crons

```bash
openclaw agents list --bindings
openclaw cron list
```

Record current state — needed to avoid ID conflicts and cron time collisions.

### 0F: Disk space

```bash
df -h ~
```

**Only proceed after ALL checks pass.**

---

## PHASE 1: Gather Requirements

Ask for these. Use sensible defaults for anything not provided.

| Question | Example | Default |
|----------|---------|---------|
| **Agent ID** (short, lowercase) | `archie` | *(required)* |
| **Display name** | Archie | = ID capitalized |
| **Role / personality** | CTO, technical operations | *(required)* |
| **Model provider** | `openai-codex`, `openai`, `anthropic`, `google` | `openai-codex` |
| **Model name** | `gpt-5.4`, `claude-opus-4-6` | `gpt-5.4` |
| **API key** | the key itself | *(required)* |
| **Channels** | telegram, slack, both, none | both |
| **Telegram bot token** | from @BotFather | *(if telegram)* |
| **Your Telegram numeric ID** | `123456789` | *(if telegram)* |
| **Slack app token** | `xapp-...` | *(if slack)* |
| **Slack bot token** | `xoxb-...` | *(if slack)* |
| **Timezone** | `America/Sao_Paulo` | `UTC` |
| **MCP integrations** | Linear, Slite, Gmail, custom | none |
| **Voyage AI key** (for memory search) | key from dash.voyageai.com | *(optional)* |
| **Browser needed?** | yes/no | no |

---

## PHASE 2: Create Agent + Model Auth

### 2A: Register agent

```bash
openclaw agents add <AGENT_ID>
```

### 2B: Create directory structure

```bash
mkdir -p ~/.openclaw/agents/<AGENT_ID>/agent
```

### 2C: Configure model auth

Write `~/.openclaw/agents/<AGENT_ID>/agent/auth-profiles.json`:

```json
{
  "version": 1,
  "profiles": {
    "<provider>": {
      "type": "api_key",
      "provider": "<provider>",
      "key": "<YOUR_API_KEY>"
    }
  }
}
```

```bash
chmod 600 ~/.openclaw/agents/<AGENT_ID>/agent/auth-profiles.json
```

### 2D: Add to openclaw.json

**Back up first:**
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
```

**MERGE** into existing `agents.list` array (preserve ALL existing entries):

```json
{
  "id": "<AGENT_ID>",
  "workspace": "<ABSOLUTE_PATH>/.openclaw/agents/<AGENT_ID>/workspace",
  "agentDir": "<ABSOLUTE_PATH>/.openclaw/agents/<AGENT_ID>",
  "model": "<provider>/<model>"
}
```

### 2E: Validate

```bash
openclaw doctor
openclaw agents list --bindings
```

---

## PHASE 3: Channel Setup

### 3A: Telegram

**User creates bot (manual step):**
1. Open @BotFather on Telegram
2. `/newbot` → set name and username
3. Copy the bot token

**Add to openclaw.json** — MERGE into `channels.telegram.accounts`:

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "<AGENT_ID>": {
          "botToken": "<BOT_TOKEN>",
          "dmPolicy": "allowlist",
          "allowFrom": [<USER_TELEGRAM_NUMERIC_ID>]
        }
      }
    }
  }
}
```

**CRITICAL:** `allowFrom` MUST contain at least one numeric Telegram ID. Empty `allowFrom` with `allowlist` policy = bot silently ignores ALL messages. Looks "connected" but never responds — extremely hard to debug.

For secrets management: use 1Password SecretRef or env vars instead of plaintext. Format:
```json
"botToken": { "source": "exec", "provider": "op", "id": "VaultName/<ITEM_UUID>/password" }
```

**Telegram commands** (optional) — MERGE at `channels.telegram` level (NOT inside accounts):

```json
{
  "channels": {
    "telegram": {
      "commands": {
        "native": true,
        "nativeSkills": true
      },
      "customCommands": [
        { "command": "status", "description": "Agent status check" }
      ]
    }
  }
}
```

### 3B: Slack

**User creates app (manual step at api.slack.com):**
1. Create New App > From Manifest
2. Add scopes: `chat:write`, `channels:read`, `groups:read`, `im:read`, `im:write`, `im:history`, `mpim:read`, `users:read`
3. Enable Socket Mode → get App Token (`xapp-...`)
4. Install to Workspace → get Bot Token (`xoxb-...`)

**Add to openclaw.json** — MERGE into `channels.slack.accounts`:

```json
{
  "channels": {
    "slack": {
      "accounts": {
        "<AGENT_ID>": {
          "appToken": "<XAPP_TOKEN>",
          "botToken": "<XOXB_TOKEN>",
          "dmPolicy": "pairing"
        }
      }
    }
  }
}
```

### 3C: Routing bindings

MERGE into `bindings` array:

```json
[
  { "agentId": "<AGENT_ID>", "match": { "channel": "telegram", "accountId": "<AGENT_ID>" } },
  { "agentId": "<AGENT_ID>", "match": { "channel": "slack", "accountId": "<AGENT_ID>" } }
]
```

### 3D: Validate and restart

```bash
openclaw doctor
openclaw daemon restart
```

Wait, then verify:

```bash
systemctl --user status openclaw-gateway
openclaw agents list --bindings
```

**If restart fails:**
1. `openclaw doctor` → fix config errors
2. `tail -30 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log`
3. Restore backup: `cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json && openclaw daemon restart`

---

## PHASE 4: Workspace Identity Files

Replace default placeholders. All files go in `~/.openclaw/agents/<AGENT_ID>/workspace/`:

### SOUL.md — Who the agent IS

```markdown
# SOUL — <Display Name>

## Identity
<Who the agent is, role, organization>

## Personality
<Communication style, tone, values — shapes every response>

## Principles
<Core operating principles — what the agent always/never does>

## Behaviors
<Specific behavioral rules>
```

### USER.md — Who the agent SERVES

```markdown
# USER — <User Name>

## Profile
<Role, responsibilities, context>

## Preferences
<Communication style, formats, timing>

## Team
<People the user works with>
```

### IDENTITY.md — What the agent CAN DO

```markdown
# IDENTITY — <Display Name>

## Role
<Specific responsibilities>

## Capabilities
<Tools, integrations, access>

## Boundaries
<What it should NOT do without asking>

## Team
<Other agents, how they interact>
```

### AGENTS.md — Operating Rules

```markdown
# AGENTS — <Display Name> Operating Rules

## Every Session
1. Read SOUL.md (who I am)
2. Read USER.md (who I serve)
3. Read memory/ (recent notes)
4. In private sessions: read MEMORY.md
Without asking. Just do it.

## What I Can Do Freely
- <autonomous actions list>

## Ask Before
- <actions requiring approval>

## Escalate To User
- <situations needing human judgment>

## Participation in Groups
### When to respond
- Mentioned directly
- Questions in my domain
- <domain-specific triggers>

### When to stay quiet
- Casual conversation between humans
- Topics outside my domain
- Another agent already answered

## Security
- Never leak confidential data
- Never execute destructive actions without confirmation
- Never send external messages without approval
```

---

## PHASE 5: Cortex v2 Memory System

Cortex gives agents persistent memory across sessions with automatic nightly consolidation.

### What is Cortex v2

| Concept | Description |
|---------|-------------|
| **Topic Files** | Structured files: decisions.md, lessons.md, preferences.md, people.md |
| **Dream Pipeline** | Nightly cron (12 steps) consolidates transcripts into topic files |
| **Memory Scoring** | Entries gain/lose relevance: decay 0.997/day, gain +0.15/hit, archive if <0.3 |
| **Self-Improving** | `.learnings/` captures errors and corrections automatically |
| **Hybrid Search** | BM25 + Vector (Voyage AI) via `memory_search` |
| **3 Context Layers** | Shared (all agents) > Factory (per agent) > Domain (specific) |

### 5A: Directory structure

```bash
WORKSPACE=~/.openclaw/agents/<AGENT_ID>/workspace
mkdir -p $WORKSPACE/memory/context
mkdir -p $WORKSPACE/memory/projects
mkdir -p $WORKSPACE/memory/sessions
mkdir -p $WORKSPACE/memory/integrations
mkdir -p $WORKSPACE/.learnings
```

Shared workspace (create ONCE, all agents share):
```bash
mkdir -p ~/.openclaw/workspace-shared/context
mkdir -p ~/.openclaw/workspace-shared/scripts
```

### 5B: Factory topic files

Create in `$WORKSPACE/memory/context/`:

**decisions.md** — Domain decisions with scoring:
```markdown
# Decisions — <Agent Name>

> Domain decisions and rules. Filled by Dream or explicit save.
> Format: title + scoring metadata + content.
```

**lessons.md** — Errors and learnings:
```markdown
# Lessons — <Agent Name>

> Lessons learned. Source: Dream Pipeline + promoted from .learnings/.
```

**preferences.md** — User preferences:
```markdown
# Preferences — <Agent Name>

> User preferences (record after validation, don't infer).
```

**people.md** — Domain contacts:
```markdown
# People & Contacts — <Agent Name>

> People in this agent's domain. Universal catalog at workspace-shared/context/contacts.md.
```

Every entry with scoring uses:
```markdown
### [title]
- added: YYYY-MM-DD
- score: 1.0
- last_hit: YYYY-MM-DD

[content]
```

### 5C: State files

**pending.md** at `$WORKSPACE/memory/pending.md`:
```markdown
# Pending Items

> Items awaiting input, access, or decision.
> Agent reminds user at session boot.

## Awaiting User

## Awaiting Third Parties
```

### 5D: MEMORY.md (enriched index)

Create at `$WORKSPACE/MEMORY.md` (workspace root):

```markdown
# MEMORY.md — <Agent Name> Cortex Index

> Last consolidation: none (initial setup)

## Context (boot)
- memory/context/decisions.md (0 entries)
  > [empty — will be filled by Dream]
  > Boot: always loaded

- memory/context/preferences.md (0 entries)
  > [empty — will be filled by Dream]
  > Boot: always loaded

- memory/context/lessons.md (0 entries)
  > [empty — will be filled by Dream]
  > Boot: always loaded

- memory/pending.md (0 items)
  > Boot: always loaded

## Context (on-demand)
- memory/context/people.md (0 entries)
  > Search: when someone is mentioned

## Projects
[Dream adds as detected in transcripts]

## Integrations
[Dream adds as tools are configured]

## Sessions (history)
[Dream generates daily summaries: memory/sessions/YYYY-MM-DD.md]
```

### 5E: Self-improving capture

Create in `$WORKSPACE/.learnings/`:

**LEARNINGS.md:**
```markdown
# Learnings
> Dream promotes entries with Recurrence >= 3 to lessons.md.
> Format: LRN-YYYYMMDD-XXX with Pattern-Key, Recurrence-Count, Category.
```

**ERRORS.md:**
```markdown
# Errors
> Command failures. Dream promotes recurring patterns.
> Format: ERR-YYYYMMDD-XXX with Pattern-Key, Command, Error, Fix.
```

**FEATURE_REQUESTS.md:**
```markdown
# Feature Requests
> Missing capabilities. Format: FEAT-YYYYMMDD-XXX with Status, Context.
```

### 5F: ROUTING.md

Create at `$WORKSPACE/ROUTING.md`:

```markdown
# ROUTING.md — <Agent Name>

> Agent-specific routing delta. Applied AFTER the universal DREAM.md table.

## Additional routing
| Pattern | Destination |
|---------|-------------|
| [domain patterns] | [destination file] |

## What to ignore
- [other agents' domains]

## Domain rules
- [specific rules]
```

### 5G: Shared files (create ONCE)

Skip any that already exist.

**DREAM.md** at `~/.openclaw/workspace-shared/DREAM.md` — the 12-step Dream Pipeline:

1. Read ROUTING.md (agent's local delta)
2. Extract transcripts via transcript-extract.py
3. Analyze for persistable information
4. Classify and route (universal table + ROUTING.md)
5. Consolidate (dedup, merge > create, resolve contradictions, absolute dates, no secrets)
6. Daily summary → memory/sessions/YYYY-MM-DD.md
7. Update MEMORY.md (counts, summaries, boot/on-demand)
8. Self-Learning (promote .learnings/ Recurrence >= 3)
9. Memory Scoring (hit +0.15, miss x0.997, archive <0.3 + 60 days)
10. Stats → stats.jsonl
11. Prune (stale pointers, >15k chars alert, clean promoted)
12. Report (DREAM_REPORT: agent, duration, chars, items, archived, alerts)

**Universal routing table:**

| Pattern | Destination |
|---------|-------------|
| Person with context | people.md (local) + contacts.md (shared) |
| Decision or rule | decisions.md |
| Lesson or correction | lessons.md |
| Validated preference | preferences.md |
| Project with status | projects/[name].md |
| Pending created/resolved | pending.md |
| Tool configured | integrations/[name].md |

**transcript-extract.py** at `~/.openclaw/workspace-shared/scripts/`:
Python script extracting messages from session JSONLs. Filters metadata, small talk, duplicates, heartbeat.

**contacts.md** at `~/.openclaw/workspace-shared/context/`:
Universal catalog of people known across all agents.

### 5H: Add Cortex rules to AGENTS.md

Append to the existing AGENTS.md:

```markdown
## Cortex: Memory System

Cortex is this agent's memory system. "dream" or "consolidate memory" triggers the Dream Pipeline.

**Day:** Work normally. Zero overhead.
- If user asks to save ("save", "record", "store"), write directly to the topic file.
- Search memory (memory_search) before answering questions about the past.
- Decisions → decisions.md, Preferences → preferences.md, Lessons → lessons.md

**Night:** Dream Pipeline runs automatically.
- Reads all transcripts since last checkpoint
- Classifies and consolidates into topic files per DREAM.md + ROUTING.md
- Zero action needed from agent

**Boot:** At session start:
- Read .learnings/ for pending patterns
- Check memory/pending.md and remind user of open items

### Self-Improving
- Command failures → .learnings/ERRORS.md (Pattern-Key + Recurrence-Count)
- User corrections → .learnings/LEARNINGS.md (Category: correction/knowledge_gap/best_practice)
- Missing capabilities → .learnings/FEATURE_REQUESTS.md
- Dream step 8 promotes patterns with Recurrence >= 3 to lessons.md
```

---

## PHASE 6: Memory Search + MCP Integrations

### 6A: Memory search config

MERGE into the agent's entry in `agents.list`:

```json
{
  "memorySearch": {
    "sources": ["memory", "sessions"],
    "extraPaths": ["<ABSOLUTE_PATH>/.openclaw/workspace-shared/context"],
    "provider": "voyage",
    "remote": { "apiKey": "<VOYAGE_API_KEY>" },
    "query": {
      "hybrid": {
        "enabled": true,
        "vectorWeight": 0.5,
        "textWeight": 0.5
      }
    }
  }
}
```

**Voyage AI:** Free tier 200M tokens. Needs payment method added for usable rate limits (3 RPM without → standard with).

### 6B: MCP servers (global, shared by all agents)

MERGE into `mcp.servers`:

```json
{
  "mcp": {
    "servers": {
      "<name>": {
        "command": "npx",
        "args": ["-y", "<package>"],
        "env": { "<EXACT_ENV_VAR_NAME>": "<value>" }
      }
    }
  }
}
```

**CRITICAL:** SecretRef objects are NOT supported in `mcp.servers.*.env` — must be plaintext strings. File is chmod 600 so this is acceptable.

**Tested packages:**

| Package | Env var (EXACT) | Notes |
|---------|----------------|-------|
| `mcp-server-linear` | `LINEAR_ACCESS_TOKEN` | NOT LINEAR_API_KEY |
| `slite-mcp-server` | `SLITE_API_KEY` | NOT SLITE_API_TOKEN |
| Gmail | file-based OAuth | `--creds-file-path` + `--token-path` args |

### 6C: Custom MCP servers

For services without npm packages:

1. `mkdir -p ~/.openclaw/mcp-servers/<name>/`
2. `cd` there → `npm init -y && npm install @modelcontextprotocol/sdk`
3. Set `"type": "module"` in package.json
4. Write `index.mjs` using `McpServer` + `StdioServerTransport`
5. **Test before deploying:**
```bash
printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}\n{"jsonrpc":"2.0","method":"notifications/initialized"}\n{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}\n' | ENV_VAR=val node index.mjs 2>/dev/null
```
6. Add to openclaw.json: `"command": "node"`, `"args": ["/absolute/path/index.mjs"]`

**WARNING:** Some APIs accept unknown fields silently (return "success" but don't persist). Always verify with Swagger docs AND test that written data appears in read responses.

### 6D: Index memory

```bash
openclaw memory index --force --agent <AGENT_ID>
```

---

## PHASE 7: Browser Setup (optional)

If the agent needs web UI access (e.g., services without API support for all operations).

### 7A: Install prerequisites

```bash
sudo apt-get install -y xvfb
sudo snap install chromium
```

### 7B: Create systemd services

**Xvfb** at `~/.config/systemd/user/xvfb.service`:
```ini
[Unit]
Description=Xvfb Virtual Frame Buffer

[Service]
Type=simple
ExecStart=/usr/bin/Xvfb :99 -screen 0 1920x1080x24 -nolisten tcp
Restart=on-failure
RestartSec=3

[Install]
WantedBy=default.target
```

**Chromium** at `~/.config/systemd/user/chromium-headless.service`:
```ini
[Unit]
Description=Chromium CDP (headed on Xvfb)
After=xvfb.service
Requires=xvfb.service

[Service]
Type=simple
Environment=DISPLAY=:99
ExecStart=/snap/bin/chromium --no-sandbox --disable-gpu --disable-dev-shm-usage --remote-debugging-port=9222 --remote-debugging-address=127.0.0.1 --remote-allow-origins=* --user-data-dir=%h/.config/chromium-headless --window-size=1920,1080 about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

### 7C: Start services

```bash
systemctl --user daemon-reload
systemctl --user enable xvfb chromium-headless
systemctl --user start xvfb chromium-headless
```

Verify:
```bash
curl -s http://127.0.0.1:9222/json/version
```

### 7D: Configure OpenClaw

MERGE into openclaw.json:
```json
{
  "browser": {
    "enabled": true,
    "cdpUrl": "http://127.0.0.1:9222"
  }
}
```

**CRITICAL lessons (from production):**
- Snap Chromium MUST run **headed on Xvfb** — headless fails for interactive SPA operations (uploads, dropdowns)
- OpenClaw MUST use `cdpUrl` to external Chromium — managed mode fails with snap sandbox
- Don't use `attachOnly: true` — reads work but clicks timeout
- If SingletonLock error: `rm -f ~/snap/chromium/common/chromium/SingletonLock ~/.config/chromium-headless/SingletonLock && pkill -9 -f chromium && systemctl --user restart chromium-headless`

### 7E: Web service login (if needed)

Store credentials:
```bash
echo '{"email":"<email>","password":"<pass>"}' > ~/.openclaw/<service>-credentials.json
chmod 600 ~/.openclaw/<service>-credentials.json
```

Create a login script using CDP WebSocket to navigate and fill forms. Session persists in `~/.config/chromium-headless/` between restarts.

---

## PHASE 8: Dream Cron

### 8A: Check existing crons

```bash
openclaw cron list
```

### 8B: Calculate schedule

Space agents 10 minutes apart:
- Agent 1: 03:00
- Agent 2: 03:10
- Agent 3: 03:20
- Consolidated Report: 03:50

### 8C: Create Dream cron

```bash
openclaw cron create \
  --name "Dream <AGENT_ID>" \
  --schedule "cron <MINUTE> 3 * * * @ <TIMEZONE>" \
  --agent <AGENT_ID> \
  --target isolated \
  --model anthropic/claude-opus-4-6 \
  --timeout 600 \
  --prompt "You are the Dream Pipeline. Read ~/.openclaw/workspace-shared/DREAM.md and execute all instructions. Your agent id is <AGENT_ID>."
```

**CRITICAL:** Each cron needs its own timezone in the schedule (`@ America/Sao_Paulo`). Without it, runs in UTC.

### 8D: Report consolidator (create once)

```bash
openclaw cron create \
  --name "Dream Report" \
  --schedule "cron 50 3 * * * @ <TIMEZONE>" \
  --agent <FIRST_AGENT_ID> \
  --target isolated \
  --model anthropic/claude-sonnet-4-6 \
  --timeout 120 \
  --prompt "Read Dream runs from the last 2 hours for all agents. Build a single DREAM_REPORT and send to user."
```

---

## PHASE 9: Test & Verify

### Infrastructure
- [ ] Gateway running (`systemctl --user status openclaw-gateway`)
- [ ] Agent registered (`openclaw agents list --bindings`)
- [ ] auth-profiles.json correct format + chmod 600
- [ ] Config valid (`openclaw doctor`)

### Channels
- [ ] Telegram connected (if configured)
- [ ] Slack connected (if configured)
- [ ] Bindings route correctly
- [ ] DM policy works (send test message)
- [ ] `allowFrom` includes your Telegram ID

### Workspace
- [ ] SOUL.md — actual personality (NOT placeholder)
- [ ] USER.md — actual user profile
- [ ] IDENTITY.md — actual capabilities
- [ ] AGENTS.md — operating rules + Cortex section

### Cortex
- [ ] Directories: memory/context, projects, sessions, integrations, .learnings
- [ ] 4 topic files (decisions, lessons, preferences, people)
- [ ] pending.md
- [ ] MEMORY.md at workspace root
- [ ] .learnings/ with 3 files
- [ ] ROUTING.md
- [ ] Shared files (DREAM.md, transcript-extract.py, contacts.md)

### Memory & MCP
- [ ] Memory indexed (`openclaw memory index --force`)
- [ ] Voyage AI configured (if used)
- [ ] MCP servers accessible

### Dream
- [ ] Dream cron configured
- [ ] Report consolidator configured

### Browser (if configured)
- [ ] Xvfb running
- [ ] Chromium running with CDP
- [ ] OpenClaw browser plugin loaded
- [ ] Web service logged in (if needed)

### Live test
- [ ] Send DM → agent responds with correct personality
- [ ] Ask about something in memory → uses memory_search
- [ ] Ask to save a decision → writes to decisions.md with scoring

---

## PHASE 10: Seed Content (optional)

Inject initial content into empty topic files:

1. Fill people.md with known contacts
2. Update shared/contacts.md (ADD, never overwrite)
3. Create project files in memory/projects/
4. Create integration files in memory/integrations/
5. Inject initial decisions with scoring metadata
6. Update MEMORY.md with counts and summaries
7. Reindex: `openclaw memory index --force --agent <AGENT_ID>`

---

## MEMORY SCORING REFERENCE

| Parameter | Value |
|-----------|-------|
| Decay | x0.997/day (~0.3%/day, 1 year unused → score 0.33) |
| Gain | +0.15 per hit (1 hit recovers ~50 days of decay) |
| Archive | score < 0.3 AND last_hit > 60 days |
| Cap | 1.0 |
| Applied to | decisions.md, preferences.md, lessons.md |
| NOT applied to | people.md, pending.md |

---

## QUICK REFERENCE: File Map

```
~/.openclaw/
├── openclaw.json                          ← gateway config (MERGE only, chmod 600)
├── workspace-shared/                      ← shared across all agents
│   ├── DREAM.md                           ← universal Dream Pipeline (12 steps)
│   ├── context/
│   │   └── contacts.md, user.md, etc.
│   └── scripts/
│       └── transcript-extract.py
├── agents/<AGENT_ID>/
│   ├── agent/
│   │   └── auth-profiles.json             ← model API key (chmod 600)
│   └── workspace/
│       ├── SOUL.md                        ← personality
│       ├── USER.md                        ← user profile
│       ├── IDENTITY.md                    ← capabilities
│       ├── AGENTS.md                      ← rules + Cortex
│       ├── MEMORY.md                      ← Cortex index (boot)
│       ├── ROUTING.md                     ← Dream routing delta
│       ├── .learnings/
│       │   ├── LEARNINGS.md, ERRORS.md, FEATURE_REQUESTS.md
│       └── memory/
│           ├── context/
│           │   ├── decisions.md, lessons.md, preferences.md, people.md
│           ├── projects/
│           ├── sessions/                  ← Dream output (YYYY-MM-DD.md)
│           ├── integrations/
│           └── pending.md
├── mcp-servers/                           ← custom MCP servers
│   └── <name>/
│       ├── index.mjs
│       └── package.json
└── <service>-credentials.json             ← web service creds (chmod 600)
```

---

## ANTI-PATTERNS

| Don't | Do instead |
|-------|-----------|
| Overwrite openclaw.json | MERGE — preserve all existing content |
| Remove `gateway.mode` | NEVER — gateway won't start |
| Use `~` in JSON configs | Absolute paths only |
| Use `apiKey` in auth-profiles | Use `key` (and `type`, not `mode`) |
| Skip replacing default workspace files | ALWAYS replace with actual content |
| Create Dream cron without timezone | Always include timezone in schedule |
| Add heartbeat to one agent silently | Add to ALL agents or none |
| Guess MCP env var names | Check package README first |
| Trust API "Stored" responses | Verify data was actually persisted |
| Run Chromium headless for SPA | Use headed on Xvfb |
| Let OpenClaw manage snap Chromium | Use external service + cdpUrl |
| Deploy MCP tools without testing | Test with protocol pipe before restart |
| Skip `openclaw doctor` before restart | Always validate config first |
| Skip backup before config edits | Always `cp openclaw.json openclaw.json.bak` |
| Set Telegram allowlist without IDs | Always include numeric Telegram ID in allowFrom |
| Use `operationId` for API detail queries | Check exact param name in Swagger (often just `id`) |
