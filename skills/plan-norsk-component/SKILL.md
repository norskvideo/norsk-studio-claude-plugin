---
name: Plan Norsk Studio Component
description: |
  Use when the user wants to create a new Norsk Studio component OR update/modify/improve an existing component
  (processor, input, or output). Guides the user through requirements gathering, codebase research, planning,
  and an 8-step implementation roadmap.

  Trigger this skill when the user says something like:
  - "I need a component that..."
  - "Can you build a processor for..."
  - "I want to create an input/output that..."
  - "Add [feature] to the [component] component"
  - "Update/modify/improve [component]..."
---

# Plan Norsk Studio Component

The planning stage is non-negotiable. A document must be produced and agreed to before any code is written.

## Process Overview

1. **Gather Requirements**
2. **Research Similar Components**
3. **Identify Norsk SDK Nodes**
4. **Create Planning Document**
5. **Confirm Plan with User**
6. **Begin Implementation**

---

## Phase 1: Gather Requirements

Ask clarifying questions:
- What problem does this component solve?
- What are the inputs? (video streams, audio streams, data)
- What are the outputs?
- What processing/transformation happens?
- Configuration options needed?
- Does it need custom UI views? (inline, summary, fullscreen - not all components do)
- What commands/controls for users?
- What HTTP API for external control?

The best example of a component sharing commands/API is `input.file` - refer to it when implementing command/route handlers.

## Phase 2: Research Similar Components

Search for similar existing components in your workspace:

```bash
# Find components by category
ls src/processor.* src/input.* src/output.*

# Look at similar implementations
# Each component has: info.ts, runtime.ts, types.source.yaml
```

Key files to read:
- `info.ts` - Subscription model, config form, validation
- `runtime.ts` - Norsk SDK node usage, event handling
- `types.source.yaml` - Schema definitions (Config, State, Events, Commands, API paths)

## Phase 3: Identify Norsk SDK Nodes

Determine which SDK nodes are needed from `@norskvideo/norsk-sdk`:

**Input Nodes:**
- `norsk.input.whip()` - WebRTC WHIP input
- `norsk.input.rtmpServer()` - RTMP input
- `norsk.input.udpTs()` - UDP TS input
- `norsk.input.srt()` - SRT input

**Audio Processing:**
- `norsk.processor.transform.audioGain()` - Volume control
- `norsk.processor.transform.audioMix()` - Mix multiple audio sources
- `norsk.processor.transform.audioEncode()` - Audio encoding

**Video Processing:**
- `norsk.processor.transform.videoEncode()` - Video encoding
- `norsk.processor.transform.videoDecode()` - Video decoding
- `norsk.processor.transform.videoCompose()` - Compose multiple videos

**Output Nodes:**
- `norsk.output.whep()` - WebRTC WHEP output
- `norsk.output.rtmpServer()` - RTMP output
- `norsk.output.cmafMultiVariant()` - CMAF output

**Utility:**
- `norsk.processor.transform.streamKeyOverride()` - Change stream key
- `norsk.processor.transform.streamMetadataOverride()` - Change metadata

## Phase 4: Create Planning Document

Create a planning document with this structure:

### Section 1: Executive Summary
- Purpose and key features
- Differences from similar components
- Use case diagram showing inputs -> component -> outputs

### Section 2: Architecture Overview
- Server-side node graph (ASCII diagram)
- Client-side UI mockup (if applicable)
- Data flow explanation

### Section 3: Key Design Decisions

**Stream Subscription Model:**
```typescript
subscription: {
  accepts: {
    type: 'dynamic-streams',
    mode: 'any',  // or { type: 'multiple', groupBy: 'sourceName' }
    streams: () => [
      { media: 'video' },
      { media: 'audio' },
    ]
  },
  produces: {
    type: 'dynamic-streams',
    streams(cfg, inputStreams) {
      return inputStreams;  // or custom output stream definitions
    },
  }
}
```

