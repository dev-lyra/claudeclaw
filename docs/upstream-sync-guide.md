# Upstream Sync Guide

How to pull updates from the official ClaudeClaw OS repo without breaking your customizations.

## One-Time Setup

Run this once to add the upstream repo as a remote:

```bash
git remote add upstream https://github.com/earlyaidopters/claudeclaw-os.git
```

Verify it worked:

```bash
git remote -v
```

You should see:
```
origin    https://github.com/dev-lyra/claudeclaw.git (fetch)
origin    https://github.com/dev-lyra/claudeclaw.git (push)
upstream  https://github.com/earlyaidopters/claudeclaw-os.git (fetch)
upstream  https://github.com/earlyaidopters/claudeclaw-os.git (push)
```

## When Mark Releases an Update

### Step 1: Make sure your work is clean

```bash
git status
```

If you have uncommitted changes, commit or stash them first:

```bash
git stash
```

### Step 2: Fetch the latest from upstream

```bash
git fetch upstream
```

This downloads Mark's latest code but does NOT change yours.

### Step 3: See what changed

```bash
git log --oneline upstream/main..HEAD
git log --oneline HEAD..upstream/main
```

First command: your commits that upstream doesn't have.
Second command: upstream commits you don't have yet.

### Step 4: Create a branch for the merge

```bash
git checkout -b upgrade/upstream-YYYY-MM-DD
```

Replace YYYY-MM-DD with today's date.

### Step 5: Merge upstream into your branch

```bash
git merge upstream/main
```

Three possible outcomes:

**A) Clean merge (no conflicts):** Git auto-merges everything. You'll see a merge commit message editor. Save and close.

**B) Merge conflicts:** Git will tell you which files conflict. Open each one, look for the conflict markers:

```
<<<<<<< HEAD
your version of the code
=======
Mark's version of the code
>>>>>>> upstream/main
```

For each conflict:
- If it's YOUR customization: keep your version
- If it's Mark's new feature: keep his version
- If both changed the same thing: combine them carefully

After resolving all conflicts:

```bash
git add .
git commit
```

**C) Major conflicts everywhere:** Abort and ask for help:

```bash
git merge --abort
```

### Step 6: Build and test

```bash
npm install
npm run build
npm run test
```

All 347+ tests should pass. If something breaks, check the conflict resolutions.

### Step 7: Test manually

- Start a meeting in Gemini Live mode - does it work?
- Switch to ElevenLabs mode - does it work?
- Check the dashboard loads correctly
- Send a message to the bot in Telegram

### Step 8: Merge to main

```bash
git checkout main
git merge upgrade/upstream-YYYY-MM-DD
git push origin main
```

### Step 9: Restart services

```bash
npm run build
# Restart all agents
for agent in main comms content ops research; do
  launchctl bootout gui/$(id -u)/com.claudeclaw.$agent 2>/dev/null
  sleep 1
  launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claudeclaw.$agent.plist
done
```

### Step 10: Clean up

Delete the upgrade branch:

```bash
git branch -d upgrade/upstream-YYYY-MM-DD
```

If you stashed changes in Step 1:

```bash
git stash pop
```

## Your Customizations (What May Conflict)

These are the files we've modified from the upstream base. When merging, pay attention to conflicts in these files:

