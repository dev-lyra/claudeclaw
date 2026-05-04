# Meta Ray-Ban Glasses Integration Report

Research into connecting Meta Ray-Ban smart glasses to Pantheon Automata (ClaudeClaw).

## Repos Evaluated

### 1. VisionClaw (Intent-Lab/VisionClaw)
**Most mature. 2.1k stars, 390 forks, active development.**

- iOS 17+ and Android 14+
- Streams glasses camera at ~1fps JPEG + 16kHz PCM audio to Gemini Live over WebSocket
- Gemini responds with audio (24kHz PCM) back through phone speaker
- Optional OpenClaw gateway for 56+ tool integrations
- WebRTC for live POV broadcasting to browsers
- Phone camera fallback for testing without glasses

**Verdict:** Strong project, but built around Gemini Live as the AI backend. Not a drop-in for ClaudeClaw since we use Claude, not Gemini. The architecture (camera stream -> AI -> audio response) is the right pattern though.

### 2. VisionClaude (mrdulasolutions/visionclaude)
**Most relevant to our stack. 8 stars, 4 forks, v1.2.0 (March 2026).**

- iOS Swift app + Node.js/Bun backend
- Connects glasses camera to Claude Code via MCP server (WebSocket)
- 720p @ 30fps from glasses, 1080p from iPhone
- Apple Speech Recognition for STT, ElevenLabs Flash v2.5 for TTS
- Web dashboard for monitoring
- Two modes: MCP channel (connects to existing Claude Code) or Gateway (standalone Express server with Claude API)
- Full MCP tool access through Claude

**Verdict:** This is the closest match. Same AI (Claude), same TTS (ElevenLabs), MCP-native. Could potentially connect directly to ClaudeClaw's agent system.

### 3. OpenVision (rayl15/OpenVision)
**41 stars, 3 forks, v1.2.1 (Feb 2026).**

- iOS 16+ Swift app
- Connects to either OpenClaw or Gemini Live
- Photo capture on voice command + live video (1fps) to Gemini
- Wake word activation ("Ok Vision")
- Barge-in and conversation mode support

**Verdict:** Simpler than VisionClaw. The OpenClaw integration path is interesting but the project is less active.

### 4. Meta Wearables DAT SDK (facebook/meta-wearables-dat-ios)
**Official Meta SDK. 345 stars, 82 forks, v0.6.0, developer preview.**

- Swift Package Manager distribution
- Video streaming + photo capture from glasses
- Session lifecycle management (pause/resume)
- MockDeviceKit for testing without hardware
- Requires Meta Developer account + app registration

**Verdict:** This is the foundation. All three projects above use this SDK. Any custom integration would also start here.

## Integration Assessment

### Can we integrate? Yes.

### Best path: Fork or adapt VisionClaude

VisionClaude already speaks our language:
- **Claude Code** as the AI (same as ClaudeClaw)
- **ElevenLabs** for TTS (same as our Agora Nexus voice pipeline)
- **MCP** for tool access (ClaudeClaw agents expose tools via Claude Code)
- **Node.js backend** (same runtime as ClaudeClaw)

### What integration would look like

```
Meta Ray-Ban Glasses
    |
    | (Bluetooth)
    v
iOS App (VisionClaude fork or custom Swift app)
    |
    | Camera frames (720p) + Audio (speech)
    | via WebSocket
    v
ClaudeClaw Backend (new endpoint or MCP bridge)
    |
    | Route to appropriate agent
    v
Pantheon Automata Agent (GiGi, Prometheus, etc.)
    |
    | Response text
    v
ElevenLabs TTS -> Audio back to phone -> Glasses speaker
```

### Three approaches, ranked by effort

#### Option A: Use VisionClaude as-is (Lowest effort)
- Clone VisionClaude, point its MCP channel at ClaudeClaw's Claude Code sessions
- The iOS app handles glasses -> camera -> Claude Code
- ClaudeClaw agents are already accessible via Claude Code's tool ecosystem
- Limitation: runs as a separate Claude Code session, not through the existing ClaudeClaw bot/agents directly

**Effort:** 1-2 days setup, no code changes to ClaudeClaw
**Trade-off:** Two separate systems (VisionClaude + ClaudeClaw) that don't share memory/context

#### Option B: Bridge VisionClaude into ClaudeClaw (Medium effort)
- Run VisionClaude's Gateway mode backend
- Add a new endpoint to ClaudeClaw's dashboard server that accepts camera frames + audio
- Route through ClaudeClaw's agent system (message queue, memory, hive mind)
- Responses go back through ElevenLabs TTS (already configured)

**Effort:** 1-2 weeks, new TypeScript module in ClaudeClaw
**Trade-off:** More work but full integration with memory, agent routing, and Agora Nexus

