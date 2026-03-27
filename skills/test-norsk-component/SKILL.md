---
name: Test Norsk Studio Component
description: |
  Reference guide for writing component tests using the @norskvideo/norsk-studio shared test libraries.
  Covers the full API surface of test utilities, helpers, and patterns for testing input, processor,
  and output components in any workspace that depends on @norskvideo/norsk-studio.

  Trigger this skill when the user says:
  - "Write tests for my component..."
  - "How do I test a Norsk Studio component?"
  - "Help me write tests for..."
  - "What test helpers are available?"
  - "Test my processor/input/output component"
---

# Testing Norsk Studio Components

This guide covers the shared test libraries published in the `@norskvideo/norsk-studio` npm package.
All utilities are available as compiled JS + type declarations under `lib/`.

## Overview

Every component test follows this flow:

1. Create a runtime and register your components
2. Build a YAML document describing the test workflow
3. Connect to Norsk and execute the document
4. Create test sources and wire them to your component
5. Assert on outputs, state, events, and/or API responses
6. Clean up

## Import Reference

### Test Utilities (from `@norskvideo/norsk-studio`)

```typescript
// Document building
import { YamlBuilder, YamlNodeBuilder, emptyRuntime } from "@norskvideo/norsk-studio/lib/test/_util/builder";

// Execution engine - runs a compiled document, returns RunResult
import go, { RunResult, StudioNodeSubscriptionSource, ComponentSubscriptions } from "@norskvideo/norsk-studio/lib/runtime/execution";

// Document compilation - parses YAML into a CompiledDocument
import * as document from "@norskvideo/norsk-studio/lib/runtime/document";

// Test media sources - generate video/audio test signals via Norsk SDK
import { video, audio, videoAndAudio, videoAndAudioAndSubs, testSourceDescription, testSourceDescriptionWithSubs } from "@norskvideo/norsk-studio/lib/test/_util/sources";

// Assertions and test sinks
import { assertNodeOutputsVideoFrames, assertNodeOutputsAudioFrames, waitForAssert, TraceSink } from "@norskvideo/norsk-studio/lib/test/_util/sinks";

// Pre-registered test components (AvInput is the most commonly used)
import { testRuntime, AvInput, AvOutput, AudioInput, AudioOutput, AvProcessor } from "@norskvideo/norsk-studio/lib/test/_util/runtime";

// Async waiting utilities
import { waitForCondition, waitForEvent } from "@norskvideo/norsk-studio/lib/shared/util";

// Extension types
import { RegistrationConsts, BaseConfig, NodeInfo, Av, All } from "@norskvideo/norsk-studio/lib/extension/client-types";
import { RuntimeSystem } from "@norskvideo/norsk-studio/lib/extension/runtime-system";
import { SimpleSinkWrapper, SimpleProcessorWrapper, SimpleInputWrapper } from "@norskvideo/norsk-studio/lib/extension/base-nodes";

// FFmpeg helpers - spawn ffmpeg to send test streams to input components
import { Ffmpeg, testCard, silence, singleH264AacEncode, rtmpOutput, srtOutput, rtpOutput } from "@norskvideo/norsk-studio/lib/test/_util/ffmpeg";
```

### Your Workspace Imports

```typescript
// Norsk SDK
import { Norsk } from "@norskvideo/norsk-sdk";

// Standard libraries
import YAML from "yaml";
import { expect } from "chai";
import express from "express";
import { Server } from "http";
import { AddressInfo } from "net";

// Your component's info function (takes Registration, returns component definition)
import MyComponentInfo from "../processor.myComponent/info";

// Your component's runtime types
import { MyComponent, MyComponentConfig, MyComponentState, MyComponentCommand, MyComponentEvent } from "../processor.myComponent/runtime";

// Your workspace's registerAll function (registers all components with the runtime)
import registerAll from "..";
```

---

## Core API Reference

### YamlBuilder

Builds a workflow document programmatically.

```typescript
const yaml = new YamlBuilder();

// Add component nodes
yaml.addNode(nodeBuilder.reify());

// Set global config values
yaml.setGlobalValue('key', value);

// Finalise the document
const document = yaml.reify();
```

