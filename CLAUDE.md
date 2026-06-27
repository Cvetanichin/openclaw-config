# CLAUDE.md — openclaw-config

This repository stores the configuration, agent workspace, and generated assets for an **OpenClaw** instance — a self-hosted AI gateway that wraps Claude CLI and exposes a local OpenAI-compatible HTTP API with messaging channel integrations (WhatsApp).

## What is OpenClaw?

OpenClaw is an npm-installed gateway (`openclaw` package) that:
- Runs a local HTTP server on port 18789 with an OpenAI-compatible `/v1/chat/completions` endpoint
- Proxies requests to the Claude CLI (`claude.exe`) as the AI backend
- Connects messaging channels (WhatsApp) and routes messages to a persistent agent session
- Provides a canvas web UI for direct interaction

This repo is **not a software project to build** — it is a config store that the OpenClaw process reads from `C:\Users\cvetanichin\.openclaw\` (or equivalent user data directory).

## Repository Structure

```
openclaw-config/
├── openclaw.json          # Main gateway configuration (model, channels, plugins, hooks)
├── gateway.cmd            # Windows batch script to start the gateway (port 18789)
├── .claude/
│   ├── launch.json        # VS Code / Claude Code launch config for the gateway process
│   └── settings.local.json
├── agents/
│   └── main/
│       └── sessions/      # Claude CLI session files for the main agent
├── canvas/
│   └── index.html         # Canvas web UI for direct chat with the agent
├── completions/           # Shell completion scripts (bash, zsh, fish, PowerShell)
│   ├── openclaw.bash
│   ├── openclaw.zsh
│   ├── openclaw.fish
│   └── openclaw.ps1
├── devices/
│   ├── paired.json        # Registry of paired devices (WhatsApp, etc.)
│   └── pending.json       # Pending device pairings
└── workspace/             # Agent workspace — the files the AI reads each session
    ├── AGENTS.md          # Core agent behavior rules and session protocol
    ├── BOOTSTRAP.md       # First-run onboarding guide (delete after initial setup)
    ├── HEARTBEAT.md       # Periodic check checklist (keep small to limit token usage)
    ├── IDENTITY.md        # Agent name, creature, vibe, emoji (fill in on first run)
    ├── SOUL.md            # Core values and personality guidelines
    ├── TOOLS.md           # Environment-specific tool notes (cameras, SSH, TTS)
    ├── USER.md            # Info about the human (fill in on first run)
    └── .openclaw/
        └── workspace-state.json  # Internal OpenClaw workspace state
```

## Key Configuration: openclaw.json

The main config file controls all gateway behavior. Key sections:

| Section | Purpose |
|---|---|
| `gateway` | Port (18789), bind (loopback), auth token, HTTP endpoints |
| `agents.defaults` | Default model (`claude-cli/claude-sonnet-4-6`), Claude CLI command path, workspace path |
| `channels.whatsapp` | WhatsApp integration — self-chat mode, DM policy (allowlist), allowed numbers |
| `tools` | Tool profile (`coding`), web search provider (Gemini/Google) |
| `plugins` | Google plugin with web search API key |
| `hooks.internal` | session-memory, command-logger, bootstrap-extra-files, boot-md |
| `session` | DM scope: `per-channel-peer` |

### Claude CLI Backend

The gateway invokes Claude CLI for every AI request:

```
# New session
claude.exe -p --output-format json --permission-mode bypassPermissions

# Resume session
claude.exe -p --output-format json --permission-mode bypassPermissions --resume {sessionId}
```

Sessions are stored in `agents/main/sessions/`.

## Agent Workspace Protocol

Every session, the agent reads these files in order before responding:

1. `workspace/SOUL.md` — who the agent is
2. `workspace/USER.md` — who the user is
3. `workspace/memory/YYYY-MM-DD.md` (today + yesterday) — recent context
4. `workspace/MEMORY.md` — long-term curated memory (main session only, not group chats)

### Memory System

- **Daily notes** → `workspace/memory/YYYY-MM-DD.md` — raw session logs
- **Long-term** → `workspace/MEMORY.md` — distilled, curated memories
- `MEMORY.md` must NOT be loaded in group chats or shared sessions (privacy leak risk)
- All memory is file-based — "mental notes" don't survive session restarts

### Heartbeats

The gateway sends periodic heartbeat polls. The agent reads `HEARTBEAT.md` and either:
- Returns `HEARTBEAT_OK` if nothing needs attention
- Acts on any checklist items (check email, calendar, upcoming events)

Keep `HEARTBEAT.md` small to limit token usage per heartbeat.

### Group Chat Behavior

The agent follows these rules in group chats (e.g. WhatsApp groups):
- Only respond when directly mentioned, asked a question, or able to add genuine value
- Stay silent (`HEARTBEAT_OK`) for casual banter or already-answered questions
- Never share the user's private files, memory, or context with third parties
- Use emoji reactions instead of reply messages when acknowledgment suffices

## Starting the Gateway

On Windows, run `gateway.cmd`:

```batch
gateway.cmd
```

This starts the OpenClaw gateway on port 18789 via Node.js. The gateway binary is installed globally at:
```
C:\Users\cvetanichin\AppData\Roaming\npm\node_modules\openclaw\dist\index.js
```

To upgrade OpenClaw: `npm install -g openclaw`

## Git Conventions

- `master` is the default branch
- The `.gitignore` excludes: `credentials/`, `identity/`, `logs/`, `*.sqlite*`, `*.bak*`, `update-check.json`
- Sensitive files (API keys in `openclaw.json`, tokens in `devices/paired.json`) are committed — treat this repo as **private**
- Shell completions in `completions/` are auto-generated by OpenClaw; do not edit manually

## What to Edit vs. What Not To

**Safe to edit freely:**
- `workspace/SOUL.md` — agent personality (inform the user when changed)
- `workspace/USER.md` — user profile info
- `workspace/IDENTITY.md` — agent identity (name, vibe, emoji)
- `workspace/TOOLS.md` — local environment notes
- `workspace/HEARTBEAT.md` — periodic task checklist
- `workspace/memory/*.md` — session logs (agent manages these)

**Edit with care:**
- `openclaw.json` — changes take effect on gateway restart; invalid JSON or wrong values break connectivity
- `devices/paired.json` — modifying can break existing device pairings

**Never edit manually:**
- `completions/` — auto-generated by the `openclaw` CLI
- `agents/main/sessions/` — live session state managed by OpenClaw
- `workspace/.openclaw/` — internal state managed by OpenClaw