**Configuration Schema (types.source.yaml):**
```yaml
openapi: 3.1.0
info:
  title: Component Name
  version: 1.0.0

paths: {}

components:
  schemas:
    Config:
      type: object
      properties:
        id:
          type: string
        displayName:
          type: string
        # ... component-specific config
      required: [id, displayName]

    State:
      type: object
      properties:
        # ... component state properties

    Events:
      oneOf:
        - $ref: '#/components/schemas/SomeEvent'

    Commands:
      oneOf:
        - $ref: '#/components/schemas/SomeCommand'

    SomeEvent:
      type: object
      properties:
        type:
          enum: ['some-event']
        # ... event properties
      required: [type]

    SomeCommand:
      type: object
      properties:
        type:
          enum: ['some-command']
        # ... command parameters
      required: [type]
```

**Configuration Form and Validation (info.ts):**

Convention:
- Important fields at the top (order is honoured)
- Defaults provided for as much as possible (user can just click 'ok')
- `advanced: true` for anything outside basic usage
- `notes` field is not advanced and belongs at the bottom

```typescript
import type Registration from "@norskvideo/norsk-studio/lib/extension/registration";

export default function({ defineComponent, validation }: Registration) {
  const { Z, SourceName, Port } = validation;

  return defineComponent({
    identifier: 'myplugin.processor.myComponent',
    // ...
    configForm: {
      form: {
        url: {
          help: "URL to connect to",
          hint: {
            type: 'text',
            validation: Z.string().min(5).max(256),
            defaultValue: 'https://example.com'
          }
        },
        port: {
          help: "Port number",
          hint: {
            type: 'numeric',
            validation: Port,  // Built-in: 1024-65535
            defaultValue: 8080
          }
        },
        enabled: {
          help: "Enable this feature",
          advanced: true,
          hint: { type: 'boolean', optional: true }
        },
        notes: {
          help: "Optional notes",
          hint: { type: 'text', optional: true }
        }
      }
    }
  });
}
```

**Available Built-in Validators** (from `validation` object):
- `Z` - Direct Zod access (`Z.string().min(n)`, `Z.number().int()`, etc.)
- `SourceName` - String 2-32 chars
- `Port` - Number 1024-65535
- `NonNegativeInteger` - Integer >= 0
- `JitterBuffer` - Number 0-10000
- `IpAddress` - IPv4 address
- `SrtStreamId` - Empty string or any string
- `IceServer` - String starting with stun: or turn:
- `SrtPassphrase` - Empty or 10-255 chars
- `Hostname` - String 2-2048 chars
- `unique()` - Global unique validation

**Commands and API Integration:**

Components often need both command-based control (from UI) and HTTP API access (for external control).
The recommended pattern:

1. Implement business logic methods on the component node class
2. Methods raise events internally via `this.updates.raiseEvent()`
3. Both `handleCommand()` and HTTP routes call the same methods
4. HTTP handlers use `defineJsonApi` for type-safe, schema-driven routing

```typescript
// Business logic method on the component node
updateConfig(newValue: string): components['schemas']['ConfigUpdatedEvent'] {
  this.cfg.exampleField = newValue;
  const event: components['schemas']['ConfigUpdatedEvent'] = {
    type: 'config-updated',
    exampleField: newValue,
    timestamp: new Date().toISOString()
  };
  this.updates.raiseEvent(event);
  return event;
}

// Command handler calls the method
handleCommand(node: MyNode, command: MyCommand) {
  switch (command.type) {
    case 'update-config':
      node.updateConfig(command.value);
      break;
    default:
      assertUnreachable(command.type);
  }
}

// HTTP API also calls the same method
async instanceRoutes() {
  return defineJsonApi<paths, InstanceRouteArgs<Config, Node, State, Command, Event>>(
    path.join(__dirname, 'types.yaml'),
    {
      '/update-config': {
        post: ({ node }) => ({
          swt: UpdateConfigBody,  // Zod schema from _gen/schema
          handler: async (req) => {
            const event = node.updateConfig(req.body.value);
            return { statusCode: 200, body: event };
          }
        })
      },
      '/status': {
        get: ({ runtime }) => ({
          swt: NoBody,
          handler: async () => {
            const state = runtime.updates.latest();
            return { statusCode: 200, body: state };
          }
        })
      }
    }
  );
}
```

