# Voicewright — Technical Specification

**Version:** 0.1 (Pre-build)
**Status:** Spec — awaiting implementation
**Owner:** AI4E1 Ltd
**Authors:** Michael Ojo, Victor Udedibor

---

## 1. Overview

Voicewright is an open-source testing framework for voice AI systems. It provides a Playwright-inspired API for writing automated tests against voice agents — covering conversation flow, semantic correctness, and latency — without any manual listening or call intervention.

The central insight driving the design: browser UIs expose an inspectable layer (the DOM) that testing frameworks can query and assert against. Voice AI currently has no equivalent. Voicewright creates one — a structured, assertable layer on top of a live voice call.

### 1.1 Core design goals

1. **Familiar API.** Developers who know Playwright should feel at home within minutes.
2. **Platform-agnostic.** The framework core is decoupled from any specific voice platform. Platform behaviour is encapsulated in adapters.
3. **Semantic-first.** Assertions evaluate meaning and intent, not exact string matches. LLM outputs are non-deterministic; the assertion layer accounts for this.
4. **CI-native.** Test runs are automated, reproducible, and integrate with standard CI pipelines.
5. **Open-source core, commercial runner.** The framework itself is MIT-licensed. A hosted runner with trace viewer and dashboard is the commercial layer (Phase 2).

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Test file (user-authored)                 │
│  VoiceTest.startCall() → call.userSays() → call.expect()...      │
└────────────────────────────┬─────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────┐
│                     Voicewright Core                             │
│                                                                  │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │  Call Session   │  │  Assertion Engine │  │  Trace Logger  │  │
│  │  Manager        │  │  (semantic +      │  │  (timing,      │  │
│  │                 │  │   latency)        │  │   transcript,  │  │
│  └────────┬────────┘  └────────┬─────────┘  │   audio)       │  │
│           │                   │             └────────────────┘  │
└───────────┼───────────────────┼──────────────────────────────────┘
            │                   │
┌───────────▼───────────────────▼──────────────────────────────────┐
│                      Platform Adapter Layer                       │
│                                                                   │
│   RetellAdapter │ VapiAdapter │ WebSocketAdapter │ TwilioAdapter  │
│                                                                   │
└───────────────────────────────┬──────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────┐
│                  External Services                               │
│                                                                  │
│   Voice Agent Platform    TTS Provider     Assertion Model        │
│   (Retell / Vapi / etc)  (OpenAI / EL)    (GPT-4o-mini)         │
└──────────────────────────────────────────────────────────────────┘
```

### 2.1 Component responsibilities

**Call Session Manager**
Orchestrates the lifecycle of a test call. Opens a connection via the configured adapter, manages turn state, sequences user utterances and agent responses, and exposes the fluent assertion API.

**Assertion Engine**
Receives agent response text (and optionally audio) and evaluates assertions. For semantic assertions, sends the assertion prompt and response to a configured LLM judge. For latency assertions, reads from the Trace Logger. Assertion failures throw structured errors with diff output.

**Trace Logger**
Records all events in a call: utterances (user + agent), audio, timestamps, latency metrics per turn, call duration, platform metadata. Written to disk as `.vwtrace` files for replay and debugging. Powers the trace viewer in the commercial dashboard.

**Platform Adapter**
Abstracts all platform-specific behaviour: call initiation, audio injection, transcript retrieval, event streaming. Each adapter implements the `VoiceAdapter` interface (see Section 5). Core never calls platform APIs directly.

---

## 3. Core API specification

### 3.1 VoiceTest

Top-level module. All test sessions start here.

```ts
class VoiceTest {
  static startCall(options: CallOptions): Promise<CallSession>
  static configure(config: VoicewrightConfig): void
}

interface CallOptions {
  agent: string                    // key matching config.agents
  context?: Record<string, any>   // optional context vars passed to agent
  timeout?: number                 // call-level timeout in ms (default: 60_000)
  tts?: Partial<TTSOptions>        // override global TTS for this call
}
```

### 3.2 CallSession

The primary object returned by `startCall()`. Wraps a live call.

```ts
class CallSession {
  // Utterance
  userSays(text: string, options?: UtteranceOptions): Promise<AgentResponse>

