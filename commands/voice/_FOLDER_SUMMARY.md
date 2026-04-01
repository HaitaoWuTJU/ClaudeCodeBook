# Summary of `commands/voice/`

## Purpose of `voice/`

Provides the voice dictation feature for Claude Code, handling everything from the `/voice` command registration to audio recording and STT integration.

## Contents Overview

| File | Role |
|------|------|
| `commands/voice/index.ts` | Command registration and feature flag gates |
| `commands/voice/voice.ts` | Command handler (toggles, permission checks, UI responses) |
| `services/voice.js` | Audio capture and transcription via SoX/rec |

## How Files Relate to Each Other

```
┌──────────────────────────────────────────────────────────┐
│                      /voice command                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  commands/voice/index.ts                                 │
│  ┌────────────────┐                                      │
│  │  Command meta  │ → Is feature enabled?                │
│  │  • name        │ → Is user allowed?                   │
│  │  • description │ → Hide if disallowed?               │
│  │  • isEnabled   │ → Growth book gate                   │
│  │  • isHidden    │ → Voice mode gate                    │
│  └───────┬────────┘                                      │
│          │ lazy-load                                     │
│          ▼                                               │
│  commands/voice/voice.ts                                 │
│  ┌────────────────────────────────────────┐              │
│  │  Toggle logic                          │              │
│  │  1. Check auth / availability          │              │
│  │  2. requestMicrophonePermission()      │              │
│  │  3. updateSettingsForSource(...)       │              │
│  │  4. return text guidance               │              │
│  └───────────┬────────────────────────────┘              │
│              │ calls                                     │
│              ▼                                           │
│  services/voice.js                                       │
│  ┌────────────────────────────────────────┐              │
│  │  Audio service                         │              │
│  │  • starts SoX recording process       │              │
│  │  • pipes audio to voiceStreamSTT       │              │
│  │  • handles cleanup on stop             │              │
│  └────────────────────────────────────────┘              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Two-tier gating** — The `/voice` command is protected by both a growth book flag (`isVoiceGrowthBookEnabled`) and a user-level flag (`isVoiceModeEnabled`), ensuring the feature can be rolled out incrementally and disabled per-user.
2. **Lazy loading** — Command logic is loaded on-demand to keep startup time lean.
3. **Pre-flight chain** — Before enabling voice mode, the handler verifies authentication, microphone availability, recording tools (SoX), and OS permissions, surfacing platform-specific guidance when things go wrong.
4. **Language hint system** — The command tracks whether the user's language is unsupported by STT and informs them on the first two toggle-on events, capping the reminder via `LANG_HINT_MAX_SHOWS`.
5. **Process-based recording** — Audio capture delegates to a SoX subprocess (`sox -q -d`), keeping the Node.js event loop free during recording.
6. **No external STT library in-scope** — `services/voice.js` focuses purely on audio capture; the actual speech-to-text conversion is injected via the `voiceStreamSTT` parameter.