| File | What we changed | Conflict risk |
|------|----------------|---------------|
| `warroom/server.py` | Added `run_elevenlabs_mode()`, extracted shared tool helpers, updated `read_pin_state()` to 3-tuple with engine, updated `run_warroom()` dispatch, fixed pipecat 1.0 import path | HIGH - if Mark refactors the server |
| `src/dashboard.ts` | Added `ELEVENLABS_VOICE_CATALOG`, extended pin state with engine field, extended voice API endpoints with elevenlabs_voice | HIGH - if Mark changes voice endpoints |
| `src/dashboard-html.ts` | Added engine toggle UI, conditional voice dropdowns, `setVoiceEngine()` function. Rebranded dashboard header + browser title from "ClaudeClaw" to "Pantheon Automata" (lines 8, 160) | MEDIUM - if Mark redesigns voice UI |
| `src/warroom-html.ts` | Added engine badge, fixed bot transcript to use pinnedAgent instead of hardcoded 'main', changed to -meet avatar variants | MEDIUM |
| `src/bot.ts` | Fixed /dashboard to send localhost URL as code block instead of inline button | LOW |
| `src/gemini.ts` | Updated model from gemini-2.0-flash to gemini-2.5-flash | LOW - Mark may do this too |
| `src/skill-registry.test.ts` | Fixed tests to work with real skills directory | LOW |
| `warroom/voices.json` | Added elevenlabs_voice field per agent | LOW - additive |
| `warroom/requirements.txt` | Added elevenlabs,google extras to pipecat-ai | LOW - additive |
| `warroom/personas.py` | Agents introduce themselves by display names (GiGi, Prometheus, Iris, Apollo, Athena) alongside canonical ids. Auto-router roster lists both. | MEDIUM - if Mark rewrites persona text |
| `warroom/router.py` | Added `AGENT_ALIASES` dict mapping display names to canonical ids; extended `_agent_pattern` regex to accept both. Match handling resolves aliases back to canonical id before routing. | MEDIUM - if Mark restructures the name-prefix regex |
| `agents/{research,comms,content,ops}/CLAUDE.md` | Renamed titles to `Prometheus (Research Agent)`, `Iris (Comms Agent)`, `Apollo (Content Agent)`, `Athena (Ops Agent)`. Intro paragraph now tells the agent its display name and canonical id. | LOW - additive text at top |
| `agents/{research,comms,content,ops}/agent.yaml` | Added `display_name:` field (Prometheus/Iris/Apollo/Athena). Canonical `name` field unchanged. | LOW - additive |
| `agents/_template/agent.yaml.example` | Documented the optional `display_name` field. | LOW - additive |
| `src/agent-config.ts` | Added `displayName?: string` to `AgentConfig`, `getAgentCapabilities()`, and `listAllAgents()`. Parses `display_name` from yaml. | MEDIUM - if Mark refactors the config shape |
| `src/orchestrator.ts` | Added `displayName?: string` to `AgentInfo`. Surfaced in `initOrchestrator`. Progress messages now say `Delegating to <displayName or name>...`. | MEDIUM |
| `src/dashboard.ts` | `/api/agents` now returns `displayName` per agent; `main` is hardcoded to `GiGi`. | LOW - additive |
| `src/dashboard-html.ts` | Agent modal title and mission board columns prefer `displayName || name`. Reordered sections: Tasks/Mission Control moved up, memory highlights side-by-side with Live Meetings, Memory Landscape + Agora Nexus Voices in right column. Summary bar grid changed from 4 to 5 columns. War Room card removed (replaced by summary tile). Mission board wraps instead of scrolling. Avatar URLs cache-busted. Avatar cache TTL 1h. | HIGH - major layout reorder |
| `src/warroom-html.ts` | Agent cards render `displayName || name` (falls back to id). Renamed "War Room" to "Agora Nexus", "Assembling your war council" to "Convening the Pantheon Automata council". Stage avatar nameplates hidden. Role titles updated (Chief of Staff, Research, Comms, Content Creator, Operations). | MEDIUM |
| `src/index.ts` | `/tmp/warroom-agents.json` now includes `display_name` per agent (and `GiGi` for `main`). | LOW - additive |
| `warroom/router.py` | `_load_dynamic_aliases()` merges display-name aliases from the roster file into `AGENT_ALIASES` on import, so routing picks up custom agents without a code change. | LOW - additive |

## Files That Never Conflict

These are personal/local and not tracked in upstream:

- `.env` - your API keys and config
- `store/claudeclaw.db` - your database
- `agents/*/agent.yaml` - your agent configs (in ~/.claudeclaw)
- `warroom/voices.json` - voice selections (may diverge from upstream defaults)
- `CLAUDE.md` - your parent-level personality file is at ~/Dev/CLAUDE.md

## Tips

- Always merge on a branch first, never directly on main
- Run tests before merging to main
- If a merge looks scary, ask for help in the terminal - don't force it
- Keep this doc updated when you add new customizations