  // Assertions (chainable)
  expect(): AssertionChain

  // Helpers
  wait(ms: number): Promise<void>
  end(): Promise<CallSummary>
  transcript(): Transcript
  getTrace(): CallTrace
}

interface UtteranceOptions {
  voice?: string             // TTS voice override for this turn
  speed?: number             // TTS speed multiplier
  waitForResponse?: boolean  // default: true
  timeout?: number           // per-turn timeout override
}

interface AgentResponse {
  text: string
  audio?: Buffer
  latency: LatencyMetrics
  turnIndex: number
}

interface LatencyMetrics {
  firstByteMs: number
  fullResponseMs: number
  turnAroundMs: number    // from end of user utterance to start of agent response
}

interface CallSummary {
  durationMs: number
  turnCount: number
  passedAssertions: number
  failedAssertions: number
  transcript: Transcript
  traceFile: string
}
```

### 3.3 AssertionChain

Fluent assertion builder. All assertions are `async` — they make a judgment call or measure a metric.

```ts
class AssertionChain {
  // Semantic assertions (LLM-judged)
  agentToGreet(): Promise<void>
  agentToAcknowledge(topic: string): Promise<void>
  agentToMention(concept: string): Promise<void>
  agentToAsk(questionType: string): Promise<void>
  agentToTransitionTo(state: string): Promise<void>
  agentToConfirm(topic: string): Promise<void>
  agentToSummarise(): Promise<void>

  // Negation
  agentNotTo(): NegatedAssertionChain

  // Latency assertions (deterministic)
  responseTime(): LatencyAssertion
  firstByteTime(): LatencyAssertion
  turnAroundTime(): LatencyAssertion

  // State assertions
  conversationState(): StateAssertion
  turnCount(): NumericAssertion
  callDuration(): NumericAssertion

  // Snapshot
  semanticSnapshot(snapshotId: string): Promise<void>
}

class LatencyAssertion {
  toBeLessThan(ms: number): Promise<void>
  toBeGreaterThan(ms: number): Promise<void>
  toBeBetween(minMs: number, maxMs: number): Promise<void>
}

class NegatedAssertionChain {
  mention(concept: string): Promise<void>
  askMoreThanOneQuestion(): Promise<void>
  pushCommitmentsImmediately(): Promise<void>
  // mirrors AssertionChain — all methods negated
}
```

### 3.4 Scenario DSL

For multi-branch, multi-turn scenario definition.

```ts
class Scenario {
  constructor(name: string)

  step(name: string, fn: (call: CallSession) => Promise<void>): Scenario
  branch(description: string, fn: (call: CallSession) => Promise<void>): Scenario
  run(options: CallOptions): Promise<ScenarioResult>
}

interface ScenarioResult {
  name: string
  steps: StepResult[]
  branches: BranchResult[]
  passed: boolean
  durationMs: number
}
```

---

## 4. Semantic assertion engine

### 4.1 The problem

LLM-generated responses are non-deterministic. Asserting `response.text === 'Good morning'` is fragile — the agent might say "Morning!", "Hi there!", or "Hello! How are you feeling today?" All are valid greetings; none match the literal string.

The assertion engine solves this by using a small LLM (the "judge") to evaluate whether a response satisfies a semantic intent.

### 4.2 How it works

When `expect().agentToAcknowledge('difficulty sleeping')` is called:

1. The assertion engine retrieves the last agent response text.
2. It constructs a judge prompt:

```
You are evaluating a voice AI agent's response for a specific intent.

Agent response: "{agentResponseText}"

Question: Does this response acknowledge that the user is having difficulty sleeping?

