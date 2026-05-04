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

### Which approach to use

**Hard reset** (recommended for major releases): Wipes your local code and replaces it with Mark's latest. Then re-apply customizations on top. This is what we did for v6 and it's far cleaner than trying to merge 100+ commits across massive generated HTML files.

**Merge** (fine for small/patch updates): Pulls in Mark's changes and attempts to auto-merge. Use this when the update is small and you can see that the files we customise weren't heavily touched.

When in doubt, use hard reset.

---

### Hard reset approach (major releases)

#### Step 1: Commit any uncommitted work

```bash
git status
git stash  # if anything is uncommitted
```

#### Step 2: Note your current HEAD sha

```bash
git rev-parse HEAD
```

Save this — you'll use it to recover customisation code from history after the reset.

#### Step 3: Hard reset to upstream

```bash
git fetch upstream
git reset --hard upstream/main
rm -rf node_modules
npm install
npm run build
```

`.env`, `store/`, and all gitignored files are untouched.

#### Step 4: Build and test

```bash
npm run test
```

465+ tests should pass. Known false failures on this machine:
- `voice.test.ts` — ffmpeg not installed locally, environment issue not a code bug
- `dashboard.contract.test.ts` — one test expects 400 from `/api/chat/history` but Mark intentionally changed it to 200; test wasn't updated upstream

#### Step 5: Push and restart

```bash
git push origin main --force
for agent in main comms content ops research; do
  launchctl bootout gui/$(id -u)/com.claudeclaw.$agent 2>/dev/null
  sleep 1
  launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claudeclaw.$agent.plist
done
```

#### Step 6: Restore custom docs (always do this after a hard reset)

These files live in `docs/` but aren't in upstream, so a hard reset deletes them:

```bash
git show <your-old-HEAD-sha>:docs/upstream-sync-guide.md > docs/upstream-sync-guide.md
git show <your-old-HEAD-sha>:docs/meta-glasses-integration-report.md > docs/meta-glasses-integration-report.md
git show <your-old-HEAD-sha>:docs/agent-personality-guide.md > docs/agent-personality-guide.md
git show <your-old-HEAD-sha>:docs/elevenlabs-integration-plan.md > docs/elevenlabs-integration-plan.md
```

#### Step 7: Re-apply customisations

See the customisations section below. After a hard reset, all customisations need to be re-applied from scratch. Use `git show <old-sha>:<file>` to read the old code without checking it out.

---

### Merge approach (small/patch updates)

#### Step 1: Make sure your work is clean

```bash
git status
git stash  # if anything is uncommitted
```

#### Step 2: Fetch the latest from upstream

```bash
git fetch upstream
```

#### Step 3: See what changed

```bash
git log --oneline upstream/main..HEAD   # your commits upstream doesn't have
git log --oneline HEAD..upstream/main   # upstream commits you don't have
git diff --name-only HEAD upstream/main # files that differ
```

If you see heavy changes to `src/dashboard-html.ts`, `src/dashboard.ts`, or `src/warroom-html.ts` — switch to the hard reset approach instead.

#### Step 4: Create a branch and merge

```bash
git checkout -b upgrade/upstream-YYYY-MM-DD
git merge upstream/main
```

Conflict markers look like:
```
<<<<<<< HEAD
your version
=======
Mark's version
>>>>>>> upstream/main
```

Rule of thumb:
- Your customisation in a section Mark didn't touch: keep yours
- Mark's new feature in a section you didn't touch: keep his
- Both touched the same section: combine carefully

After resolving:
```bash
git add .
git commit
npm install && npm run build && npm run test
```

#### Step 5: Merge to main

```bash
git checkout main
git merge upgrade/upstream-YYYY-MM-DD
git push origin main
git branch -d upgrade/upstream-YYYY-MM-DD
```

#### Step 6: Restart agents

```bash
for agent in main comms content ops research; do
  launchctl bootout gui/$(id -u)/com.claudeclaw.$agent 2>/dev/null
  sleep 1
  launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claudeclaw.$agent.plist
done
```

---

## Your Customisations (What May Conflict)

**Current state as of 2026-05-04:** We are on a clean v6 base (hard reset). Customisations below are pending re-implementation. Once re-applied, update this table with the actual files and line numbers.

### Pending re-implementation

| Customisation | Files to modify | Notes |
|--------------|----------------|-------|
| **ElevenLabs voice engine** | `warroom/server.py`, `src/dashboard.ts`, `src/dashboard-html.ts`, `warroom/voices.json`, `warroom/requirements.txt` | Third voice mode alongside Pika and Gemini Live. Reference code at commit `f76dba5`. |
| **Agent DisplayNames (Pantheon Automata)** | `src/agent-config.ts`, `src/orchestrator.ts`, `src/dashboard.ts`, `src/dashboard-html.ts`, `src/bot.ts`, `src/index.ts`, `warroom/personas.py`, `warroom/router.py`, `agents/*/agent.yaml`, `agents/*/CLAUDE.md` | GiGi / Prometheus / Iris / Apollo / Athena. Reference code at commit `a2a9f1d`. |
| **Animated Agora Nexus loading page** | `src/warroom-html.ts` | Cinematic boardroom reveal animation, Agora Nexus branding. Reference code at commit `a2a9f1d`. |

### Recovering old customisation code

All pre-v6 code is still in git history. Pull any file without checking it out:

```bash
git show a2a9f1d:<file>                 # pre-v6 HEAD (all customisations)
git show f76dba5:warroom/server.py      # ElevenLabs server implementation
git show f76dba5:docs/elevenlabs-integration-plan.md
```

Use `git reflog` if you lose track of the old sha.

---

## Files That Never Conflict

These are personal/local and not tracked in upstream:

- `.env` — API keys and config
- `store/claudeclaw.db` — database and memories
- `warroom/voices.json` — voice selections (gitignored)
- `~/Dev/CLAUDE.md` — personality file, lives outside the repo

## Tips

- For major releases: hard reset, then re-apply customisations
- For patch releases: merge, watch for conflicts in the files listed above
- After any hard reset: immediately restore custom docs from git history (Step 6)
- Keep this table updated when customisations are re-applied