### YamlNodeBuilder

Builds a single component node. Generic type parameters are `<Config, State, Command, Event>`.

```typescript
const node = new YamlNodeBuilder<MyConfig, MyState, MyCommand, MyEvent>(
  'node-id',                             // unique ID for this node
  MyComponentInfo(RegistrationConsts),   // component info object
  { /* config without id/displayName */ } // config (id + displayName auto-filled)
);

// Subscribe to another node's output
node.subscribe("source-node-id", ["video", "audio"]);

// Access the underlying node object for config tweaks
node.node.config = { ...node.node.config, someField: 'override' };

// Finalise
const yamlNode = node.reify();
```

### document.load

Compiles a YAML string into a `CompiledDocument` ready for execution.

```typescript
const compiled = document.load(
  __filename,        // used for relative path resolution
  runtime,           // RuntimeSystem with registered components
  YAML.stringify(yaml.reify()),
  { resolveConfig: true }  // optional: resolve config defaults
);
```

### go (execution)

Executes a compiled document, creating all components and wiring subscriptions.

```typescript
// Without HTTP API
const result = await go(norsk, compiled);

// With HTTP API (pass an Express app to enable component routes)
const result = await go(norsk, compiled, expressApp);
```

### RunResult

Returned by `go()`. Key fields:

| Field | Type | Description |
|-------|------|-------------|
| `components` | `{ [id]: CreatedMediaNode }` | Component instances by ID |
| `runtimeState` | `RuntimeState` | State management |
| `runtimeState.getNodeState(id)` | function | Get current state for a node |
| `runtimeState.latest` | `{ [id]: State }` | Direct state access |
| `runtimeState.events` | EventEmitter | Listen for events/state updates |
| `document` | `CompiledDocument` | The compiled document with definitions |
| `nodes` | `{ [id]: CreatedNode }` | All nodes including metadata |
| `overrideSources(target, sources)` | function | Replace a component's input sources |
| `getSubscriptions(target)` | function | Get a component's ComponentSubscriptions |

---

## Test Source Helpers

All from `@norskvideo/norsk-studio/lib/test/_util/sources`. These create real Norsk SDK media nodes that produce test signals.

| Function | Description |
|----------|-------------|
| `video(norsk, id, opts?)` | Video test card |
| `audio(norsk, id, opts?)` | Audio signal generator |
| `videoAndAudio(norsk, id, opts?)` | Combined A/V |
| `videoAndAudioAndSubs(norsk, id, opts?)` | A/V + WebVTT subtitles |
| `testSourceDescription(id?, output?)` | Creates a `NodeDescription` for use with `StudioNodeSubscriptionSource` |
| `testSourceDescriptionWithSubs(id?, output?)` | Same, including subtitle stream info |

**Source options:**
```typescript
{
  resolution?: { width: number, height: number },  // default varies
  frameRate?: { frames: number, seconds: number },  // e.g. { frames: 25, seconds: 1 }
  sampleRate?: number,          // default 48000
  channelLayout?: string,       // 'stereo', 'mono', '5.1'
  sourceName?: string,          // stream key source name
  renditionName?: string,       // stream key rendition name
  streamId?: number,
  programNumber?: number
}
```

**TestSourceWrapper** (returned by all source helpers):
- Extends `SimpleInputWrapper` (implements `CreatedMediaNode`)
- `pause()` / `play()` - stop/resume output
- `close()` - destroy the source
- `media: Media[]` - array of media types produced
- `initialised: Promise<void>` - resolves when ready

### AvInput (test component)

An alternative to the source helpers. `AvInput` is a pre-registered test component that produces A/V.
Add it to your YAML document as a node, and it will produce test streams when the workflow runs.

```typescript
yaml.addNode(new YamlNodeBuilder('source', AvInput.info, { sourceName: true }).reify());
```

Use AvInput when you want subscriptions wired via YAML. Use the source helpers when you want
to programmatically control sources via `overrideSources` or `ComponentSubscriptions`.

