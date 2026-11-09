# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process that connects to WhatsApp, routes messages to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/whatsapp.ts` | WhatsApp connection, auth, send/receive |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals, env var defaults |
| `src/env.ts` | Custom .env parser — secrets stay off `process.env` |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/container-runtime.ts` | Docker abstraction (swap runtime here) |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `src/group-queue.ts` | Per-group concurrency queue with global container limit |
| `src/group-folder.ts` | Path validation and group folder security |
| `src/mount-security.ts` | Mount allowlist validation |
| `src/types.ts` | Channel interface and shared types |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/agent-runner/src/index.ts` | Container-side agent runner (reads stdin, writes stdout) |
| `container/skills/agent-browser.md` | Browser automation tool (available to all agents via Bash) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update` | Pull upstream NanoClaw changes, merge with customizations, run migrations |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload (tsx watch)
npm run build        # Compile TypeScript → dist/
npm run start        # Run compiled version
npm run typecheck    # Type check without emitting
npm run format       # Format with prettier
npm run test         # Run tests once (vitest)
npm run test:watch   # Run tests in watch mode
npm run test -- src/db.test.ts   # Run a single test file
npm run auth         # WhatsApp authentication setup
./container/build.sh # Rebuild agent container image
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Environment Variables

All variables have sensible defaults; `.env` is optional. Secrets are never loaded into `process.env` — they're parsed from disk via `src/env.ts` and passed to containers via stdin only.

| Variable | Default | Purpose |
|----------|---------|---------|
| `ASSISTANT_NAME` | `Andy` | Trigger word for non-main groups |
| `ASSISTANT_HAS_OWN_NUMBER` | `false` | Whether assistant has its own WhatsApp number |
| `CONTAINER_IMAGE` | `nanoclaw-agent:latest` | Container image to use |
| `MAX_CONCURRENT_CONTAINERS` | `5` | Global concurrent container limit |
| `CONTAINER_TIMEOUT` | `1800000` | Max execution time per invocation (ms) |
| `IDLE_TIMEOUT` | `1800000` | Keep-alive after last result (ms) |
| `CONTAINER_MAX_OUTPUT_SIZE` | `10485760` | Max container stdout size (bytes) |
| `LOG_LEVEL` | `info` | Pino log level |
| `TZ` | system | Timezone for cron tasks |

## Architecture: Message Flow

```
WhatsApp (Baileys) → SQLite (messages table)
                              ↓
                    Polling loop (src/index.ts, ~2s)
                              ↓
                    Per-group queue (src/group-queue.ts)
                    [global MAX_CONCURRENT_CONTAINERS limit]
                              ↓
                    Container spawn (src/container-runner.ts)
                    [per-group mounts, isolated session]
                              ↓
                    Claude Agent SDK (container/agent-runner/)
                    [reads JSON stdin, writes JSON stdout]
                              ↓
                    IPC watcher (src/ipc.ts)
                              ↓
                    Message router (src/router.ts) → WhatsApp
```

The orchestrator tracks two cursors per group: `lastTimestamp` (messages seen) and `lastAgentTimestamp` (messages processed by agent). On restart, unprocessed messages are recovered automatically.

## Container I/O Protocol

**Stdin** (JSON): `{ prompt, sessionId, groupFolder, chatJid, isMain, secrets }`

**Stdout**: JSON wrapped in sentinel markers (one object per output):
```
---NANOCLAW_OUTPUT_START---
{"status":"success","result":"...","newSessionId":"..."}
---NANOCLAW_OUTPUT_END---
```

**IPC follow-up messages**: Written as files to `/workspace/ipc/input/` inside the container while the agent is running. The host IPC watcher (`src/ipc.ts`) handles outbound messages written by the agent.

## Key Patterns

**Cursor rollback**: If a container errors _before_ sending any output, the cursor rolls back so the message retries. If it errors _after_ sending output, the cursor advances to avoid duplicates.

**Session persistence**: Each group's Claude Agent SDK session ID is stored in SQLite and passed on every container invocation. Sessions are kept alive for 30 min (idle timeout) to avoid cold starts.

**Sentinel markers**: Container output is parsed via `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` markers. `<internal>...</internal>` blocks are stripped before sending to users.

**IPC namespaces**: Agent IPC lives at `data/ipc/{groupFolder}/` — per-group directories prevent cross-group access. Non-main groups can only write to their own namespace.

**Trigger rules**: Main group processes all messages. Other groups only wake on `@Andy` prefix (configurable via `TRIGGER_PATTERN`). All accumulated unread messages since last agent run are sent as context.

**Task snapshots**: Before each agent run, the current task list (filtered to the group's scope) is written into the container mount so the agent can query and schedule tasks.

**Secrets pattern**: Never pass secrets via `process.env` (leaks to child processes). Use `readEnvFile()` from `src/env.ts` to read from disk, then pass to containers via stdin JSON only.

**Group folder validation**: Folder names must match `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`. The name `global` is reserved. `isValidGroupFolder()` / `assertValidGroupFolder()` in `src/group-folder.ts` enforce this.

**Swapping container runtime**: All Docker calls are isolated to `src/container-runtime.ts`. Change that single file (or set `CONTAINER_RUNTIME` env) to swap Docker for Podman or another OCI runtime.

## Database Tables

| Table | Purpose |
|-------|---------|
| `chats` | Chat metadata (JID, name, channel, is_group) |
| `messages` | Message log with sender, content, timestamp |
| `scheduled_tasks` | Cron/interval/once tasks with schedule and context_mode |
| `task_run_logs` | Execution history with duration and error info |
| `router_state` | Per-group cursor state (last_timestamp, last_agent_timestamp) |
| `sessions` | Claude Agent SDK session IDs per group folder |
| `registered_groups` | Group config: JID → name, folder, trigger_pattern, containerConfig |

Tests use `_initTestDatabase()` (from `src/db.ts`) in `beforeEach` for a fresh in-memory SQLite instance per test.

## Security Model

- **Container isolation**: Agents run in containers (Apple Container on macOS, Docker elsewhere) — no host filesystem access
- **Per-group mounts**: Each group only sees its own folder plus a global read-only directory
- **Mount allowlist**: External file at `~/.config/nanoclaw/mount-allowlist.json` controls what extra paths can be mounted (not accessible from inside container)
- **Authorization**: Main group has admin powers (register groups, manage all tasks). Other groups are restricted to their own scope
- **Secrets**: Never written to disk — passed via stdin only
- **Path traversal defense**: `ensureWithinBase()` in `src/group-folder.ts` prevents escaping base directories; applied to both group folders and IPC paths

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