Respond ONLY with a JSON object: { "pass": true|false, "reason": "brief explanation" }
```

3. The judge model returns `{ "pass": true, "reason": "Agent says 'sounds like you had a tough night' which acknowledges the difficulty" }`.
4. If `pass: false`, the assertion throws with the reason included in the error message.

### 4.3 Judge model configuration

```ts
interface AssertionModelConfig {
  provider: 'openai' | 'anthropic' | 'custom'
  model: string                  // e.g. 'gpt-4o-mini', 'claude-haiku-4-5'
  apiKey?: string                // falls back to env var
  temperature?: number           // default: 0 (maximise determinism)
  maxRetries?: number            // default: 2
  fallbackToKeyword?: boolean    // if judge fails, fall back to keyword match
}
```

### 4.4 Determinism and retry strategy

Semantic assertions can themselves be non-deterministic — the judge model might assess ambiguous responses differently across runs. Mitigations:

- Judge temperature is set to 0 by default.
- Assertions retry up to `maxRetries` times on a `false` result before failing. This handles edge cases where a valid response is initially misclassified.
- `fallbackToKeyword: true` uses simple substring matching as a last resort when the judge model is unavailable.

### 4.5 Semantic snapshots

`expect().semanticSnapshot('snapshot-id')` creates a semantic fingerprint of a response:

- **First run:** calls judge with prompt `"Describe the intent and key content of this response in 3-5 bullet points"`. Saves result as `snapshots/snapshot-id.json`.
- **Subsequent runs:** calls judge with prompt `"Does this new response match the intent described in this profile? {savedProfile}"`. Asserts `pass: true`.

Snapshots can be reviewed and committed to version control. This creates a form of semantic regression testing.

---

## 5. Platform adapter interface

### 5.1 VoiceAdapter interface

All adapters implement this interface. The core framework calls only these methods.

```ts
interface VoiceAdapter {
  // Open a call to the configured agent. Returns when the agent is ready.
  connect(config: AgentConfig, context?: Record<string, any>): Promise<void>

  // Inject audio representing the user utterance. Returns the agent's audio response.
  sendUtterance(audio: Buffer): Promise<AdapterResponse>

  // Get the current transcript (all turns so far)
  getTranscript(): Transcript

  // Get latency metrics for the most recent turn
  getLastTurnLatency(): LatencyMetrics

  // Terminate the call cleanly
  disconnect(): Promise<void>
}

interface AdapterResponse {
  text: string
  audio?: Buffer
  rawEvent?: unknown   // platform-specific event payload, preserved for trace
}

interface AgentConfig {
  agentId: string
  apiKey: string
  [key: string]: unknown   // adapter-specific extra config
}

interface Transcript {
  turns: Turn[]
}

interface Turn {
  role: 'user' | 'agent'
  text: string
  audio?: Buffer
  timestamp: number
  latency?: LatencyMetrics
}
```

### 5.2 Retell adapter

```ts
class RetellAdapter implements VoiceAdapter {
  // Uses Retell's Web Call API to open a call session.
  // Injects TTS-generated audio via the WebRTC channel.
  // Reads agent responses from the Retell transcript WebSocket event stream.
  // Extracts latency from Retell's response_start_timestamp and response_end_timestamp.
}
```

### 5.3 Vapi adapter

```ts
class VapiAdapter implements VoiceAdapter {
  // Uses Vapi's /call/web endpoint.
  // Injects audio via the Vapi WebSocket stream.
  // Reads agent responses from the vapi-message event stream.
}
```

### 5.4 Raw WebSocket adapter

For custom agents not on a managed platform.

```ts
class WebSocketAdapter implements VoiceAdapter {
  // Connects to a WebSocket URL.
  // Sends audio as binary frames.
  // Reads text responses from message events.
  // Users configure message schema via WebSocketAdapterConfig.
}

interface WebSocketAdapterConfig extends AgentConfig {
  websocketUrl: string
  messageSchema: {
    utteranceEvent: string       // event name/type to send
    responseEvent: string        // event name/type to listen for
    textField: string            // field path in response JSON containing agent text
  }
}
```

---

## 6. TTS layer

Voicewright generates synthetic user utterances using a configured TTS provider. This audio is injected into the call as if a real user spoke.

### 6.1 TTS provider interface

```ts
interface TTSProvider {
  synthesise(text: string, options: TTSOptions): Promise<Buffer>
}

