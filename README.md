# Voicewright

**Automated testing for voice AI — the way Playwright is for the web.**

Voicewright is an open-source testing framework for voice AI systems. It gives developers a reliable, repeatable way to test conversational flows, agent behaviour, latency, and semantic correctness across any voice AI platform.

> Built by [AI4E1](https://ai4e1.com) — the team building Page, Linda, and Ivy — because we felt this pain ourselves.

---

## The problem

Browser UIs have Playwright. Mobile apps have Appium. Voice AI has... manual call testing and hope.

Testing a voice AI system today means:
- Calling your agent by hand and listening to see if it sounds right
- No repeatable test runs across conversation branches
- No semantic assertions — just "does this feel correct?"
- No latency benchmarks baked into CI
- No coverage of multi-turn conversation state

Voicewright fixes this.

---

## What it does

```js
import { VoiceTest } from 'voicewright';

const call = await VoiceTest.startCall({ agent: 'ivy-morning-checkin' });

await call.userSays('I had a really rough night');
await call.expect().agentToAcknowledge('difficulty');

await call.userSays("I still want to do my workout though");
await call.expect().agentToTransitionTo('commitment-setting');
await call.expect().responseTime().toBeLessThan(1200); // ms

await call.end();
```

No audio files to manage. No manual listening. Just test code that runs in CI.

---

## Key features

**Synthetic callers** — Voicewright generates TTS audio from test utterances and injects it into your voice pipeline as if a real user spoke. Supports OpenAI TTS, ElevenLabs, and Google TTS out of the box.

**Semantic assertions** — LLM responses vary. Voicewright uses an assertion model to evaluate intent rather than exact strings. `expect().agentToSay('medication reminder')` passes even if the exact wording changes.

**Conversation flow testing** — Define multi-turn test scenarios as readable scripts. Branch across paths. Assert at any point in the conversation.

**Latency testing** — First byte, full response, and turn-around time are tracked per turn. Assert latency as part of your test suite.

**Platform-agnostic** — Works with Retell AI, Vapi, Twilio, ElevenLabs Conversational AI, and raw WebSocket-based agents. Bring your own adapter.

**CI-native** — Runs as a standard Node.js test suite. Integrates with Jest, Vitest, GitHub Actions, and any standard CI pipeline.

---

## Installation

```bash
npm install voicewright --save-dev
```

---

## Quick start

### 1. Configure your agent

```js
// voicewright.config.js
export default {
  agents: {
    'ivy-morning': {
      provider: 'retell',
      agentId: process.env.RETELL_AGENT_ID,
      apiKey: process.env.RETELL_API_KEY,
    }
  },
  tts: {
    provider: 'openai',
    voice: 'nova',
    apiKey: process.env.OPENAI_API_KEY,
  },
  assertionModel: {
    provider: 'openai',
    model: 'gpt-4o-mini',
  }
};
```

### 2. Write a test

```js
// tests/ivy-morning.test.js
import { VoiceTest } from 'voicewright';

describe('Ivy morning check-in', () => {

  test('acknowledges a hard night and pivots to commitment', async () => {
    const call = await VoiceTest.startCall({ agent: 'ivy-morning' });

    await call.userSays('Hey');
    await call.expect().agentToGreet();

    await call.userSays("I didn't sleep well, feeling pretty low");
    await call.expect().agentToAcknowledge('low energy or poor sleep');
    await call.expect().agentNotTo().pushCommitmentsImmediately();

    await call.userSays("I still want to try to exercise today");
    await call.expect().agentToTransitionTo('commitment-setting');
    await call.expect().responseTime().toBeLessThan(1500);

    await call.end();
  }, 30_000);

});
```

### 3. Run

```bash
npx voicewright test
```

---

## Assertion API

### Intent assertions

```js
await call.expect().agentToAcknowledge('topic or emotion');
await call.expect().agentToMention('keyword or concept');
await call.expect().agentToAsk('type of question'); // e.g. 'open-ended question'
await call.expect().agentToTransitionTo('conversation state');
await call.expect().agentToGreet();
await call.expect().agentToSummarise();
await call.expect().agentToConfirm('topic');
await call.expect().agentNotTo().mention('topic');
await call.expect().agentNotTo().askMoreThanOneQuestion();
```

### Latency assertions

```js
await call.expect().responseTime().toBeLessThan(1200); // ms
await call.expect().firstByteTime().toBeLessThan(500);
await call.expect().turnAroundTime().toBeLessThan(2000);
```

### State assertions

```js
await call.expect().conversationState().toBe('commitment-setting');
await call.expect().turnCount().toBeLessThan(10);
await call.expect().callDuration().toBeLessThan(300_000); // 5 mins
```

### Snapshot assertions

```js
await call.expect().semanticSnapshot('morning-greeting-v1');
// On first run: saves the semantic profile of the response
// On subsequent runs: asserts the response matches the saved profile
```

---

## Scenario scripting

For complex branching flows, use the scenario DSL:

```js
import { Scenario } from 'voicewright';

const lindalMedicationFlow = new Scenario('linda-medication-reminder')
  .step('opening', async (call) => {
    await call.userSays('Hello');
    await call.expect().agentToGreet();
  })
  .branch('user confirms taking medication', async (call) => {
    await call.userSays("Yes I took my tablets this morning");
    await call.expect().agentToAcknowledge('medication taken');
    await call.expect().agentNotTo().sendAlert();
  })
  .branch('user declines or is confused', async (call) => {
    await call.userSays("I can't remember if I took them");
    await call.expect().agentToPrompt('clarification');
    await call.expect().agentTo().logUncertainty();
  });

await lindaMedicationFlow.run({ agent: 'linda-pilot' });
```

---

## CI integration

### GitHub Actions

```yaml
name: Voice AI Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx voicewright test
        env:
          RETELL_API_KEY: ${{ secrets.RETELL_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

---

## Platform adapters

Voicewright ships with adapters for:

| Platform | Status |
|---|---|
| Retell AI | ✅ Supported |
| Vapi | ✅ Supported |
| ElevenLabs Conversational AI | 🚧 In progress |
| Twilio Voice + custom LLM | 🚧 In progress |
| Raw WebSocket agent | ✅ Supported |
| Bland AI | 📋 Planned |

Building your own? Implement the `VoiceAdapter` interface — it's small.

---

## Roadmap

- [ ] Core framework (synthetic caller, semantic assertions, latency tracking)
- [ ] Retell and Vapi adapters
- [ ] Scenario DSL
- [ ] CLI runner (`npx voicewright test`)
- [ ] Trace viewer (replay full call audio + assertion results)
- [ ] Coverage map (which conversation branches have been tested)
- [ ] Dashboard (hosted test history, latency trends, assertion pass rates)
- [ ] GitHub Action
- [ ] VS Code extension

---

## Contributing

Voicewright is early. We welcome contributors, especially those building on Retell, Vapi, or ElevenLabs who want first-class support.

- [Open an issue](https://github.com/ai4e1/voicewright/issues)
- [Read the contributing guide](./CONTRIBUTING.md)
- [Join the discussion](https://github.com/ai4e1/voicewright/discussions)

---

## Why we built this

We're AI4E1 — a Bristol-based AI company building three voice AI products:
- **Page** — AI learning companion for students, powered by voice
- **Linda** — proactive voice AI maintaining continuity with elderly patients
- **Ivy** — daily accountability coach via voice call

Testing all three manually at scale wasn't an option. So we built the thing that should have already existed.

If you're building voice AI, Voicewright will save you hours. And if you want to help shape the tool that becomes the standard for voice AI testing, get involved.

---

## License

MIT — free to use, build on, and contribute to.

---

*Built by [AI4E1 Ltd](https://ai4e1.com) · Bristol, UK*
