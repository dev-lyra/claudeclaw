# ClaudeClaw - Full Codebase Analysis & Replication Guide

## What Is It

ClaudeClaw is a personal AI assistant that connects Claude Code (via the Claude Agent SDK) to Telegram. It runs as a persistent background service on macOS or Linux, giving you a conversational AI assistant on your phone that can execute code, manage files, search the web, read your Obsidian vault, send emails, check calendars, and delegate work across multiple specialized agents.

Think of it as: **Telegram bot frontend + Claude Code backend + persistent memory + multi-agent orchestration + integrations (WhatsApp, Slack, voice, Obsidian)**.

---

## Architecture Overview

```
Telegram (grammy) ----> Message Queue (per-chat FIFO)
                              |
                              v
                     Bot Logic (routing, media, commands)
                              |
                              v
                     Claude Agent SDK (subprocess)
                        - Session resumption
                        - CLAUDE.md system prompt
                        - Tools (bash, files, web, MCP)
                              |
                              v
                     Response streamed back to Telegram
                              |
                              v
                     Async post-processing:
                        - Conversation logged to SQLite
                        - Memory extraction (Gemini)
                        - Token usage tracked

Parallel systems:
  - Scheduler (cron tasks, mission tasks)
  - Memory consolidation (every 30 min)
  - Memory decay (daily)
  - Dashboard (Hono web UI on port 3141)
  - Security (PIN lock, audit log, kill switch)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | TypeScript (ES2022, NodeNext modules) |
| Runtime | Node.js >= 20 |
| AI Backend | Claude Code via `@anthropic-ai/claude-agent-sdk` |
| Telegram | `grammy` (Bot API framework) |
| Database | SQLite via `better-sqlite3` (synchronous, embedded) |
| Web Dashboard | `hono` + `@hono/node-server` (lightweight HTTP) |
| Memory Embeddings | Google Gemini (`@google/genai`, text-embedding-001) |
| Memory Extraction | Gemini (structured JSON extraction from conversations) |
| Voice STT | Groq Whisper API (primary), whisper-cpp (fallback) |
| Voice TTS | ElevenLabs > Gradium AI > macOS `say` + ffmpeg |
| WhatsApp | `whatsapp-web.js` (Chromium-based) |
| Slack | `@slack/web-api` |
| Cron Parsing | `cron-parser` |
| Logging | `pino` + `pino-pretty` |
| Agent Config | `js-yaml` (YAML parsing for agent.yaml files) |
| Build | `tsc` (TypeScript compiler) |
| Dev | `tsx` (TypeScript execution without pre-compilation) |
| Tests | `vitest` |
| Service Manager | macOS launchd / Linux systemd / Windows PM2 |

---

## Source Files Breakdown (~14k lines total)

### Core (the stuff that makes it work)

| File | Lines | Purpose |
|------|-------|---------|
| `bot.ts` | 1631 | The big one. Telegram bot setup, message handling, command routing, media processing, file sending, streaming, model switching. Everything flows through here. |
| `db.ts` | 1952 | Database layer. Schema creation, all CRUD operations, field-level AES-256-GCM encryption, migration support. Every table lives here. |
| `index.ts` | 228 | Entry point. Parses `--agent` flag, loads config, inits DB, starts bot + scheduler + dashboard + decay sweeps + consolidation loops. PID locking. |
| `agent.ts` | 298 | Claude Code SDK wrapper. Spawns Claude as subprocess, manages sessions, tracks tokens/cost, handles abort/timeout. |
| `config.ts` | ~100 | Loads `.env`, exports typed config object with defaults. |
| `env.ts` | ~50 | Environment variable validation. |
| `state.ts` | ~30 | Shared mutable state (active sessions, abort controllers). |
| `logger.ts` | ~20 | Pino logger setup. |

### Memory System

| File | Lines | Purpose |
|------|-------|---------|
| `memory.ts` | 271 | Memory retrieval (semantic search + importance-ranked + consolidation insights). Decay sweep logic. Builds the `[Memory context]` block injected into each prompt. |
| `memory-ingest.ts` | ~180 | Post-conversation memory extraction via Gemini. Fires async after every turn. High bar for what gets saved (preferences, relationships, policies). Deduplication at 85% similarity. |
| `memory-consolidate.ts` | ~200 | Runs every 30 min. Batches unconsolidated memories, finds cross-cutting patterns via Gemini, generates insights. Detects stale/superseded memories. |
| `embeddings.ts` | ~80 | Gemini text-embedding-001 wrapper. 768-dimension vectors. Cosine similarity. |

### Multi-Agent System

| File | Lines | Purpose |
|------|-------|---------|
| `orchestrator.ts` | 236 | Agent registry (scans `agents/` dirs for `agent.yaml`). Delegation routing (`@agent: prompt` or `/delegate`). Runs delegated tasks in-process with timeout. |
| `agent-config.ts` | ~100 | YAML config loader for agent definitions. |
| `agent-create.ts` | 555 | Interactive agent creation wizard. Generates agent.yaml, CLAUDE.md, launchd plist, env vars. |
| `agent-create-cli.ts` | ~30 | CLI wrapper for agent creation. |

### Scheduling & Tasks

| File | Lines | Purpose |
|------|-------|---------|
| `scheduler.ts` | ~250 | Cron-based task execution. Checks every 60s, fires tasks through message queue. Also handles mission task claiming and execution. |
| `schedule-cli.ts` | ~120 | CLI tool for creating/listing/deleting/pausing scheduled tasks. |
| `mission-cli.ts` | ~120 | CLI tool for creating/listing/cancelling mission tasks (async agent work queues). |

### Integrations

| File | Lines | Purpose |
|------|-------|---------|
| `voice.ts` | 442 | Full voice pipeline. STT via Groq Whisper with local fallback. TTS cascade (ElevenLabs > Gradium > macOS say). Audio format conversion. |
| `whatsapp.ts` | ~300 | WhatsApp Web bridge. QR auth, message read/send, encrypted DB storage, outbox retry. Runs as separate daemon. |
| `slack.ts` | ~200 | Slack integration. Conversation listing, message history, send/reply. |
| `slack-cli.ts` | ~50 | CLI wrapper for Slack operations. |
| `obsidian.ts` | ~80 | Obsidian vault access helpers. |
| `media.ts` | ~150 | Media download/processing. Photo, document, video, audio handling from Telegram. Cleanup after 24h. |
| `gemini.ts` | ~100 | Gemini API wrapper for video analysis and structured extraction. |

### Dashboard

| File | Lines | Purpose |
|------|-------|---------|
| `dashboard.ts` | 668 | Hono HTTP server. REST API endpoints for memories, tokens, tasks, agents, audit log. SSE for real-time updates. Token auth. |
| `dashboard-html.ts` | 2289 | Inline HTML/CSS/JS for the dashboard UI. Responsive, single-page app served as a string. |

### Security

| File | Lines | Purpose |
|------|-------|---------|
| `security.ts` | 214 | PIN lock (salted SHA-256), idle auto-lock, emergency kill phrase, audit logging. |

### Infrastructure

| File | Lines | Purpose |
|------|-------|---------|
| `message-queue.ts` | ~80 | Per-chat FIFO queue. Prevents race conditions when multiple messages arrive. Different chats process in parallel. |
| `migrations.ts` | ~150 | Version-based schema migration runner. Tracks applied versions. |

---

## Database Schema (SQLite)

All tables defined in `db.ts`. Key tables:

```sql
-- Session persistence (composite PK: one session per chat per agent)
sessions (chat_id, agent_id, session_id, created_at, updated_at)

