# wire-walkie-sdk

Shared client library for wire-walkie apps — platform-agnostic design,
reference implementations in Swift, Kotlin, and TypeScript.

## Status

🚧 **Work in progress — no code yet.**

This library will be extracted from the first working client
([wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift))
once the patterns have proven themselves in production.

## Planned scope

- **Yodel HTTP client** — `POST /v1/chat/completions` with SSE stream parsing
- **Pairing** — Sound-QR device registration and per-device secret management
- **Session management** — persistent session IDs across requests
- **TTS** — on-device text-to-speech with optional gateway audio fallback
- **Error handling** — retry, backoff, timeout policies
- **Configuration** — endpoint discovery via `/.well-known/yodel.json`

## Language targets (planned)

| Language | Status | First consumer |
|----------|--------|---------------|
| Swift | 🚧 after MVP | wire-walkie-swift (iOS) |
| Kotlin | 🔮 planned | wire-walkie-android |
| TypeScript | 🔮 planned | wire-walkie-web (PWA) |

## Why not now?

The first client implements the HTTP+SSE logic inline — ~200 lines
of straightforward code. Extracting a library before the patterns
are battle-tested is premature abstraction.

Once a second client needs the same logic, the shared code moves here.

## Related

- [wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift) — first client (iOS)
- [plugin-openyodel-hermes](https://github.com/openyodel/plugin-openyodel-hermes) — server-side plugin
- [INTEGRATION.md](https://github.com/openyodel/plugin-openyodel-hermes/blob/main/INTEGRATION.md) — protocol reference