---

## Assertion Helpers

All from `@norskvideo/norsk-studio/lib/test/_util/sinks`.

### assertNodeOutputsVideoFrames

Waits for at least 5 video frames from a component. Fails if no frames arrive within timeout.

```typescript
// Basic check
await assertNodeOutputsVideoFrames(norsk, result, 'node-id');

// With resolution assertion
await assertNodeOutputsVideoFrames(norsk, result, 'node-id', {
  resolution: { width: 1920, height: 1080 }
});

// Match a specific stream by key (for multi-output components like encode ladders)
await assertNodeOutputsVideoFrames(norsk, result, 'node-id', {
  match: { renditionName: 'h264_640x360' },
  resolution: { width: 640, height: 360 }
});
```

**Match options:** `{ renditionName?, streamId?, sourceName?, programNumber? }`

### assertNodeOutputsAudioFrames

Same pattern for audio:

```typescript
await assertNodeOutputsAudioFrames(norsk, result, 'node-id');
await assertNodeOutputsAudioFrames(norsk, result, 'node-id', {
  match: { renditionName: 'default' }
});
```

### waitForAssert

Polls a condition function. When it returns true (or on timeout), runs the assertion function.

```typescript
await waitForAssert(
  () => someCondition(),    // condition: () => boolean | Promise<boolean>
  () => {                   // assert: () => void (throw on failure)
    expect(x).to.equal(y);
  },
  10000,                    // timeout ms (default 10000)
  10                        // poll interval ms (default 10)
);
```

### TraceSink

A test sink that subscribes to streams and counts incoming frames per stream key.

```typescript
const sink = new TraceSink(norsk, "test-sink");
await sink.initialised;

// Wire it using ComponentSubscriptions
const subs = new ComponentSubscriptions(norsk, sink);
subs.setSources([
  new StudioNodeSubscriptionSource(
    sourceNode,
    compiled.components['node-id'].yaml,
    { type: 'take-all-streams', filter: Av.map((media) => ({ media })) },
    MyComponentInfo(RegistrationConsts) as unknown as NodeInfo<BaseConfig>
  )
]);

// Inspect results
await waitForCondition(() => sink.streamCount() == 2 && sink.totalMessages() > 10, 30000, 50);

sink.streamCount();                                    // number of unique streams
sink.totalMessages();                                  // total frame count across all streams
sink.messagesForKey(k => k.sourceName == 'foo');       // frame count for matching keys
sink.streams();                                        // metadata of subscribed sources
```

---

## Async Waiting Utilities

From `@norskvideo/norsk-studio/lib/shared/util`.

### waitForCondition

Polls until condition is true. Rejects on timeout.

```typescript
await waitForCondition(
  () => component.isReady,        // condition: () => boolean | Promise<boolean>
  5000,                           // timeout ms (default 10000)
  100,                            // poll interval ms (default 10)
  'waiting for component ready'   // description (shown on timeout error)
);
```

### waitForEvent

Waits for a specific runtime event from a component. Returns the event object.

```typescript
// IMPORTANT: start listening BEFORE triggering the action that causes the event
const eventPromise = waitForEvent<MyComponentEvent>(
  result,              // RunResult
  'node-id',           // component node ID
  'source-connected',  // event type to wait for
  10000                // timeout ms (default 5000)
);

// Now trigger the action
triggerSomething();

// Wait for and inspect the event
const event = await eventPromise;
expect(event.type).equals('source-connected');
```

---

## Subscription Configuration

When creating `StudioNodeSubscriptionSource`, the third argument is a `SubscriptionConfiguration`:

```typescript
// Subscribe to all matching streams
{ type: 'take-all-streams', filter: [{ media: 'video' }, { media: 'audio' }] }

// Subscribe to first matching stream only
{ type: 'take-first-stream', filter: [{ media: 'video' }, { media: 'audio' }] }

// Using the Av helper constant for video + audio
{ type: 'take-all-streams', filter: Av.map((media) => ({ media })) }

// Video only
{ type: 'take-all-streams', filter: [{ media: 'video' }] }
```

