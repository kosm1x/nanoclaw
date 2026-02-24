# NanoClaw User Manual

NanoClaw is a personal AI assistant that runs Claude agents securely in isolated containers, connected to your phone via WhatsApp (and optionally Telegram, Discord, and more). For the project philosophy, see [README.md](README.md). For the full security model, see [docs/SECURITY.md](docs/SECURITY.md). This manual covers everything from setup to daily use to advanced configuration.

## Table of Contents

- [Requirements & Setup](#requirements--setup)
- [Getting Started — Your First Conversation](#getting-started--your-first-conversation)
- [Groups](#groups)
- [Memory & Personality](#memory--personality)
- [Scheduled Tasks](#scheduled-tasks)
- [Customization](#customization)
- [What the Agent Can Do](#what-the-agent-can-do)
- [Configuration Reference](#configuration-reference)
- [Service Management](#service-management)
- [Directory Structure](#directory-structure)
- [Troubleshooting](#troubleshooting)
- [Security Overview](#security-overview)
- [FAQ](#faq)

---

## Requirements & Setup

### Requirements

- **macOS** or **Linux**
- **Node.js 20+**
- **[Claude Code](https://claude.ai/download)** — the CLI for Claude
- **[Docker](https://docker.com/products/docker-desktop)** (macOS or Linux) or **[Apple Container](https://github.com/apple/container)** (macOS only)

### Quick Start

```bash
git clone https://github.com/qwibitai/NanoClaw.git
cd NanoClaw
claude
```

Then run `/setup`. Claude Code handles everything:

1. **Dependencies** — installs Node.js packages
2. **Container build** — builds the agent container image (`nanoclaw-agent:latest`)
3. **WhatsApp authentication** — guides you through QR code scanning
4. **Service configuration** — installs NanoClaw as a background service

### WhatsApp Authentication

During setup, you'll be shown a QR code in your terminal. Open WhatsApp on your phone, go to **Linked Devices**, and scan the code. The session is saved to `store/auth/` and persists across restarts.

If your session expires (WhatsApp disconnects the linked device), NanoClaw exits and shows:

```
WhatsApp authentication required. Run /setup in Claude Code.
```

To re-authenticate:

```bash
npm run auth
```

Then scan the QR code again.

### Verifying It's Running

**macOS (launchd):**

```bash
launchctl list | grep nanoclaw
# A PID in the first column means it's running
# "-" means it's not running
# Non-zero exit code means it crashed
```

**Linux (systemd):**

```bash
systemctl --user status nanoclaw
```

---

## Getting Started — Your First Conversation

### The Main Channel

Your **main channel** is your WhatsApp self-chat (Messages with yourself). This is your private admin control channel. Every message you send here is processed automatically — no trigger word needed.

From the main channel you can:
- Talk to your assistant directly
- Register and manage groups
- Schedule tasks across any group
- View system status

### The Trigger Word

The default trigger word is **`@Andy`** (configurable). In group chats, messages must start with `@Andy` to wake the assistant. In your main channel, every message is processed automatically.

### What Happens When You Send a Message

1. WhatsApp delivers your message to NanoClaw via the Baileys library
2. The message is stored in SQLite
3. The polling loop (~2 seconds) picks it up
4. A container is spawned for your group (or reused if one is already running)
5. Claude Agent SDK processes your message inside the container
6. The response is sent back to your chat

While the agent is working, you'll see a **typing indicator** ("Andy is typing...") in your chat.

### Example Conversation

```
You:   @Andy what's the weather in San Francisco?
Andy:  *San Francisco Weather*
       Currently 62°F (17°C) with partly cloudy skies.
       High of 65°F expected today with light winds from the west.

You:   @Andy remind me to call the dentist tomorrow at 9am
Andy:  Done! I've scheduled a reminder for tomorrow at 9:00 AM.
       I'll message you here when it's time.

You:   @Andy search for the latest news about AI regulation in the EU
Andy:  Here's a summary of the latest EU AI regulation developments...
```

---

## Groups

### What Is a Group?

A group is an isolated context for your assistant. Each group has:
- Its own **memory** (`CLAUDE.md` file)
- Its own **filesystem** (mounted into its container)
- Its own **conversation history** (session)
- Its own **container sandbox** — groups cannot see each other's files

### Main Group vs Registered Groups

| Capability | Main Group | Other Groups |
|------------|------------|--------------|
| Trigger required | No — all messages processed | Yes — must start with `@Andy` |
| Register/remove groups | Yes | No |
| Schedule tasks for other groups | Yes | No |
| Send messages to other chats | Yes | No |
| View all tasks | Yes | Own tasks only |
| Access project root | Yes (read-only) | No |
| Access global memory | Yes (implicit) | Yes (read-only) |

### Registering a Group

From your main channel, tell the assistant to join a group:

```
@Andy join the Family Chat group
```

The assistant will:
1. Look up the group in WhatsApp
2. Register it with a folder name (e.g., `family-chat`)
3. Create the group folder at `groups/family-chat/`
4. Optionally create an initial `CLAUDE.md` for the group

You can also specify additional options:

```
@Andy register the Dev Team group with read-write access to ~/projects/webapp
```

This mounts the specified directory into that group's container at `/workspace/extra/webapp`.

### Trigger Behavior

- **Main group**: Always on. Every message is processed.
- **Registered groups** (default): Messages must start with `@Andy` (or your configured trigger word). Non-trigger messages are still saved — they accumulate as context and are included when a trigger message eventually arrives.
- **Solo chats** (`requiresTrigger: false`): All messages are processed, like the main group. Use this for 1-on-1 chats where you don't want to type the trigger prefix every time.

### Message Accumulation

When you send messages without the trigger word in a group, they aren't lost. They're stored in the database and accumulate. When someone eventually sends a message with `@Andy`, all the accumulated messages since the last agent run are sent as context. This means the agent sees the full conversation, not just the trigger message.

### Group Folders

Each group's files live at `groups/{folder-name}/`:

```
groups/
├── main/           # Your main channel
│   ├── CLAUDE.md   # Main group memory/personality
│   ├── logs/       # Container run logs
│   └── conversations/  # Archived conversation history
├── family-chat/    # A registered group
│   ├── CLAUDE.md
│   └── logs/
└── global/         # Shared read-only memory
    └── CLAUDE.md
```

---

## Memory & Personality

### Per-Group Memory (`CLAUDE.md`)

Each group has a `CLAUDE.md` file at `groups/{folder}/CLAUDE.md`. This is the agent's memory and personality for that group. The agent reads it at the start of every invocation and can update it to remember things.

You can customize the agent's behavior by telling it what to remember:

```
@Andy remember that I prefer concise responses with bullet points
@Andy remember that our team standup is at 9:30am Pacific
@Andy from now on, always include action items at the end of summaries
```

The agent writes these preferences into its `CLAUDE.md` so they persist across sessions.

### Global Memory

The file at `groups/global/CLAUDE.md` is shared across all groups as read-only. It contains personality and instructions that apply everywhere (formatting rules, general behavior, etc.).

Only the main group can edit global memory. Non-main groups can read it but not modify it. To update global memory:

```
@Andy remember globally that my timezone is US/Pacific
```

### Conversation Archive

Each group has a `conversations/` folder that contains searchable history of past conversations. When sessions are compacted (to manage context length), the conversation history is archived here. The agent can search these archives to recall context from previous sessions.

### Structured Data Files

For persistent information, the agent creates structured files in its group folder:

```
@Andy keep track of my customer list — add Acme Corp, contact is Jane at jane@acme.com
```

The agent might create `groups/main/customers.md` with a structured format. You can ask it to create any kind of reference file: preferences, project notes, contact lists, etc.

---

## Scheduled Tasks

### What Are Tasks?

Scheduled tasks are automated agent runs that execute on a schedule. The agent runs your prompt at the specified time and sends the result back to the chat.

### Schedule Types

There are three schedule types:

**Cron** — recurring schedule using cron expressions:

```
@Andy every weekday morning at 9am, send me a summary of overnight emails
```
Cron expression: `0 9 * * MON-FRI`

```
@Andy every Monday at 8am, compile AI news from Hacker News and TechCrunch
```
Cron expression: `0 8 * * MON`

```
@Andy at noon every day, check my calendar and remind me of afternoon meetings
```
Cron expression: `0 12 * * *`

**Interval** — runs every N milliseconds:

```
@Andy check the server status every 30 minutes
```
Interval: `1800000` (30 min in ms)

**Once** — runs at a specific time:

```
@Andy remind me to call the dentist tomorrow at 9am
```
Once: `2026-02-25T09:00:00` (ISO timestamp)

### Context Modes

Each task runs in one of two context modes:

- **`group`** — the task runs with the group's existing session, maintaining conversational continuity. The agent remembers previous interactions.
- **`isolated`** — the task runs in a fresh session with no prior conversation context. Good for independent, repeating jobs.

### Scheduling Tasks

Just tell the agent what you want in natural language:

```
@Andy every Friday at 5pm, review this week's git commits and update the README if needed
@Andy in 2 hours, remind me to take the pizza out of the oven
@Andy every morning at 7am, check the weather and send me a forecast
```

The agent figures out the schedule type, cron expression, and context mode automatically.

### Managing Tasks

```
@Andy list all scheduled tasks
@Andy pause the Monday briefing task
@Andy resume the weather task
@Andy cancel the server status check
```

From the main channel, you can manage tasks across all groups:

```
@Andy list all scheduled tasks across groups
@Andy pause task task-1708123456-abc123
```

### Cross-Group Scheduling

Only the main channel can schedule tasks for other groups:

```
@Andy schedule a daily standup reminder in the Dev Team group at 9:15am
```

The task runs in that group's context with access to that group's files and memory.

---

## Customization

### No Config Files

NanoClaw doesn't use configuration files. The codebase is small enough that you customize it by modifying the code directly. Tell Claude Code what you want:

```
"Change the trigger word to @Bob"
"Make responses shorter and more direct"
"Add a custom greeting when I say good morning"
"Store conversation summaries weekly"
```

Or run `/customize` in Claude Code for guided changes.

### Adding Channels

NanoClaw supports multiple messaging channels through skills:

- **`/add-telegram`** — Add Telegram (alongside WhatsApp or as replacement)
- **`/add-discord`** — Add Discord
- **`/add-gmail`** — Add Gmail (as a tool or full channel)
- **`/add-voice-transcription`** — Add voice message transcription via Whisper

Run these skills in Claude Code to transform your installation.

### Skills System

Skills are Claude Code skill files (`.claude/skills/*/SKILL.md`) that teach Claude Code how to make specific changes to your NanoClaw installation. Instead of adding features to the base codebase, contributors create skills that transform your fork. You end up with clean, focused code that does exactly what you need.

---

## What the Agent Can Do

### Web Search and Fetch

The agent can search the web and fetch content from URLs:

```
@Andy search for the latest TypeScript 5.7 features
@Andy summarize the article at https://example.com/post
```

### Browser Automation

The agent has a full browser automation tool (`agent-browser`) that can:

- **Open pages**: `agent-browser open <url>`
- **Take snapshots**: `agent-browser snapshot -i` (shows interactive elements)
- **Click, fill, type**: Interact with page elements by reference
- **Take screenshots**: `agent-browser screenshot`
- **Save as PDF**: `agent-browser pdf output.pdf`
- **Fill forms**: Navigate multi-step forms
- **Extract data**: Get text, attributes, counts from page elements

The agent uses this automatically when a task requires web interaction — you don't need to tell it to use the browser explicitly.

### File Operations

The agent can read and write files in its sandboxed workspace (`/workspace/group/`). Files persist across sessions. This is useful for:

- Storing research notes
- Building structured data files
- Creating reports and documents
- Maintaining project files

### Bash Commands

The agent can run bash commands inside its container. This is sandboxed — commands execute in the container, not on your host machine. Common uses:

- Running scripts
- Processing data with command-line tools
- SQLite queries on the message database (main group only)
- Package management within the container

### Agent Swarms / Teams

NanoClaw supports agent swarms — teams of specialized sub-agents that collaborate on complex tasks. The main agent can spawn sub-agents, each with their own role, and coordinate their work. This is useful for large tasks that benefit from parallel work.

### Sending Messages Mid-Task

The agent can send messages to your chat while still working on a longer task using the `send_message` tool. This lets it acknowledge your request immediately while continuing to process in the background.

### Internal Thoughts

The agent can include internal reasoning in `<internal>` tags:

```
<internal>User wants a summary, let me check the three reports first.</internal>
Here's the summary you requested...
```

Text inside `<internal>` tags is logged but never sent to the user. This keeps the conversation clean while allowing the agent to reason transparently.

---

## Configuration Reference

### Environment Variables

Set these in your `.env` file or as system environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `ASSISTANT_NAME` | `Andy` | The assistant's name and trigger word prefix |
| `ASSISTANT_HAS_OWN_NUMBER` | `false` | Set to `true` if the assistant has its own dedicated phone number |
| `CONTAINER_IMAGE` | `nanoclaw-agent:latest` | Docker image for agent containers |
| `CONTAINER_TIMEOUT` | `1800000` (30 min) | Hard timeout for container execution (ms) |
| `CONTAINER_MAX_OUTPUT_SIZE` | `10485760` (10 MB) | Maximum stdout/stderr size before truncation |
| `IDLE_TIMEOUT` | `1800000` (30 min) | How long to keep a container alive after its last output (ms) |
| `MAX_CONCURRENT_CONTAINERS` | `5` | Maximum number of containers running simultaneously |
| `MAX_CONTEXT_MESSAGES` | `200` | Maximum messages sent as context per agent invocation |
| `TZ` | System timezone | Timezone for scheduled tasks (e.g., `America/New_York`) |
| `LOG_LEVEL` | — | Set to `debug` or `trace` for verbose container logs |

**Not configurable via environment** (hardcoded, change in code):

| Constant | Value | Location |
|----------|-------|----------|
| `POLL_INTERVAL` | `2000` ms | `src/config.ts` — how often the message loop checks for new messages |
| `SCHEDULER_POLL_INTERVAL` | `60000` ms | `src/config.ts` — how often the scheduler checks for due tasks |
| `IPC_POLL_INTERVAL` | `1000` ms | `src/config.ts` — how often IPC files are scanned |

### Per-Group Container Config

Each registered group can have custom container configuration in its registration entry:

```json
{
  "containerConfig": {
    "timeout": 600000,
    "idleTimeout": 300000,
    "additionalMounts": [
      {
        "hostPath": "~/projects/webapp",
        "containerPath": "webapp",
        "readonly": false
      }
    ]
  }
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `timeout` | `CONTAINER_TIMEOUT` | Hard timeout for this group's containers (ms) |
| `idleTimeout` | `IDLE_TIMEOUT` | Idle timeout for this group's containers (ms) |
| `additionalMounts` | `[]` | Extra directories to mount into the container |

Additional mounts appear at `/workspace/extra/{containerPath}` inside the container.

### Mount Allowlist

The mount allowlist at `~/.config/nanoclaw/mount-allowlist.json` controls which host directories can be mounted into containers. It's stored outside the project root so agents cannot modify it.

```json
{
  "allowedRoots": [
    {
      "path": "~/projects",
      "allowReadWrite": true,
      "description": "Project directories"
    }
  ],
  "blockedPatterns": [
    ".ssh", ".gnupg", ".aws", ".azure", ".gcloud",
    ".kube", ".docker", "credentials", ".env",
    ".netrc", ".npmrc", "id_rsa", "id_ed25519",
    "private_key", ".secret"
  ],
  "nonMainReadOnly": true
}
```

| Field | Description |
|-------|-------------|
| `allowedRoots` | Directories that can be mounted, with read-write permission flag |
| `blockedPatterns` | Glob patterns for paths that should never be mounted |
| `nonMainReadOnly` | If `true`, non-main groups can only mount directories as read-only |

---

## Service Management

### macOS (launchd)

```bash
# Start the service
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist

# Stop the service
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# Restart the service
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Check status
launchctl list | grep nanoclaw

# View live logs
tail -f logs/nanoclaw.log
```

### Linux (systemd)

```bash
# Start the service
systemctl --user start nanoclaw

# Stop the service
systemctl --user stop nanoclaw

# Restart the service
systemctl --user restart nanoclaw

# Check status
systemctl --user status nanoclaw

# View live logs
journalctl --user -u nanoclaw -f
```

### Rebuilding After Code Changes

If you've modified source code:

```bash
# Build and restart (macOS)
npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Build and restart (Linux)
npm run build && systemctl --user restart nanoclaw
```

### Rebuilding the Container Image

After changes to files in `container/`:

```bash
./container/build.sh
```

Note: The container buildkit caches aggressively. If `--no-cache` doesn't pick up your changes, you may need to prune the builder first and then re-run `./container/build.sh`.

---

## Directory Structure

```
NanoClaw/
├── src/                        # Source code
│   ├── index.ts                # Orchestrator: state, message loop, agent invocation
│   ├── channels/
│   │   └── whatsapp.ts         # WhatsApp connection, auth, send/receive
│   ├── config.ts               # Configuration constants and env vars
│   ├── container-runner.ts     # Spawns agent containers with mounts
│   ├── container-runtime.ts    # Container runtime detection and helpers
│   ├── db.ts                   # SQLite database operations
│   ├── group-queue.ts          # Per-group concurrency queue
│   ├── ipc.ts                  # IPC watcher and task processing
│   ├── router.ts               # Message formatting and outbound routing
│   ├── task-scheduler.ts       # Scheduled task execution
│   └── types.ts                # TypeScript interfaces
├── container/                  # Container image definition
│   ├── agent-runner/           # Agent runner code (runs inside container)
│   ├── skills/                 # Skills available to agents (e.g., agent-browser)
│   └── build.sh                # Container build script
├── groups/                     # Per-group data (memory, logs, files)
│   ├── main/                   # Main channel
│   │   ├── CLAUDE.md           # Agent memory and personality
│   │   ├── logs/               # Container execution logs
│   │   └── conversations/      # Archived conversation history
│   ├── global/                 # Shared read-only memory
│   │   └── CLAUDE.md           # Global personality/instructions
│   └── {group-name}/           # Registered group folders
├── store/                      # Persistent data
│   ├── messages.db             # SQLite database (messages, groups, tasks, sessions)
│   └── auth/                   # WhatsApp authentication state
├── data/                       # Runtime data
│   ├── ipc/                    # IPC file directories (per-group namespaces)
│   │   └── {group}/
│   │       ├── messages/       # Outbound message files
│   │       ├── tasks/          # Task operation files
│   │       └── input/          # Follow-up message input
│   └── sessions/               # Per-group Claude sessions
│       └── {group}/
│           ├── .claude/        # Claude Code settings and skills
│           └── agent-runner-src/  # Per-group agent runner source
├── logs/                       # Application logs
│   └── nanoclaw.log            # Main log file
├── docs/                       # Documentation
│   ├── SECURITY.md             # Security model
│   └── DEBUG_CHECKLIST.md      # Debug commands
├── .env                        # Environment variables (secrets, config)
├── CLAUDE.md                   # Project instructions for Claude Code
└── MANUAL.md                   # This file
```

---

## Troubleshooting

### Agent Not Responding

1. **Check if the service is running:**

   ```bash
   # macOS
   launchctl list | grep nanoclaw

   # Linux
   systemctl --user status nanoclaw
   ```

2. **Check if WhatsApp is connected:**

   ```bash
   grep -E 'Connected to WhatsApp|Connection closed' logs/nanoclaw.log | tail -5
   ```

3. **Check if messages are being received:**

   ```bash
   grep 'New messages' logs/nanoclaw.log | tail -10
   ```

4. **Check if containers are spawning:**

   ```bash
   grep -E 'Processing messages|Spawning container' logs/nanoclaw.log | tail -10
   ```

5. **Check for running containers:**

   ```bash
   docker ps --format '{{.Names}} {{.Status}}' | grep nanoclaw
   ```

### WhatsApp Auth Expired

If you see `WhatsApp authentication required` in the logs:

```bash
npm run auth
```

Scan the QR code in WhatsApp > Linked Devices, then restart the service.

### Container Timeout

If agents are timing out:

1. Check recent timeout logs:

   ```bash
   grep -E 'Container timeout|timed out' logs/nanoclaw.log | tail -10
   ```

2. Check the container's log file:

   ```bash
   ls -lt groups/*/logs/container-*.log | head -5
   ```

3. Increase the timeout if needed — set `CONTAINER_TIMEOUT` in your `.env` file (value in milliseconds).

### Messages Not Being Processed

- **Check the trigger pattern**: Messages in group chats must start with `@Andy` (or your configured trigger word). The main channel doesn't need a trigger.
- **Check group registration**: The group must be registered. Ask in your main channel: `@Andy list registered groups`.
- **Check concurrency**: If `MAX_CONCURRENT_CONTAINERS` containers are already running, new groups queue up. Check with:

  ```bash
  grep 'concurrency limit' logs/nanoclaw.log | tail -5
  ```

### Running `/debug`

For guided troubleshooting, open Claude Code in the NanoClaw directory and run:

```
/debug
```

This walks you through checking logs, container status, WhatsApp connection, and common issues.

### Quick Status Check

```bash
# Is the service running?
# macOS:
launchctl list | grep nanoclaw
# Linux:
systemctl --user status nanoclaw

# Any running containers?
docker ps --format '{{.Names}} {{.Status}}' | grep nanoclaw

# Recent errors?
grep -E 'ERROR|WARN' logs/nanoclaw.log | tail -20

# Is WhatsApp connected?
grep -E 'Connected to WhatsApp|Connection closed' logs/nanoclaw.log | tail -5

# Are groups loaded?
grep 'groupCount' logs/nanoclaw.log | tail -3
```

---

## Security Overview

NanoClaw's security is based on **container isolation**, not application-level permission checks. For the full security model, see [docs/SECURITY.md](docs/SECURITY.md).

### Container Isolation

Agents run in Linux containers (Docker or Apple Container):
- **Process isolation** — container processes cannot affect the host
- **Filesystem isolation** — only explicitly mounted directories are visible
- **Non-root execution** — runs as unprivileged `node` user (uid 1000)
- **Ephemeral containers** — fresh environment per invocation (`--rm`)

### Mount Security

- The mount allowlist (`~/.config/nanoclaw/mount-allowlist.json`) is stored outside the project and never mounted into containers — agents cannot modify it
- Sensitive paths (`.ssh`, `.gnupg`, `.aws`, credentials, private keys, etc.) are blocked by default
- Symlinks are resolved before validation to prevent traversal attacks
- The main group's project root is mounted read-only to prevent code modification

### IPC Authorization

Messages and task operations are authorized based on which group's IPC directory they originate from:
- Non-main groups can only send messages to their own chat
- Non-main groups can only manage their own tasks
- Only the main group can register groups, schedule cross-group tasks, and send messages to other chats

### Credential Handling

- **Claude auth tokens** are passed to containers via stdin (never written to disk as files)
- **WhatsApp session data** (`store/auth/`) stays on the host — never mounted into containers
- Only `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` are exposed to containers

### Privilege Summary

| Capability | Main Group | Non-Main Group |
|------------|------------|----------------|
| Project root access | Read-only | None |
| Group folder | Read-write | Read-write (own only) |
| Global memory | Implicit via project | Read-only |
| Additional mounts | Configurable | Read-only (when `nonMainReadOnly` is set) |
| Network access | Unrestricted | Unrestricted |
| Cross-group messaging | Yes | No |
| Cross-group task management | Yes | No |
| Group registration | Yes | No |

---

## FAQ

**Can I use this without WhatsApp?**

Yes. NanoClaw supports multiple channels. Run `/add-telegram` to use Telegram, `/add-discord` for Discord, or configure headless operation. You can also run multiple channels simultaneously.

**How do I change the bot's name?**

Set `ASSISTANT_NAME` in your `.env` file. This changes both the display name and the trigger word (from `@Andy` to `@YourName`). Then restart the service.

**Is my data stored?**

Messages are stored in a local SQLite database at `store/messages.db`. Conversation history is kept in per-group session files. Everything stays on your machine — nothing is sent to third parties beyond the Claude API for agent execution.

**Can multiple people use the same instance?**

Yes. Each WhatsApp group with NanoClaw registered is isolated — they have separate memory, files, and container sandboxes. The main channel (your self-chat) is the only admin channel. Other groups have restricted privileges.

**Why Docker? Can I use something lighter?**

Docker provides cross-platform support. On macOS, you can switch to Apple Container (a lighter native runtime) by running `/convert-to-apple-container` in Claude Code.

**How do I debug issues?**

The AI-native approach: open Claude Code in the NanoClaw directory and ask. "Why isn't the scheduler running?" "What's in the recent logs?" "Why did this message not get a response?" Or run `/debug` for guided troubleshooting.

**What if the setup isn't working?**

During setup, Claude will try to fix issues dynamically. If that doesn't work, run `claude` then `/debug`. Check the [Troubleshooting](#troubleshooting) section for common issues.

**Can the agent modify my files?**

Only files inside its mounted directories. The main group gets the project root as read-only and its own group folder as read-write. Other groups only get their own group folder (read-write) and global memory (read-only). Additional mounts are controlled by the allowlist.

**How do I give a group access to a directory on my machine?**

Register the group with `additionalMounts` in its container config, and make sure the path is listed in your mount allowlist (`~/.config/nanoclaw/mount-allowlist.json`). See [Configuration Reference](#per-group-container-config) for details.

**Can the agent access the internet?**

Yes. Containers have unrestricted network access. The agent can search the web, fetch URLs, and use the browser automation tool.

**How do I update NanoClaw?**

Run `/update` in Claude Code. It fetches the latest changes from upstream, merges them with your customizations, and runs any necessary migrations.

**Why no configuration files?**

NanoClaw is designed to be customized through code changes rather than configuration sprawl. The codebase is small enough that Claude Code can safely modify it to match your exact needs. If you prefer config files, you can ask Claude Code to add them.
