# wire-walkie-sdk

Shared Swift SDK for wire-walkie clients.

## Status

🚧 **Work in progress — no code yet.**

This SDK will be extracted from the first working client
([wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift))
once the patterns have proven themselves in production.

## Planned scope

- **Yodel HTTP client** — `POST /v1/chat/completions` with SSE stream parsing
- **Pairing** — Sound-QR device registration and per-device secret management
- **Session management** — persistent session IDs across requests
- **TTS** — on-device (AVSpeechSynthesizer) with optional gateway audio fallback
- **Error handling** — retry, backoff, timeout policies
- **Configuration** — endpoint discovery via `/.well-known/yodel.json`

## Why not now?

The first client (wire-walkie-swift) implements the HTTP+SSE logic
inline — ~200 lines of straightforward Swift. Extracting a library
before the patterns are battle-tested is premature abstraction.

Once a second client needs the same logic (e.g. wire-walkie-android,
macOS widget, CLI tool), the shared code will move here.

## Related

- [wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift) — first client (iOS)
- [plugin-openyodel-hermes](https://github.com/openyodel/plugin-openyodel-hermes) — server-side plugin
- [INTEGRATION.md](https://github.com/openyodel/plugin-openyodel-hermes/blob/main/INTEGRATION.md) — protocol reference