---

## Wiring Sources to Components

There are two approaches:

### Approach 1: YAML Subscriptions (via AvInput test component)

Best for: testing the full subscription pipeline as it runs in production.

```typescript
const yaml = new YamlBuilder();

// Add AvInput test source
yaml.addNode(new YamlNodeBuilder('source', AvInput.info, { sourceName: true }).reify());

// Add your component, subscribed to the source
const myNode = new YamlNodeBuilder('my-component', MyInfo(RegistrationConsts), config);
myNode.subscribe("source", ["video", "audio"]);
yaml.addNode(myNode.reify());
```

### Approach 2: overrideSources / ComponentSubscriptions

Best for: testing source changes, reconnection, dynamic subscriptions.

```typescript
// After go(), create a source and wire it programmatically
const source = await videoAndAudio(norsk, 'source', { sourceName: 'my-source' });
const component = result.components['my-component'];

// Option A: overrideSources (simpler)
result.overrideSources(component, [
  new StudioNodeSubscriptionSource(
    source,
    testSourceDescription(),
    { type: 'take-all-streams', filter: [{ media: 'video' }, { media: 'audio' }] }
  )
]);

// Option B: ComponentSubscriptions (more control)
const subs = new ComponentSubscriptions(norsk, component);
subs.setSources([
  new StudioNodeSubscriptionSource(
    source,
    testSourceDescription(),
    { type: 'take-all-streams', filter: [{ media: 'video' }, { media: 'audio' }] },
    MyComponentInfo(RegistrationConsts) as unknown as NodeInfo<BaseConfig>
  )
]);

// To change sources later (test reconnection):
await source.close();
result.overrideSources(component, []);  // or subs.setSources([])

const source2 = await videoAndAudio(norsk, 'source2');
result.overrideSources(component, [
  new StudioNodeSubscriptionSource(source2, testSourceDescription(), /* ... */)
]);
```

---

## FFmpeg Test Utilities

For input components that receive external streams (RTMP, SRT, RTP):

```typescript
import { Ffmpeg, testCard, singleH264AacEncode, rtmpOutput, srtOutput } from "@norskvideo/norsk-studio/lib/test/_util/ffmpeg";

// Send an RTMP stream to your input component
const ffmpeg = await Ffmpeg.create({
  sources: testCard(),
  encode: singleH264AacEncode({ duration: 10, resolution: { width: 1920, height: 1080 } }),
  transport: rtmpOutput({ port: 1935, app: 'live', str: 'stream1' })
});

// Send an SRT stream
const ffmpeg = await Ffmpeg.create({
  sources: testCard(),
  encode: singleH264AacEncode({ duration: 10 }),
  transport: srtOutput({ port: 9000, mode: 'caller' })
});

// Stop when done
await ffmpeg.stop();
```

**Transport helpers:** `rtmpOutput({ port, app?, str? })`, `srtOutput({ port, streamId?, mode? })`, `rtpOutput({ port })`, `udpOut({ port })`, `mp4Output(filename)`

**Source helpers:** `testCard(spec?)` (video+audio), `silence(spec?)` (audio only), `blankFrame(settings?)` (black video)

---

## Complete Test Templates

### Input Component