interface TTSOptions {
  voice: string
  speed?: number           // 0.5–2.0
  format?: 'pcm' | 'mp3' | 'wav'  // default: pcm (most platforms need raw PCM)
  sampleRate?: number      // default: 16000
}
```

### 6.2 Supported providers (Phase 1)

| Provider | Notes |
|---|---|
| OpenAI TTS | Default. Fast, low latency, reliable. |
| ElevenLabs | Higher quality, supports more voices. Higher cost. |
| Google TTS | Good multilingual support. |
| Custom | Implement `TTSProvider` and register it in config. |

### 6.3 Voice consistency

For tests that assert on acoustic properties (future — see Section 9.4), the same voice should be used consistently across a test suite. The config-level `tts.voice` is the default. Per-call and per-utterance overrides are available but should be used sparingly.

---

## 7. Configuration

### 7.1 Config file

`voicewright.config.js` (or `.ts`) in project root. Loaded automatically by the CLI.

```ts
interface VoicewrightConfig {
  agents: Record<string, AgentConfig & { provider: string }>
  tts: TTSOptions & { provider: string; apiKey?: string }
  assertionModel: AssertionModelConfig
  trace?: {
    outputDir?: string         // default: './voicewright-traces'
    keepAudio?: boolean        // default: false (audio adds size)
    retainDays?: number        // auto-delete traces older than N days
  }
  defaults?: {
    callTimeout?: number       // ms, default: 60_000
    turnTimeout?: number       // ms, default: 15_000
    retries?: number           // per-assertion retry count, default: 2
  }
}
```

### 7.2 Environment variables

All API keys can be provided via env vars to avoid committing secrets:

```
VOICEWRIGHT_RETELL_API_KEY
VOICEWRIGHT_VAPI_API_KEY
VOICEWRIGHT_OPENAI_API_KEY
VOICEWRIGHT_ELEVENLABS_API_KEY
VOICEWRIGHT_ASSERTION_MODEL_KEY
```

---

## 8. CLI

```bash
# Run all tests
npx voicewright test

# Run specific file
npx voicewright test tests/ivy-morning.test.js

# Run with verbose output
npx voicewright test --verbose

# List traces
npx voicewright traces list

# Open trace viewer (Phase 2 — hosted)
npx voicewright traces view --id <trace-id>

# Validate config
npx voicewright config check