-- Structured memories with embeddings
memories (id, chat_id, content, sector, importance, salience,
          embedding, pinned, consolidated, superseded_by,
          created_at, accessed_at)

-- Cross-memory pattern synthesis
consolidations (id, chat_id, summary, insight, source_ids,
                embedding, created_at)

-- Full conversation history
conversation_log (id, chat_id, agent_id, role, content, created_at)

-- Per-turn cost tracking
token_usage (id, session_id, chat_id, agent_id,
             input_tokens, output_tokens, cache_read, context_tokens,
             cost_usd, did_compact, model, created_at)

-- Cron-scheduled prompts
scheduled_tasks (id, chat_id, agent_id, prompt, cron, status,
                 next_run, last_run, last_result, created_at)

-- Async work queue for agents
mission_tasks (id, chat_id, agent_id, title, prompt, priority,
               status, result, claimed_at, completed_at, created_at)

-- Cross-agent activity awareness
hive_mind (id, agent_id, action, summary, artifacts, created_at)

-- Delegation tracking
inter_agent_tasks (id, from_agent, to_agent, prompt, result,
                   status, created_at, completed_at)

-- WhatsApp messages (encrypted)
wa_messages (id, chat_id, wa_chat_id, role, content, created_at)
wa_outbox (id, wa_chat_id, content, status, created_at)

