# Adapter code templates

Code skeletons for the create-agent-adapter skill. Copy the closest existing adapter
(`packages/adapters/claude-local`, `packages/adapters/codex-local`) when it fits
better than these skeletons.

## package.json (four-export convention)

```json
{
  "name": "@paperclipai/adapter-<name>",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./server": "./src/server/index.ts",
    "./ui": "./src/ui/index.ts",
    "./cli": "./src/cli/index.ts"
  },
  "dependencies": {
    "@paperclipai/adapter-utils": "workspace:*",
    "picocolors": "^1.1.1"
  },
  "devDependencies": {
    "typescript": "^5.7.3"
  }
}
```

## Root `index.ts` metadata

```ts
export const type = "my_agent";        // snake_case, globally unique
export const label = "My Agent (local)";

export const models = [
  { id: "model-a", label: "Model A" },
  { id: "model-b", label: "Model B" },
];

export const agentConfigurationDoc = `# my_agent agent configuration
...document all config fields here...
`;
```

## `agentConfigurationDoc` as routing logic

```ts
export const agentConfigurationDoc = `# my_agent agent configuration

Adapter: my_agent

Use when:
- The agent needs to run MyAgent CLI locally on the host machine
- You need session persistence across runs (MyAgent supports thread resumption)
- The task requires MyAgent-specific tools (e.g. web search, code execution)

Don't use when:
- You need a simple one-shot script execution (use the "process" adapter instead)
- The agent doesn't need conversational context between runs (process adapter is simpler)
- MyAgent CLI is not installed on the host

Core fields:
- cwd (string, required): absolute working directory for the agent process
...
`;
```

## `server/index.ts`

```ts
export { execute } from "./execute.js";
export { testEnvironment } from "./test.js";
export { parseMyAgentOutput, isMyAgentUnknownSessionError } from "./parse.js";

// Session codec — required for session persistence
export const sessionCodec: AdapterSessionCodec = {
  deserialize(raw) { /* raw DB JSON -> typed params or null */ },
  serialize(params) { /* typed params -> JSON for DB storage */ },
  getDisplayId(params) { /* -> human-readable session id string */ },
};
```

## `ui/parse-stdout.ts`

```ts
export function parseMyAgentStdoutLine(line: string, ts: string): TranscriptEntry[] {
  // Parse JSON line, map to appropriate TranscriptEntry kind(s)
  // Return [{ kind: "stdout", ts, text: line }] as fallback
}
```

## `ui/build-config.ts`

```ts
export function buildMyAgentConfig(v: CreateConfigValues): Record<string, unknown> {
  const ac: Record<string, unknown> = {};
  if (v.cwd) ac.cwd = v.cwd;
  if (v.promptTemplate) ac.promptTemplate = v.promptTemplate;
  if (v.model) ac.model = v.model;
  ac.timeoutSec = 0;
  ac.graceSec = 15;
  // ... adapter-specific fields
  return ac;
}
```

## `cli/format-event.ts`

```ts
import pc from "picocolors";

export function printMyAgentStreamEvent(raw: string, debug: boolean): void {
  // Parse JSON line from agent stdout
  // Print colored output: blue for system, green for assistant, yellow for tools
  // In debug mode, print unrecognized lines in gray
}
```

## Registration snippets

### 4.1 Server Registry (`server/src/adapters/registry.ts`)

```ts
import { execute as myExecute, sessionCodec as mySessionCodec } from "@paperclipai/adapter-my-agent/server";
import { agentConfigurationDoc as myDoc, models as myModels } from "@paperclipai/adapter-my-agent";

const myAgentAdapter: ServerAdapterModule = {
  type: "my_agent",
  execute: myExecute,
  sessionCodec: mySessionCodec,
  models: myModels,
  supportsLocalAgentJwt: true,  // true if agent can use Paperclip API
  agentConfigurationDoc: myDoc,
};

// Add to the adaptersByType map
const adaptersByType = new Map<string, ServerAdapterModule>(
  [..., myAgentAdapter].map((a) => [a.type, a]),
);
```

### 4.2 UI Registry (`ui/src/adapters/registry.ts`)

```ts
import { myAgentUIAdapter } from "./my-agent";

const adaptersByType = new Map<string, UIAdapterModule>(
  [..., myAgentUIAdapter].map((a) => [a.type, a]),
);
```

With `ui/src/adapters/my-agent/index.ts`:

```ts
import type { UIAdapterModule } from "../types";
import { parseMyAgentStdoutLine } from "@paperclipai/adapter-my-agent/ui";
import { MyAgentConfigFields } from "./config-fields";
import { buildMyAgentConfig } from "@paperclipai/adapter-my-agent/ui";

export const myAgentUIAdapter: UIAdapterModule = {
  type: "my_agent",
  label: "My Agent",
  parseStdoutLine: parseMyAgentStdoutLine,
  ConfigFields: MyAgentConfigFields,
  buildAdapterConfig: buildMyAgentConfig,
};
```

### 4.3 CLI Registry (`cli/src/adapters/registry.ts`)

```ts
import { printMyAgentStreamEvent } from "@paperclipai/adapter-my-agent/cli";

const myAgentCLIAdapter: CLIAdapterModule = {
  type: "my_agent",
  formatStdoutEvent: printMyAgentStreamEvent,
};

// Add to the adaptersByType map
```

## Session resume pattern

**Pattern** (from both claude-local and codex-local):

```ts
const canResumeSession =
  runtimeSessionId.length > 0 &&
  (runtimeSessionCwd.length === 0 || path.resolve(runtimeSessionCwd) === path.resolve(cwd));
const sessionId = canResumeSession ? runtimeSessionId : null;

// ... run attempt ...

// If resume failed with unknown session, retry fresh
if (sessionId && !proc.timedOut && exitCode !== 0 && isUnknownSessionError(output)) {
  const retry = await runAttempt(null);
  return toResult(retry, { clearSessionOnMissingSession: true });
}
```