# Initialise project
npx voicewright init
```

---

## 9. Development phases

### Phase 1 — Core framework (MVP)

**Goal:** A working framework that can run test files and assert against Retell and Vapi agents.

**Deliverables:**
- Call session manager
- `userSays()` + TTS injection (OpenAI TTS)
- Semantic assertion engine (GPT-4o-mini judge)
- Latency assertions
- Retell adapter
- Vapi adapter
- CLI runner (`voicewright test`)
- Trace logger (text only, no audio)
- Config file loading
- Jest integration

**Estimated build:** 4–6 weeks (solo developer with AI-assisted build)

**Stack:** TypeScript, Node.js 20+, Jest, ws (WebSocket), OpenAI SDK

### Phase 2 — Scenario DSL + ElevenLabs + Trace viewer

**Goal:** Richer test authoring and debugging visibility.

**Deliverables:**
- Scenario DSL (`Scenario` class)
- ElevenLabs and Twilio adapters
- Semantic snapshots
- Trace viewer (local HTML report)
- Vitest integration
- GitHub Action

**Estimated build:** 6–8 weeks

### Phase 3 — Hosted runner (commercial layer)

**Goal:** Cloud-based test runner with persistent history, latency trend dashboards, and team collaboration.

**Deliverables:**
- Hosted runner service (serverless, per-run billing)
- Dashboard: test history, latency trends, assertion pass rates
- Call audio replay (with assertion overlaid at each turn)
- Branch coverage map across conversation flows
- Team workspaces and shared snapshots
- Slack and GitHub PR notifications

**Pricing model:** Free tier (100 runs/month). Pro ($49/month, 1,000 runs). Team ($199/month, unlimited).

---

## 10. Non-goals (Phase 1)

The following are explicitly out of scope for the initial build:

- Real-time audio analysis (vocal biomarkers, tone detection) — this is a research problem
- Visual testing of voice agent UIs (though a Playwright plugin for hybrid voice+web flows is a Phase 3 idea)
- Load testing / stress testing at scale — a separate tool concern
- Any proprietary assertion logic — the judge model integration is the full semantic layer in Phase 1
- On-device / edge TTS — cloud TTS only in Phase 1

---

## 11. Open questions

These require decisions before Phase 1 build begins:

1. **Test runner coupling:** Should Voicewright ship its own runner or lean fully on Jest/Vitest? Leaning on existing runners reduces build effort and leverages ecosystem tooling, but a custom runner unlocks better trace integration.

2. **Call concurrency:** Should Voicewright support running multiple test calls in parallel? This reduces total test time but increases platform API usage and risk of rate limiting. Recommend: sequential by default, `--workers N` flag for parallel.

3. **Assertion model cost:** At 2 LLM judge calls per assertion × 10 assertions × 50 tests, that's 1,000 judge calls per full suite. At GPT-4o-mini pricing, negligible. But teams should be made aware. Consider a `--dry-run` mode that validates test structure without making judge calls.

4. **Audio injection method:** Retell's web call API accepts audio via WebRTC. WebRTC in a Node.js test environment is non-trivial — requires a headless WebRTC implementation. Evaluate: `node-webrtc`, `wrtc`, or a server-side injection API if Retell exposes one. This is the highest-risk technical uncertainty in Phase 1.

5. **Snapshot storage:** Should semantic snapshots be committed to version control (simple, transparent) or stored in a cloud snapshot registry (enables cross-team sharing and history)? Recommend: filesystem in Phase 1, cloud registry in Phase 3.

---

## 12. Repository structure (target)

```
voicewright/
├── packages/
│   ├── core/                  # VoiceTest, CallSession, assertions, trace logger
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── session.ts
│   │   │   ├── assertions/
│   │   │   │   ├── semantic.ts
│   │   │   │   ├── latency.ts
│   │   │   │   └── snapshot.ts
│   │   │   ├── trace.ts
│   │   │   └── scenario.ts
│   │   └── package.json
│   ├── adapters/
│   │   ├── retell/
│   │   ├── vapi/
│   │   └── websocket/
│   └── tts/
│       ├── openai/
│       ├── elevenlabs/
│       └── google/
├── cli/                       # CLI entrypoint
├── docs/                      # Documentation site (Docusaurus)
├── examples/                  # Example projects (Retell, Vapi, raw WS)
├── voicewright.config.example.js
└── README.md
```

**Monorepo tooling:** pnpm workspaces + Turborepo.

---

## 13. Competitive landscape

| Tool | Voice-native | Semantic assertions | Multi-turn | CI-native | Open source |
|---|---|---|---|---|---|
| **Voicewright** | ✅ | ✅ | ✅ | ✅ | ✅ |
| Botium | ❌ (text) | ❌ | ✅ | ✅ | ✅ |
| Cyara | ✅ | ❌ | ✅ | ✅ | ❌ |
| Hammer (Empirix) | ✅ | ❌ | Partial | ✅ | ❌ |
| Manual testing | ✅ | Sort of | ✅ | ❌ | — |

No current tool is voice-native, semantically aware, and open-source. That is Voicewright's position.

---

## 14. Naming and brand

**Name:** Voicewright
**Rationale:** Echoes Playwright (the inspiration). "-wright" means a maker or builder — a wheelwright makes wheels, a playwright makes plays, a voicewright makes voice tests.
**npm package:** `voicewright`
**GitHub org:** `ai4e1` (existing), repo: `voicewright`
**Domain:** `voicewright.dev` (to be registered)
**Tagline:** *Automated testing for voice AI — the way Playwright is for the web.*

---

*Voicewright is a project of AI4E1 Ltd, Bristol, UK.*
*Specification status: pre-build. Subject to revision before Phase 1 kickoff.*