**API Schema in types.source.yaml:**
```yaml
paths:
  /{id}/update-config:
    post:
      operationId: updateConfig
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateConfigBody'
      responses:
        '200':
          description: Config updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ConfigUpdatedEvent'
  /{id}/status:
    get:
      operationId: getStatus
      responses:
        '200':
          description: Current state
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/State'
```

Key imports for API pattern:
- `defineJsonApi` from `@norskvideo/norsk-studio/lib/server/api`
- `NoBody` from `@norskvideo/norsk-studio/lib/server/api-validation`
- `{ paths }` from `./_gen/types`
- Request body schemas from `./_gen/schema`

### Section 4: Implementation Plan (8 Steps)

Detailed below.

### Section 5: API Reference
- Complete schema definitions
- Norsk SDK nodes used with examples

## Phase 5: Confirm Plan with User

Present the planning document. If accepted, commit it before writing code.

## Phase 6: Implementation

Follow the 8-step roadmap. Build and test after EACH step:

```bash
npm run build
npm test -- --grep "<Component Name>"
```

---

## Step 1: Scaffold Files

Create the component directory with three files. Components are auto-discovered
by `autoRegisterComponents` - no manual registration needed. Just create the
directory, add the files, and build.

### Directory structure
```
src/<category>.<name>/
  info.ts
  runtime.ts
  types.source.yaml
```

### Minimal types.source.yaml
```yaml
openapi: 3.1.0
info:
  title: Component Name
  version: 1.0.0

paths: {}

components:
  schemas:
    Config:
      type: object
      properties:
        id:
          type: string
        displayName:
          type: string
      required: [id, displayName]
```

### Stub info.ts
```typescript
import type Registration from "@norskvideo/norsk-studio/lib/extension/registration";

export default function({ defineComponent }: Registration) {
  return defineComponent({
    identifier: '<plugin>.<category>.<name>',
    category: '<category>',
    name: "<Display Name>",
    description: '<Brief description>',
    subscription: {
      accepts: {
        type: 'dynamic-streams',
        mode: 'any',
        streams: () => [
          { media: 'video' },
          { media: 'audio' },
        ]
      },
      produces: {
        type: 'dynamic-streams',
        streams(_cfg, inputStreams) {
          return inputStreams;
        },
      }
    },
    configForm: {
      form: {}
    }
  });
}
```

### Stub runtime.ts
```typescript
import { Norsk } from '@norskvideo/norsk-sdk';
import path from 'path';
import {
  CreatedMediaNode,
  InstanceRouteArgs,
  OnCreated,
  RelatedMediaNodes,
  RuntimeUpdates,
  ServerComponentDefinition,
  ServerComponentSchemas,
  StudioComponentInputStream,
  StudioRuntime,
  StudioShared
} from '@norskvideo/norsk-studio/lib/extension/runtime-types';
import { components } from './_gen/types';
import { schemas } from './_gen/yaml-docs';

type Config = components['schemas']['Config'];

export default class ComponentDefinition
  implements ServerComponentDefinition<Config, ComponentNode, never, never, never> {

  async schemas(): Promise<ServerComponentSchemas> {
    return schemas;
  }

  async create(
    norsk: Norsk,
    cfg: Config,
    cb: OnCreated<ComponentNode>,
    runtime: StudioRuntime<never, never, never>
  ) {
    const node = new ComponentNode(norsk, cfg, runtime);
    await node.initialised;
    cb(node);
  }
}

export class ComponentNode implements CreatedMediaNode {
  id: string;
  norsk: Norsk;
  cfg: Config;
  updates: RuntimeUpdates<never, never, never>;
  shared: StudioShared;
  relatedMediaNodes: RelatedMediaNodes = new RelatedMediaNodes();
  initialised: Promise<void>;

  constructor(norsk: Norsk, cfg: Config, runtime: StudioRuntime<never, never, never>) {
    this.id = cfg.id;
    this.norsk = norsk;
    this.cfg = cfg;
    this.updates = runtime.updates;
    this.shared = runtime.shared;
    this.initialised = this.initialise();
  }

  async initialise() {
    // Create Norsk SDK nodes here
    // this.relatedMediaNodes.addOutput(myNode);
    // this.relatedMediaNodes.addInput(myNode);
  }

  async subscribe(_sources: StudioComponentInputStream[]): Promise<void> {
    // Subscribe nodes to input sources
    // myNode?.subscribe(_sources.map(s => s.select()))
  }
}
```

