# wire-walkie-sdk — Implementation Learnings

> Patterns discovered building the first client ([wire-walkie-swift](https://github.com/moongrabber/wire-walkie-swift)).
> Documented here so Kotlin and TypeScript ports don't repeat the same bugs.
> These are platform-agnostic unless marked **Swift-only**.

---

## SSE Parsing

### L1 — Stateful `\n\n` buffer, never line-split

**The bug:** Using a line-splitting API (Swift `AsyncBytes.lines`, JS `ReadableStream` + `split('\n')`) splits on single `\n`. SSE events end on `\n\n`. A content chunk arrives as:
```
data: {"choices":[{"delta":{"content":"Hallo"}}]}\n
\n
```
Line-split sees two lines and processes them separately — the empty line triggers a premature flush before the data line is processed, or worse, silently drops the event.

**The fix:** Stateful accumulator. Collect lines until an empty line arrives, then process the buffered event as a unit. Reset buffer after each event.

**Applies to:** Swift, Kotlin, TypeScript — identical pattern in all three.

---

### L2 — Check `[DONE]` before JSON parse

**The bug:** `data: [DONE]` is the SSE stream terminator. It is not valid JSON. Passing it to `JSON.parse()` / `JSONDecoder` throws a decode error, which can surface as a false stream error or silent exception.

**The fix:** Check `line == "data: [DONE]"` (or `line.hasSuffix("[DONE]")`) before attempting JSON decode. Yield a `.done` event and finish the stream.

**Applies to:** All platforms.

---

### L3 — Skip keep-alive comment lines

**The fix:** Lines starting with `:` are SSE comments (e.g., `: ping`). Skip them silently — do not pass to the JSON decoder.

**Applies to:** All platforms.

---

## UTF-8 / Byte Handling

### L4 — Never treat raw bytes as characters

**The bug (Swift-specific, but the concept is universal):** Reading bytes one-by-one and casting each `UInt8` to a character (`UnicodeScalar(byte)`) treats every byte as a scalar. Multi-byte UTF-8 sequences (German umlauts: `ä ö ü`, emoji, any non-ASCII) get corrupted — each byte becomes a separate broken character.

**The fix:** Accumulate raw bytes into a buffer (`Data` in Swift, `ByteArray` in Kotlin, `Uint8Array` in TypeScript/JS). On newline detection (`byte == 0x0A`), decode the entire accumulated buffer as UTF-8. Only then interpret as a string.

```swift
// Wrong
let char = UnicodeScalar(byte)  // corrupts multi-byte sequences

// Right
lineData.append(byte)
if byte == 0x0A {
    let line = String(data: lineData, encoding: .utf8) ?? ""
}
```

**Applies to:** Any language doing byte-level SSE parsing with non-ASCII content.

---

## URLSession / HTTP Client

### L5 — Two timeouts, both explicit

**The bug:** HTTP clients have two independent timeouts: idle timeout (time between data chunks) and total resource timeout. Default idle timeouts (60s in `URLSession`, 30s in many HTTP clients) are too short for AI inference responses that may take 30–90 seconds before first token.

**The fix:** Set both explicitly:
- `timeoutIntervalForRequest` / idle timeout: **120 seconds** (covers slow inference)
- `timeoutIntervalForResource` / total timeout: **`.infinity`** (no limit on total stream duration)

Do not rely on defaults. Verify both are set in your HTTP client configuration.

| Platform | Idle timeout | Total timeout |
|---|---|---|
| Swift `URLSession` | `timeoutIntervalForRequest = 120` | `timeoutIntervalForResource = .infinity` |
| Kotlin OkHttp | `readTimeout(120, TimeUnit.SECONDS)` | no separate total limit needed |
| TypeScript `fetch` | `AbortSignal.timeout(120_000)` per-request | N/A (streams terminate naturally) |

**Applies to:** All platforms.

---

### L6 — Two URLSession/client instances: one for health, one for streaming

**Why:** Health checks need fast fail (5s request / 10s total). Streaming needs generous timeouts (120s idle / infinite total). Sharing one client means either health checks hang or streams time out.

**The fix:** Create two separate client instances at init time. Inject them via constructor for testability.

**Applies to:** All platforms.

---

## Audio / TTS (Swift-specific)

### L7 — Sentence-boundary buffer for streaming TTS

**The bug:** Queueing one `AVSpeechUtterance` per SSE content chunk (one per word/phrase) causes audible gaps between utterances. The synthesizer resets prosody on each utterance — output sounds robotic.

**The fix:** Buffer incoming text chunks. Flush to a single utterance when:
- A sentence-ending character arrives (`. ! ? … \n`)
- OR 400ms passes without a new chunk (timeout flush)

This produces natural-sounding output from a streaming source.

**Swift-only** — but the buffering pattern applies to any streaming TTS integration.

---

### L8 — `pendingUtterances` counter, not `isSpeaking: Bool`

**The bug:** `AVSpeechSynthesizer.isSpeaking` returns `false` between queued utterances. If you rely on it to detect "TTS done", you get false negatives mid-queue — the PTT state machine transitions to `idle` while more utterances are pending.

**The fix:** Maintain `private var pendingUtterances: Int`. Increment on each `speak()` call. Decrement in `didFinish` delegate. Fire `onDidFinish` callback only when counter reaches zero.

**Swift-only.**

---

## Concurrency / Thread Safety (Swift-specific)

### L9 — `AsyncThrowingStream`, never `AsyncStream`

**The bug:** `AsyncStream` cannot propagate errors. Mid-stream network failures are silently swallowed — the consumer sees the stream end normally, the PTT state machine transitions to `idle` instead of showing an error.

**The fix:** Always use `AsyncThrowingStream<SSEParser.Event, Error>` for `sendMessage()`. All error paths call `continuation.finish(throwing: error)`.

**Swift-only** (Kotlin Flow and JS AsyncIterator handle errors natively).

---

### L10 — Store audio tap input node to remove it later

**The bug:** Installing an `AVAudioEngine` tap on `inputNode` without storing the node reference makes `removeTap(onBus:)` impossible to call later. The second PTT press attempts to install a new tap on a node that already has one → runtime crash.

**The fix:** Store `private var inputNode: AVAudioInputNode?` in the actor. `stopRecording()` calls `inputNode?.removeTap(onBus: 0)` then sets `inputNode = nil` before stopping the engine.

**Swift-only.**

---

### L11 — NotificationCenter observer tokens must be stored and removed

**The bug:** `center.addObserver(forName:object:queue:using:)` returns an `NSObjectProtocol` token. Discarding it (`_ = center.addObserver(...)`) means the observer is never removed — memory leak, and handlers keep firing after teardown.

**The fix:** Store tokens in `nonisolated(unsafe) private var observers: [NSObjectProtocol] = []`. `stopObserving()` calls `center.removeObserver(token)` for each, then clears the array.

**Swift-only (conceptual equivalent exists in all platforms).**

---

## Testing

### L12 — `URLProtocol` subclass for mock SSE in unit tests

**The pattern:** Inject a custom `URLProtocol` subclass into a `URLSession` via `URLSessionConfiguration.protocolClasses`. The subclass serves pre-scripted SSE byte sequences. No network, no server, deterministic timing.

Key: set `nonisolated(unsafe) static var` properties on the mock class to configure response per-test. Register via `makeSession()` factory that sets `protocolClasses = [SSEProtocolMock.self]`.

**Swift-specific implementation, but the pattern (mock transport layer) applies everywhere.**

---

## Privacy / App Store (iOS-specific)

### L13 — `PrivacyInfo.xcprivacy` required since May 2024

Apps using microphone, `SFSpeechRecognizer`, or `UserDefaults` require a `PrivacyInfo.xcprivacy` file in the Xcode target. Missing it causes App Store rejection without appeal path. Declare all accessed API categories with reason codes.

Minimum for this app:
- `NSPrivacyCollectedDataTypeAudioData`
- `NSPrivacyAccessedAPICategoryUserDefaults` with reason `CA92.1`

**iOS-only.**

---

*Last updated: 2026-06-27 | Source: wire-walkie-swift MVP review (M1–M6, 20+ bugs across 5 PRs)*