```typescript
import { Norsk } from "@norskvideo/norsk-sdk";
import { YamlBuilder, YamlNodeBuilder, emptyRuntime } from "@norskvideo/norsk-studio/lib/test/_util/builder";
import * as document from "@norskvideo/norsk-studio/lib/runtime/document";
import go, { RunResult } from "@norskvideo/norsk-studio/lib/runtime/execution";
import { assertNodeOutputsVideoFrames, assertNodeOutputsAudioFrames } from "@norskvideo/norsk-studio/lib/test/_util/sinks";
import { waitForCondition, waitForEvent } from "@norskvideo/norsk-studio/lib/shared/util";
import { RegistrationConsts } from "@norskvideo/norsk-studio/lib/extension/client-types";
import { RuntimeSystem } from "@norskvideo/norsk-studio/lib/extension/runtime-system";
import registerAll from "..";
import MyInputInfo from "../input.myInput/info";
import type { MyInputConfig, MyInputState, MyInputEvent } from "../input.myInput/runtime";
import { expect } from "chai";
import YAML from "yaml";

async function defaultRuntime(): Promise<RuntimeSystem> {
  const runtime = emptyRuntime();
  await registerAll(runtime);
  return runtime;
}

describe("My Input Component", () => {
  let norsk: Norsk | undefined;
  let result: RunResult | undefined;

  afterEach(async () => {
    await norsk?.close();
    norsk = undefined;
    result = undefined;
  });

  async function testDocument(overrides?: Partial<MyInputConfig>) {
    const runtime = await defaultRuntime();
    const yaml = new YamlBuilder()
      .addNode(
        new YamlNodeBuilder<MyInputConfig, MyInputState, object, MyInputEvent>(
          'input', MyInputInfo(RegistrationConsts), { port: 9999, ...overrides }
        ).reify()
      )
      .reify();
    return document.load(__filename, runtime, YAML.stringify(yaml));
  }

  it("produces video and audio", async () => {
    norsk = await Norsk.connect({ onShutdown: () => {} });
    const compiled = await testDocument();
    result = await go(norsk, compiled);

    // Send a test stream to the input (if it receives external streams)
    // const ffmpeg = await Ffmpeg.create({ ... });

    await Promise.all([
      assertNodeOutputsVideoFrames(norsk, result, 'input'),
      assertNodeOutputsAudioFrames(norsk, result, 'input')
    ]);

    // await ffmpeg.stop();
  });

  it("emits connection events", async () => {
    norsk = await Norsk.connect({ onShutdown: () => {} });
    const compiled = await testDocument();
    result = await go(norsk, compiled);

    const eventPromise = waitForEvent<MyInputEvent>(result, 'input', 'source-connected', 10000);

    // Trigger the connection...

    const event = await eventPromise;
    expect(event.type).equals('source-connected');
  });
});
```

### Processor Component