### Checkpoint
```bash
npm run build
npm run lint
```

---

## Step 2: Add State + Events Schema

### Update types.source.yaml
```yaml
    State:
      type: object
      properties:
        status:
          type: string
      required: [status]

    Events:
      oneOf:
        - $ref: '#/components/schemas/StatusChangedEvent'

    StatusChangedEvent:
      type: object
      properties:
        type:
          enum: ['status-changed']
        status:
          type: string
      required: [type, status]
```

### Update info.ts - add runtime section
```typescript
import { assertUnreachable } from "@norskvideo/norsk-studio/lib/shared/util";

// In defineComponent:
runtime: {
  initialState: () => ({
    status: 'idle'
  }),
  handleEvent: (ev, state) => {
    const evType = ev.type;
    switch (evType) {
      case 'status-changed':
        return { ...state, status: ev.status };
      default:
        assertUnreachable(evType);
    }
  }
}
```

### Update runtime.ts types
```typescript
import { components } from './_gen/types';

export type ComponentState = components['schemas']['State'];
export type ComponentEvent = components['schemas']['Events'];

// Update class signatures to use State/Event types
```

### Write initial test
```typescript
import { expect } from "chai";
import ComponentInfo from "../<category>.<name>/info";
import { RegistrationConsts } from "@norskvideo/norsk-studio/lib/extension/client-types";

describe("<Component Name>", () => {
  it("compiles with correct schema", () => {
    const info = ComponentInfo(RegistrationConsts);
    expect(info).to.exist;
    expect(info.runtime?.initialState).to.be.a('function');
    expect(info.runtime?.handleEvent).to.be.a('function');
  });
});
```

---

## Step 3: Implement Runtime Core

Key tasks:
1. Implement subscription handler
2. Create core Norsk SDK nodes
3. Connect nodes in proper order
4. Raise initial events
5. Setup node outputs via `relatedMediaNodes.addOutput()`/`addInput()`

### Critical Pattern: Guard Against Node Recreation
```typescript
async subscribe(sources: StudioComponentInputStream[]): Promise<void> {
  // ALWAYS guard to prevent recreation on re-subscription
  if (!this.processorNode) {
    this.processorNode = await this.norsk.processor.transform.someMethod({
      id: `${this.cfg.id}-processor`,
    });
    this.relatedMediaNodes.addOutput(this.processorNode);
    this.relatedMediaNodes.addInput(this.processorNode);
  }

  // Now safe to update subscriptions
  this.processorNode.subscribe(sources.map(s => s.select()));
}
```

### Critical Pattern: Always Raise Events
```typescript
setVolume(level: number, muted: boolean): void {
  // Update node if it exists
  if (this.gainNode) {
    this.gainNode.updateConfig({
      channelGains: muted ? [null, null] : this.getChannelGains(level)
    });
  }

  // ALWAYS raise event (even if node doesn't exist yet)
  this.updates.raiseEvent({
    type: 'volume-changed',
    volume: { level, muted }
  });
}
```

---

## Step 4: Add Extended Functionality

- Add complex processing nodes
- Implement multi-stream handling
- Add dynamic stream detection
- Raise detailed status events

