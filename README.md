# wire-walkie-sdk

Shared client library for wire-walkie apps — platform-agnostic Yodel protocol implementation. Reference implementations in Swift, Kotlin, and TypeScript.

## Status

🚧 **No code yet — patterns are documented, extraction pending device validation.**

The first client ([wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift)) completed its MVP (M1–M6) in June 2026. The HTTP+SSE patterns have been reviewed and bugs found. Extraction happens once the app has passed real-device testing.

See [`LEARNINGS.md`](./LEARNINGS.md) for the 13 patterns discovered during the Swift MVP — these directly inform the SDK design and prevent the same bugs in Kotlin and TypeScript ports.

## Planned scope

- **Yodel HTTP client** — `POST /v1/chat/completions` with SSE stream parsing
- **Pairing** — Sound-QR device registration and per-device secret management
- **Session management** — persistent session IDs across requests (`X-Yodel-Session`)
- **TTS** — on-device text-to-speech with optional gateway audio fallback
- **Error handling** — retry, backoff, timeout policies
- **Configuration** — endpoint discovery via `/.well-known/yodel.json`

## Language targets

| Language | Status | First consumer |
|---|---|---|
| Swift | 🚧 extract after device validation | wire-walkie-swift (iOS) |
| Kotlin | 🔮 planned | wire-walkie-android |
| TypeScript | 🔮 planned | wire-walkie-web (PWA) |

## Key patterns (from LEARNINGS.md)

Before implementing any port, read [`LEARNINGS.md`](./LEARNINGS.md). The critical platform-agnostic ones:

| # | Pattern | Impact |
|---|---|---|
| L1 | Stateful `\n\n` SSE buffer, not line-split | Events get dropped otherwise |
| L2 | Check `[DONE]` before JSON parse | Stream error on every request otherwise |
| L4 | Accumulate bytes → decode UTF-8 on newline | Non-ASCII content corrupts otherwise |
| L5 | Two timeouts: idle 120s, total ∞ | Stream times out mid-response otherwise |
| L6 | Two HTTP clients: health vs. stream | One config can't serve both otherwise |

## Why not now?

The first client implements HTTP+SSE inline (~200 lines). Extracting before real-device validation is premature — the patterns are right, but edge cases only surface on hardware.

Once a second client needs the same logic, the shared code moves here.

## Related

- [wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift) — first client (iOS, MVP complete)
- [plugin-openyodel-hermes](https://github.com/openyodel/plugin-openyodel-hermes) — server-side Yodel plugin
- [INTEGRATION.md](https://github.com/openyodel/plugin-openyodel-hermes/blob/main/INTEGRATION.md) — protocol reference