-- Slack messages
slack_messages (id, chat_id, channel_id, role, content, created_at)

-- Security audit trail
audit_log (id, chat_id, agent_id, action, detail, blocked, created_at)
```

Field-level encryption (AES-256-GCM) on WhatsApp/Slack message content. Key auto-generated and stored in `.env` as `DB_ENCRYPTION_KEY`.

---

## How to Build This From Scratch

### Phase 1: Minimal Bot (Day 1-2)

Get a Telegram bot talking to Claude Code.

1. **Init project**
   ```bash
   mkdir claudeclaw && cd claudeclaw
   npm init -y
   npm install grammy @anthropic-ai/claude-agent-sdk better-sqlite3 dotenv
   npm install -D typescript @types/node @types/better-sqlite3 tsx
   npx tsc --init  # target ES2022, module NodeNext
   ```

2. **Create the Telegram bot**
   - Get a bot token from @BotFather
   - Use grammy to listen for messages
   - Route text messages to Claude Agent SDK
   - Send responses back to Telegram (handle the 4096 char limit with message splitting)

3. **Wire up Claude Code**
   - Use `@anthropic-ai/claude-agent-sdk` to spawn Claude as a subprocess
   - It uses the Claude CLI installed on your machine
   - Pass the user's message as the prompt
   - Collect the response text from tool results

4. **Add chat ID locking**
   - First message logs the chat ID
   - Set `ALLOWED_CHAT_ID` in .env
   - Reject messages from other chats

**You now have a working bot.** Everything after this is enhancement.

### Phase 2: Persistence (Day 3-4)

Make conversations survive restarts.

5. **SQLite database**
   - Create `db.ts` with better-sqlite3
   - Sessions table: store Claude Code session IDs per chat
   - Conversation log: store all messages for history/recall
   - Token usage: track input/output tokens and cost per turn

6. **Session resumption**
   - Claude Agent SDK supports resuming sessions by ID
   - Store the session ID after each call
   - Pass it back on the next message
   - This gives you multi-turn conversations that persist across bot restarts

7. **Message queue**
   - Simple per-chat FIFO queue (Map of chat_id -> promise chain)
   - Prevents race conditions when messages arrive faster than Claude responds
   - Different chats can process in parallel

### Phase 3: Memory (Day 5-7)

Give the bot long-term memory that outlasts individual sessions.

8. **Memory storage**
   - Memories table with: content, importance (0-1), salience (decaying value), timestamps
   - Importance determines decay rate (high = slow decay, low = fast)
   - Salience starts at importance value and decays daily

9. **Memory extraction**
   - After each conversation turn, fire-and-forget an async call to Gemini
   - Ask it to extract facts worth remembering (preferences, relationships, decisions)
   - Set a high bar: only save importance >= 0.5
   - Deduplicate: check cosine similarity against existing memories, skip if > 0.85

10. **Memory retrieval**
    - Before each Claude call, build a `[Memory context]` block
    - Three retrieval paths: semantic search (embeddings), high-importance recent, consolidation insights
    - Inject this into the system prompt so Claude has context

11. **Embeddings**
    - Use Gemini text-embedding-001 (768 dimensions)
    - Embed memories and consolidations for semantic search
    - Cosine similarity with 0.3 threshold for retrieval

12. **Consolidation**
    - Every 30 minutes, batch unconsolidated memories
    - Use Gemini to find cross-cutting patterns and generate insights
    - Detect contradictions and mark superseded memories
    - This is how the bot builds higher-level understanding over time

13. **Decay sweep**
    - Run daily
    - Reduce salience based on importance tier
    - Delete memories below 0.05 salience
    - Pinned memories are exempt

### Phase 4: Media & Voice (Day 8-10)

Handle photos, documents, videos, and voice messages.

14. **Media handling**
    - Download Telegram media to `workspace/uploads/`
    - Photos: pass file path to Claude (it can read images)
    - Documents: save locally, Claude opens them via Read tool
    - Videos: send to Gemini for analysis (too large for Claude directly)
    - Clean up files older than 24 hours

15. **Voice input**
    - Telegram sends voice messages as .ogg files
    - Transcribe with Groq Whisper API (fast, free tier)
    - Fallback to local whisper-cpp if no API key
    - Prepend transcription to message: `[Voice transcribed]: ...`

16. **Voice output**
    - TTS cascade: try ElevenLabs, then Gradium, then macOS `say`
    - Convert output to .ogg for Telegram voice message
    - Triggered by user preference or command

17. **File sending**
    - Parse `[SEND_FILE:/path]` markers in Claude's response
    - Send as Telegram document attachments
    - Support captions: `[SEND_FILE:/path|caption]`

### Phase 5: Scheduling (Day 11-12)

Run prompts on a cron schedule.

18. **Scheduler**
    - Scheduled tasks table: prompt, cron expression, next_run, status
    - Check every 60 seconds for tasks whose next_run has passed
    - Execute through the message queue (same path as user messages)
    - Store results in last_result column
    - CLI tool for create/list/delete/pause/resume

19. **Mission tasks**
    - Async work queue: prompt + priority + assigned agent
    - Status: queued -> running -> completed/failed
    - Claimed by scheduler, executed with timeout
    - CLI tool for management

### Phase 6: Multi-Agent (Day 13-15)

Run multiple specialized agents, each with their own personality and Telegram bot.

20. **Agent configuration**
    - `agents/{id}/agent.yaml`: name, description, bot token, model override, Obsidian config
    - `agents/{id}/CLAUDE.md` or `~/.claudeclaw/agents/{id}/CLAUDE.md`: custom system prompt
    - Each agent is a separate process: `node dist/index.js --agent comms`

21. **Orchestrator**
    - Agent registry: scan agent directories for configs
    - Delegation syntax: `@agent: prompt` or `/delegate agent prompt`
    - Runs delegated task in-process with timeout
    - Injects memory context into delegated prompts

22. **Hive mind**
    - Shared activity log across agents
    - Each agent records actions, summaries, artifacts
    - Other agents can see what's been done recently
    - Enables loose coordination without direct communication

23. **Inter-agent tasks**
    - Track delegation: who asked whom to do what
    - Store results for the requesting agent to reference

### Phase 7: Security (Day 16)

Lock it down.

24. **PIN lock**
    - Salted SHA-256 hash of PIN stored in env
    - Bot starts locked if configured
    - `/unlock PIN` to access, `/lock` to secure
    - All messages rejected while locked

25. **Idle auto-lock**
    - Track last activity timestamp
    - Auto-lock after N minutes of inactivity

26. **Kill switch**
    - Emergency phrase triggers immediate process exit
    - Stops all launchd/systemd services

27. **Audit logging**
    - Log every action: messages, commands, delegations, unlock attempts
    - Track blocked actions (locked state, unauthorized)
    - Queryable via dashboard

### Phase 8: Dashboard (Day 17-18)

Web UI for monitoring and management.

28. **HTTP server**
    - Hono framework, lightweight
    - Token-based auth
    - REST endpoints for all dashboard data

29. **Dashboard views**
    - Memories (timeline, pinned, low-salience)
    - Token usage (daily spend, cost curves)
    - Scheduled tasks (manage from browser)
    - Mission tasks (assign, reassign)
    - Agent activity (hive mind log)
    - Conversation history
    - Audit log

30. **Real-time updates**
    - Server-Sent Events (SSE) for live data
    - Dashboard HTML served as inline string (no build step, no static files)

### Phase 9: Integrations (Day 19-21)

Bridge to other platforms.

31. **WhatsApp**
    - `whatsapp-web.js` runs a Chromium instance
    - QR code auth flow
    - Read/send messages through Telegram commands
    - Encrypted storage in DB
    - Separate daemon process

32. **Slack**
    - `@slack/web-api` with user OAuth token
    - List conversations, read messages, send replies
    - Integrated into Telegram command flow

33. **Obsidian**
    - Read vault files via filesystem tools
    - Configurable per-agent vault paths and folder access
    - Read-only folder support

### Phase 10: Service Management (Day 22)

Run it 24/7.

34. **macOS launchd**
    - Plist template for each agent
    - Auto-restart on crash
    - Log to /tmp/
    - Install/uninstall scripts

35. **Linux systemd**
    - User service files
    - Standard start/stop/status

36. **Setup wizard**
    - Interactive script that walks through .env configuration
    - Detects dependencies (Node, Claude CLI)
    - Generates service files

---

## Key Design Decisions Worth Understanding

### Why SQLite, not Postgres?
Single-user system. No need for a database server. SQLite is embedded, zero-config, fast for this workload, and the DB file is trivially portable. `better-sqlite3` gives synchronous access which simplifies the code significantly.

### Why Gemini for memory, not Claude?
Cost and separation of concerns. Memory extraction and consolidation run on every turn. Using Claude for this would double your API costs. Gemini's free/cheap tier handles it well, and keeping memory processing separate from the conversation model avoids context pollution.

### Why subprocess (Agent SDK), not API?
Claude Code runs as a subprocess with full tool access: bash, file system, web search, MCP servers, Claude Code skills. The API alone can't do this. The Agent SDK gives you all of Claude Code's capabilities programmatically.

### Why per-chat message queues?
Without queuing, rapid-fire messages cause race conditions on session state. The queue ensures messages for the same chat process sequentially while different chats can run in parallel.

### Why inline HTML for the dashboard?
No frontend build step. No static file serving. The entire dashboard is a TypeScript string template. Simple, zero dependencies, ships with `tsc`. Trade-off: harder to maintain at scale, but fine for a personal tool.

### Why fire-and-forget memory extraction?
Memory processing should never slow down the user's response. Extract, embed, consolidate all happen async after the response is sent. If they fail, the bot keeps working. Memories are a best-effort enhancement, not a critical path.

---

## File Tree Reference

```
claudeclaw/
  src/
    index.ts            # Entry point, service initialization
    bot.ts              # Telegram bot, message handling, commands
    agent.ts            # Claude Code SDK wrapper
    db.ts               # Database schema + all queries
    config.ts           # Environment config loader
    env.ts              # Env validation
    state.ts            # Shared mutable state
    logger.ts           # Pino logger
    memory.ts           # Memory retrieval + decay
    memory-ingest.ts    # Async memory extraction (Gemini)
    memory-consolidate.ts # Pattern synthesis across memories
    embeddings.ts       # Gemini embedding wrapper
    orchestrator.ts     # Multi-agent registry + delegation
    agent-config.ts     # YAML agent config loader
    agent-create.ts     # Agent creation wizard
    scheduler.ts        # Cron + mission task execution
    schedule-cli.ts     # Schedule management CLI
    mission-cli.ts      # Mission task CLI
    security.ts         # PIN, auto-lock, kill switch, audit
    dashboard.ts        # Hono web server + REST API
    dashboard-html.ts   # Inline HTML/CSS/JS for dashboard
    message-queue.ts    # Per-chat FIFO queue
    media.ts            # Media download + processing
    voice.ts            # STT + TTS pipeline
    whatsapp.ts         # WhatsApp bridge
    slack.ts            # Slack integration
    obsidian.ts         # Obsidian vault access
    gemini.ts           # Gemini API wrapper
    migrations.ts       # Schema migration runner
  agents/
    _template/          # Template for new agents
    comms/              # Communications agent
    content/            # Content creation agent
    ops/                # Operations agent
    research/           # Research agent
  scripts/
    setup.ts            # Interactive setup wizard
    migrate.ts          # Run DB migrations
    status.ts           # Health check
    notify.sh           # Telegram notification helper
    install-launchd.sh  # macOS service installer
    wa-daemon.ts        # WhatsApp daemon process
  migrations/
    version.json        # Migration version tracking
  store/
    claudeclaw.db       # SQLite database (runtime)
  workspace/
    uploads/            # Temporary media storage
  skills/               # Custom Claude Code skills
  .env                  # Secrets and configuration
  CLAUDE.md             # System prompt / project instructions
  package.json          # Dependencies and scripts
  tsconfig.json         # TypeScript config
```

---

## Dependency Count

Production: 11 packages. Dev: 6 packages. This is a lean project. The heaviest dependency is `whatsapp-web.js` (pulls in Chromium via Puppeteer). If you skip WhatsApp, the install is fast and light.