---

## Step 5: Add Commands and APIs

1. Define Commands schema in types.source.yaml
2. Define API paths in types.source.yaml (under `paths:` section)
3. Define request/response body schemas
4. Implement business logic methods on the node class
5. Implement `handleCommand()` on the definition class
6. Implement `instanceRoutes()` using `defineJsonApi`
7. Use `assertUnreachable` in switch statements for exhaustive checking
8. Test both command handling and API endpoints

Key imports:
```typescript
import { defineJsonApi } from '@norskvideo/norsk-studio/lib/server/api';
import { NoBody } from '@norskvideo/norsk-studio/lib/server/api-validation';
import { paths } from './_gen/types';
import { SomeRequestBody } from './_gen/schema';
import { assertUnreachable } from '@norskvideo/norsk-studio/lib/shared/util';
```

---

## Step 6: Add Form Validation

Add Zod validators to `configForm` field hints in `info.ts`.

Use `extraValidation` ONLY for context-aware validation (stream checks, cross-field dependencies):

```typescript
extraValidation: function(ctx) {
  ctx.requireExactVideo(1);
  ctx.requireExactAudio(1);
}
```

Do NOT use `extraValidation` for basic field validation - use Zod in configForm instead.

---

## Step 7: UI Views (optional)

Not all components need custom UI. Whether to add inline, summary, or fullscreen views
is a decision made during the planning phase based on the component's requirements.
Skip this step entirely if the component doesn't need custom views.

**View types available:**
- `inline` - Small view shown in the node editor
- `summary` - Compact status display
- `fullscreen` - Full-page interactive view (e.g. for monitoring, control panels)

**Creating a view:**

```typescript
interface ViewProps {
  state: ComponentState;
  config: ComponentConfig;
  sendCommand: (cmd: ComponentCommand) => void;
}

function FullScreen({ state, config, sendCommand }: ViewProps) {
  return (
    <div className="min-h-screen h-full w-full bg-gray-900 text-white">
      {/* Implementation */}
    </div>
  );
}

export default FullScreen;
```

**Register in info.ts:**
```typescript
import FullscreenView from './fullscreen-view';
import InlineView from './inline-view';

// In defineComponent:
runtime: {
  // ...existing handlers...
  fullscreen: FullscreenView,
  inline: InlineView
}
```

---

## Step 8: UX Polish

- Add tooltips and help text to configForm
- Add visual feedback for states
- Handle edge cases
- Add LLM hints:

```typescript
llmHints: [
  "Brief hint about component usage",
  "Requirement or constraint",
  "Expected behavior detail"
]
```

---

## Writing Tests

See the **test-norsk-component** skill for the complete reference on shared test libraries,
assertion helpers, and patterns for testing inputs, processors, and outputs. It covers:

- Test source helpers (`video()`, `audio()`, `videoAndAudio()`)
- Assertion helpers (`assertNodeOutputsVideoFrames`, `waitForAssert`, `TraceSink`)
- Async waiting (`waitForCondition`, `waitForEvent`)
- Wiring sources via `YamlBuilder` subscriptions or `overrideSources`/`ComponentSubscriptions`
- HTTP API endpoint testing with Express
- Complete test templates for each component type

---

## Key Principles

1. **Incremental verification** - Build and test after each step
2. **Guard against recreation** - Use `if (!this.node)` checks in subscribe()
3. **Always raise events** - Even if nodes don't exist yet
4. **Use assertUnreachable** - In switch statements for exhaustive type checking
5. **Auto-discovery** - No manual registration needed; just create the directory
6. **Test thoroughly** - Test commands, API endpoints, state changes, events
7. **Schema-first** - Define types.source.yaml first, then implement

## Success Criteria

- [ ] Build passes
- [ ] ESLint passes
- [ ] Tests passing (commands, API, state, events)
- [ ] Planning document complete
- [ ] LLM hints added
- [ ] Validation implemented
- [ ] UI views functional (if planned)
