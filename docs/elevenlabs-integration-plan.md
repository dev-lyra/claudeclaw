# ElevenLabs TTS Integration - War Room

## Context
The War Room currently uses Gemini Live (speech-to-speech, one voice per session). Lyra wants proper British accents and distinct per-agent voices in boardroom sessions. ElevenLabs TTS provides this. Adding it as a third engine mode alongside existing Gemini Live and Legacy (Cartesia), switchable from the dashboard.

## How It Works
**Your voice** -> Deepgram STT (speech to text) -> Gemini text LLM (thinks + tools) -> ElevenLabs TTS (text to speech with chosen voice) -> **audio back to you**

Same voice-to-voice experience. Same tool-calling (delegate, get_time, list_agents, answer_as_agent). Just different TTS.

## Prerequisites
- `ELEVENLABS_API_KEY` (from elevenlabs.io)
- `DEEPGRAM_API_KEY` (from deepgram.com, free tier available)

## Files to Change (7 files)

### 1. `warroom/voices.json` - Add elevenlabs_voice field
Add `elevenlabs_voice` per agent with ElevenLabs voice IDs. Existing fields untouched.

### 2. `warroom/requirements.txt` - Add elevenlabs extra
Already installed in venv, just make requirements.txt explicit: `pipecat-ai[websocket,deepgram,cartesia,silero,elevenlabs,google]>=1.0.0`

### 3. `warroom/server.py` - New pipeline + dispatch (biggest change)
- Update `read_pin_state()` to return 3-tuple: `(agent, mode, engine)`
- Extract shared tool schemas into `build_tool_schemas()` and `register_tool_handlers()` helpers
- Add `run_elevenlabs_mode()`: Deepgram STT -> SileroVAD -> Gemini text LLM (with tools) -> ElevenLabs TTS
- Update `run_warroom()` dispatch to check pin file engine field (env var override stays)

### 4. `src/dashboard.ts` - API changes
- Add `ELEVENLABS_VOICE_CATALOG` (~15 curated voices with British options)
- Extend `readPinState()` to include `engine` field (default: `"live"`)
- Extend POST `/api/warroom/pin` to accept `engine` field
- Extend GET/POST `/api/warroom/voices` to include `elevenlabs_voice` + `elevenlabs_catalog`

### 5. `src/dashboard-html.ts` - Engine toggle + voice dropdowns
- Add 3-way engine toggle: Gemini Live | ElevenLabs | Legacy
- Show Gemini dropdown when engine=live, ElevenLabs dropdown when engine=elevenlabs
- `setEngine()` writes to pin file, triggers respawn if meeting active

### 6. `src/warroom-html.ts` - Engine badge
- Small badge showing active engine near the mode selector
- Read from `/api/warroom/pin` on page load

### 7. `.env.example` - Document DEEPGRAM_API_KEY
Already has ELEVENLABS_API_KEY. Add DEEPGRAM_API_KEY entry.

## What Stays Untouched
- Gemini Live mode (run_live_mode) - zero changes
- Legacy mode (run_legacy_mode) - zero changes
- Client-side WebSocket/Pipecat transport - no changes needed
- src/index.ts subprocess spawning - no changes
- warroom/config.py - no changes (loads voices.json generically)
- All bot/agent/dashboard non-warroom code

## Implementation Order
1. voices.json + requirements.txt (schema + deps)
2. server.py (extract helpers, add elevenlabs mode, update dispatch)
3. dashboard.ts (API: pin engine, voice catalog, voice endpoints)
4. dashboard-html.ts (engine toggle UI, conditional dropdowns)
5. warroom-html.ts (engine badge)
6. Build, test all three modes

## Verification
1. `npm run build` - clean compile
2. `npm run test` - 347/347 pass
3. Start meeting with engine=live -> verify Gemini voice works (existing behavior)
4. Switch to engine=elevenlabs from dashboard -> verify ElevenLabs voice responds
5. Switch back to live -> verify no breakage
6. Pin different agents in elevenlabs mode -> verify distinct voices
7. Test tool calling in elevenlabs mode (delegate, get_time)