```typescript
import { Norsk } from "@norskvideo/norsk-sdk";
import { YamlBuilder, YamlNodeBuilder, emptyRuntime } from "@norskvideo/norsk-studio/lib/test/_util/builder";
import * as document from "@norskvideo/norsk-studio/lib/runtime/document";
import go, { RunResult, StudioNodeSubscriptionSource } from "@norskvideo/norsk-studio/lib/runtime/execution";
import { assertNodeOutputsVideoFrames, assertNodeOutputsAudioFrames, waitForAssert } from "@norskvideo/norsk-studio/lib/test/_util/sinks";
import { videoAndAudio, testSourceDescription } from "@norskvideo/norsk-studio/lib/test/_util/sources";
import { AvInput } from "@norskvideo/norsk-studio/lib/test/_util/runtime";
import { waitForCondition } from "@norskvideo/norsk-studio/lib/shared/util";
import { RegistrationConsts } from "@norskvideo/norsk-studio/lib/extension/client-types";
import { RuntimeSystem } from "@norskvideo/norsk-studio/lib/extension/runtime-system";
import registerAll from "..";
import MyProcessorInfo from "../processor.myProcessor/info";
import type { MyProcessor, MyProcessorConfig, MyProcessorState, MyProcessorCommand, MyProcessorEvent } from "../processor.myProcessor/runtime";
import { expect } from "chai";
import YAML from "yaml";

async function defaultRuntime(): Promise<RuntimeSystem> {
  const runtime = emptyRuntime();
  await registerAll(runtime);
  return runtime;
}

describe("My Processor", () => {
  let norsk: Norsk | undefined;
  let result: RunResult | undefined;

  afterEach(async () => {
    await norsk?.close();
    norsk = undefined;
    result = undefined;
  });

  // Helper to access component state
  const latestState = () => result?.runtimeState.getNodeState('processor') as (MyProcessorState | undefined);

  async function testDocument(overrides?: Partial<MyProcessorConfig>) {
    const runtime = await defaultRuntime();
    const yaml = new YamlBuilder();

    yaml.addNode(new YamlNodeBuilder('source', AvInput.info, { sourceName: true }).reify());

    const proc = new YamlNodeBuilder<MyProcessorConfig, MyProcessorState, MyProcessorCommand, MyProcessorEvent>(
      'processor', MyProcessorInfo(RegistrationConsts), { /* defaults */ ...overrides }
    );
    proc.subscribe("source", ["video", "audio"]);
    yaml.addNode(proc.reify());

    return document.load(__filename, runtime, YAML.stringify(yaml.reify()));
  }

  describe("Stream processing", () => {
    it("outputs video and audio", async () => {
      norsk = await Norsk.connect({ onShutdown: () => {} });
      const compiled = await testDocument();
      result = await go(norsk, compiled);

      await Promise.all([
        assertNodeOutputsVideoFrames(norsk, result, 'processor'),
        assertNodeOutputsAudioFrames(norsk, result, 'processor')
      ]);
    });

    it("reflects correct state", async () => {
      norsk = await Norsk.connect({ onShutdown: () => {} });
      const compiled = await testDocument();
      result = await go(norsk, compiled);

      await waitForAssert(
        () => latestState()?.status === 'active',
        () => { expect(latestState()?.status).to.equal('active') },
        5000, 100
      );
    });
  });

  describe("Commands", () => {
    it("handles commands", async () => {
      norsk = await Norsk.connect({ onShutdown: () => {} });
      const compiled = await testDocument();
      result = await go(norsk, compiled);

      const definition = result.document.components['processor'].definition as unknown as MyProcessorDefinition;
      const node = result.components['processor'] as MyProcessor;

      definition.handleCommand(node, { type: 'some-command', value: 42 });

      await waitForAssert(
        () => latestState()?.someValue === 42,
        () => { expect(latestState()?.someValue).to.equal(42) },
        5000, 100
      );
    });
  });

  describe("Source changes", () => {
    it("handles source disconnection and reconnection", async () => {
      norsk = await Norsk.connect({ onShutdown: () => {} });
      const runtime = await defaultRuntime();
      const yaml = new YamlBuilder()
        .addNode(new YamlNodeBuilder('processor', MyProcessorInfo(RegistrationConsts), {}).reify())
        .reify();
      const compiled = document.load(__filename, runtime, YAML.stringify(yaml));
      result = await go(norsk, compiled);

      const processor = result.components['processor'] as MyProcessor;
      const source1 = await videoAndAudio(norsk, 'source1');

      result.overrideSources(processor, [
        new StudioNodeSubscriptionSource(source1, testSourceDescription(), {
          type: 'take-all-streams', filter: [{ media: 'video' }, { media: 'audio' }]
        })
      ]);

      await assertNodeOutputsVideoFrames(norsk, result, 'processor');

      // Disconnect
      await source1.close();
      result.overrideSources(processor, []);

      // Reconnect with new source
      const source2 = await videoAndAudio(norsk, 'source2');
      result.overrideSources(processor, [
        new StudioNodeSubscriptionSource(source2, testSourceDescription(), {
          type: 'take-all-streams', filter: [{ media: 'video' }, { media: 'audio' }]
        })
      ]);

      await assertNodeOutputsVideoFrames(norsk, result, 'processor');
    });
  });
});
```

### HTTP API Testing

