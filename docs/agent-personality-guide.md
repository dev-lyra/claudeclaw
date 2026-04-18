# Agent Personality Guide

How to expand or modify an agent's personality. Two files per agent, two different contexts.

## Files to Edit

### 1. `agents/{id}/CLAUDE.md` (Telegram / text personality)

This is the agent's system prompt. Claude reads it before every message. Put detailed personality traits, communication style, boundaries, and domain expertise here.

**Location:** `agents/comms/CLAUDE.md`, `agents/research/CLAUDE.md`, etc.

**Format:** Standard markdown. The first section should identify the agent:

```markdown
# Iris (Comms Agent)

Your name is Iris. Your canonical agent id is `comms`. Both names refer to you...
```

Then add whatever sections you want: personality traits, communication style, things to avoid, domain knowledge, etc.

**After editing:** No build or restart needed. Agents read this file live on every message.

### 2. `warroom/personas.py` (Agora Nexus voice personality)

The `AGENT_PERSONAS` dict (starts around line 45) defines how each agent behaves in voice conversations. Gemini Live uses this as the system prompt when speaking as that agent.

**Format:** Python triple-quoted string inside the dict:

```python
"comms": (
    """You are Iris (agent id: comms), Comms lead...

Specialty: drafting messages, customer replies...

"""
    + SHARED_RULES
),
```

**Keep it shorter than CLAUDE.md.** This is injected into every voice turn, so longer = more tokens per utterance. Focus on personality and specialty, not detailed instructions.

**After editing:** Restart main to pick up changes:

```bash
launchctl kickstart -k gui/$(id -u)/com.claudeclaw.main
```

## Step-by-Step: Expand an Agent's Personality

1. **Pick the agent.** Know its canonical id (main, research, comms, content, ops) and display name (GiGi, Prometheus, Iris, Apollo, Athena).

2. **Edit `agents/{id}/CLAUDE.md`.** Add personality details. Example sections you might add:

   - Personality traits (tone, quirks, how they handle conflict)
   - Communication style (formal vs casual, verbose vs terse)
   - Domain expertise (what they know deeply)
   - Boundaries (what they refuse to do, when they escalate)
   - Relationship to other agents (how they talk about the team)

3. **Edit `warroom/personas.py`.** Update the matching entry in `AGENT_PERSONAS`. Keep the opening line format:

   ```
   You are {DisplayName} (agent id: {id}), {Role} in the War Room. {One sentence about what they do}. Personality: {2-3 adjectives}. The user may address you as either "{DisplayName}" or "{id}". Both names refer to you.
   ```

   Then a `Specialty:` paragraph describing their focus.

4. **Keep them consistent.** The CLAUDE.md and persona should describe the same character. CLAUDE.md can go deeper, but they shouldn't contradict each other.

5. **Restart if you edited personas.py:**

   ```bash
   launchctl kickstart -k gui/$(id -u)/com.claudeclaw.main
   ```

6. **Test.** Send a message to the agent in Telegram and check the tone. Open the Agora Nexus and talk to them in voice.

## Agent Map

| ID | Display Name | CLAUDE.md | Persona (line ~) |
|----|-------------|-----------|-----------------|
| main | GiGi | `~/Dev/CLAUDE.md` (project-level) | Line 46 |
| research | Prometheus | `agents/research/CLAUDE.md` | Line 57 |
| comms | Iris | `agents/comms/CLAUDE.md` | Line 66 |
| content | Apollo | `agents/content/CLAUDE.md` | Line 75 |
| ops | Athena | `agents/ops/CLAUDE.md` | Line 84 |

## Tips

- Don't make personas too long. 3-5 sentences of personality + 2-3 sentences of specialty is the sweet spot for voice.
- CLAUDE.md can be as detailed as you want. More detail = more consistent behavior.
- Test voice personas by saying "tell me about yourself" in the Agora Nexus. You'll hear if the personality comes through.
- If you add a new agent, follow the same pattern. The `+ New Agent` wizard creates a basic CLAUDE.md, but you'll want to flesh it out.