#### Option C: Build custom iOS app with DAT SDK (Highest effort)
- Build a minimal Swift app using Meta's DAT SDK directly
- Stream camera + mic to a new ClaudeClaw WebSocket endpoint
- Full control over the UX and data flow
- Could share the Agora Nexus voice pipeline entirely

**Effort:** 3-4 weeks, Swift + TypeScript development
**Trade-off:** Most control, most work, requires iOS development

### Requirements (all options)

- iPhone with iOS 17+ (physical device, no simulator)
- Meta Ray-Ban smart glasses (any model with camera)
- Meta Developer account (free, register at developers.meta.com)
- Meta AI app installed, glasses paired, Developer Mode enabled
- Xcode 15+ (to build/sideload the iOS app)

### Limitations to know

- **iOS only for now.** DAT SDK is iOS-only (Android version exists for VisionClaw but is separate)
- **Developer Preview.** Meta's DAT SDK is v0.6.0, still preview. API may change.
- **Camera is ~1fps JPEG** in most implementations (not real-time video). Good enough for "what am I looking at" but not for recording or fast motion.
- **Bluetooth dependency.** Glasses connect to phone via Bluetooth, phone connects to ClaudeClaw backend via internet. Latency chain: glasses -> phone -> server -> AI -> TTS -> phone -> glasses speaker.
- **Battery.** Continuous camera streaming drains glasses battery faster.

## Primary Use Case: Hands-Free Agent Access

The goal is to speak to any Pantheon Automata agent through the glasses. GiGi acts as the front desk (same as the Agora Nexus auto-router) and delegates to the right agent:

- "GiGi, what's on my calendar today" -> routes to Athena (ops)
- "Prometheus, research this company" -> routes directly to research
- "Iris, draft a follow-up to that email" -> routes to comms
- "What am I looking at?" -> GiGi uses the camera frame for visual context

The agent routing, display names, and alias system already built into ClaudeClaw handle this natively. The glasses are just a new input device feeding into the same pipeline the Agora Nexus voice room uses today.

The camera adds a visual layer: "read this label", "scan this business card", "what brand is this".

### What you see and hear

The **Ray-Ban Meta Display** (2025) has a full-color in-lens display (up to 3K Ultra HD), 12MP camera with 3x zoom, and open-ear speakers. This is a step up from the earlier camera-only model.

The display natively shows: notifications, AI responses, navigation, live captions, translations, photos/video, calendar, weather. The question is whether developers get API access to push custom content to the display (agent avatars, transcripts, status indicators).

**If display API access exists:**

| Device | Input | Output |
|--------|-------|--------|
| Glasses | Voice (mic), Vision (camera) | Voice (speakers), Agent avatar + name on display, response text |
| Phone | Touch (fallback) | Full dashboard, conversation history |

You could see the active agent's avatar and name on the lens display when they respond, with short text summaries. The full Agora Nexus council view would still be phone-only (the lens display is small and private, not a full screen).

**If display API access is limited/locked:**

The display would only show native Meta AI responses and system notifications. Custom agent visuals would fall back to the phone companion app. Voice interaction through the glasses would still work regardless.

**Unknown at this time:** Whether the DAT SDK (v0.6.0) exposes display output APIs, or if that's locked to Meta's own AI. This needs to be verified during Phase 1 testing. The SDK currently documents camera streaming and photo capture but display output is not confirmed for third-party apps.

## Recommendation

**Target Option B** (bridge into ClaudeClaw) since the goal is full agent team access, not just a single Claude session. The Agora Nexus auto-router already does this routing - the glasses would be another input channel into the same system.

### Phased approach:

**Phase 1: Validate the hardware (1-2 days)**
- Clone VisionClaude, run standalone
- Confirm glasses camera + mic + speaker pipeline works
- Test latency and audio quality

**Phase 2: Bridge into ClaudeClaw (1-2 weeks)**
- Add a WebSocket endpoint to ClaudeClaw's dashboard server
- Accept camera frames + transcribed speech from the iOS app
- Route through the existing agent system (message queue, memory, hive mind)
- Return responses via ElevenLabs TTS (already configured per-agent)
- Phone app shows agent avatars and transcript

**Phase 3: Polish the phone companion UI (1 week)**
- Build an Agora Nexus-style agent panel in the iOS app
- Show which agent is active, speaking indicators
- Camera preview overlay
- Conversation history

## Next Steps

1. Get a Meta Developer account if you don't have one
2. Enable Developer Mode on your glasses via Meta AI app
3. Clone VisionClaude and test Phase 1 (glasses -> Claude works at all?)
4. If latency and audio quality are acceptable, plan Phase 2 bridge build
5. Decide whether the phone companion UI (Phase 3) is worth the iOS development effort