```typescript
import express from "express";
import { Server } from "http";
import { AddressInfo } from "net";

describe("API", () => {
  let app: express.Express;
  let listener: Server;
  let port: number;
  let norsk: Norsk | undefined;
  let result: RunResult | undefined;

  beforeEach(async () => {
    app = express();
    app.use(express.json());
    await new Promise<void>((resolve) => {
      listener = app.listen(0, '127.0.0.1', () => {
        port = (listener.address() as AddressInfo).port;
        resolve();
      });
    });
  });

  afterEach(async () => {
    await norsk?.close();
    listener.close();
    await new Promise<void>(r => setTimeout(r, 10));
    norsk = undefined;
    result = undefined;
  });

  const latestState = () => result?.runtimeState.getNodeState('component') as (MyState | undefined);

  it("responds to API calls", async () => {
    // Build and run, passing app to go() to enable HTTP routes
    norsk = await Norsk.connect({ onShutdown: () => {} });
    const compiled = await testDocument();
    result = await go(norsk, compiled, app);

    const component = result.components['component'];

    // Wait for component to be ready before calling API
    await waitForCondition(
      () => component.relatedMediaNodes.output.flatMap(o => o.outputStreams).length > 0
    );

    // PUT request
    const response = await fetch(`http://127.0.0.1:${port}/component/status`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: 'closed' })
    });
    expect(response.status).to.equal(204);

    // Verify state changed
    await waitForAssert(
      () => latestState()?.status === 'closed',
      () => { expect(latestState()?.status).to.equal('closed') },
      5000, 100
    );
  });

  it("GET endpoint returns current state", async () => {
    norsk = await Norsk.connect({ onShutdown: () => {} });
    const compiled = await testDocument();
    result = await go(norsk, compiled, app);

    const response = await fetch(`http://127.0.0.1:${port}/component/status`);
    expect(response.status).to.equal(200);
    const body = await response.json();
    expect(body.status).to.equal('idle');
  });

  it("POST command endpoint", async () => {
    norsk = await Norsk.connect({ onShutdown: () => {} });
    const compiled = await testDocument();
    result = await go(norsk, compiled, app);

    const response = await fetch(`http://127.0.0.1:${port}/component/enable`, {
      method: 'POST'
    });
    expect(response.status).to.equal(204);
  });
});
```

---

## Testing with Multiple Components

For integration tests with multiple components wired together:

```typescript
async function multiComponentDocument() {
  const runtime = await defaultRuntime();
  const yaml = new YamlBuilder();

  // Input
  yaml.addNode(new YamlNodeBuilder('source', AvInput.info, { sourceName: true }).reify());

  // Processor subscribed to input
  const proc = new YamlNodeBuilder('processor', MyProcessorInfo(RegistrationConsts), {});
  proc.subscribe("source", ["video", "audio"]);
  yaml.addNode(proc.reify());

  // Output subscribed to processor
  const output = new YamlNodeBuilder('output', MyOutputInfo(RegistrationConsts), { destination: 'rtmp://...' });
  output.subscribe("processor", ["video", "audio"]);
  yaml.addNode(output.reify());

  return document.load(__filename, runtime, YAML.stringify(yaml.reify()));
}
```

---

## Anti-Patterns

### NEVER use fixed setTimeout for waiting

```typescript
// BAD
await new Promise(resolve => setTimeout(resolve, 5000));
expect(component.ready).to.be.true;

// GOOD
await waitForCondition(() => component.ready, 5000, 100, 'component ready');
```

### NEVER call API before component is ready

```typescript
// BAD - race condition
result = await go(norsk, compiled, app);
await fetch(`http://localhost:${port}/component/status`);

// GOOD
result = await go(norsk, compiled, app);
await waitForCondition(
  () => result.components['component'].relatedMediaNodes.output.flatMap(o => o.outputStreams).length > 0
);
await fetch(`http://localhost:${port}/component/status`);
```

### NEVER forget cleanup

Always close Norsk in `afterEach`. Leaked connections cause subsequent tests to fail.

```typescript
afterEach(async () => {
  await norsk?.close();
  norsk = undefined;
  result = undefined;
});
```

---

## Build and Run

```bash
# Build your workspace
npm --workspace workspaces/<your-workspace> run build

# Run a specific test by description match
npm --workspace workspaces/<your-workspace> test -- --grep "My Component"

# Run with debug logging
LOG_LEVEL=debug npm --workspace workspaces/<your-workspace> test -- --grep "My Component"

# Lint
npm --workspace workspaces/<your-workspace> run build:eslint
```

Tests require a running Norsk Engine. In CI, this is typically a Docker container.
Mocha is used with 120s timeout and sequential execution (`--no-parallel --jobs=0`).
