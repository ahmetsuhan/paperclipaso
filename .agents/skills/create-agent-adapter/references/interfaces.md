# Adapter interfaces and helper reference

Types and helpers for the create-agent-adapter skill. The source of truth is
`packages/adapter-utils/src/types.ts`; re-check it when exact signatures matter.

## 2. Shared Types (`@paperclipai/adapter-utils`)

All adapter interfaces live in `packages/adapter-utils/src/types.ts`. Import from `@paperclipai/adapter-utils` (types) or `@paperclipai/adapter-utils/server-utils` (runtime helpers).

### Core Interfaces

```ts
// The execute function signature — every adapter must implement this
interface AdapterExecutionContext {
  runId: string;
  agent: AdapterAgent;          // { id, companyId, name, adapterType, adapterConfig }
  runtime: AdapterRuntime;      // { sessionId, sessionParams, sessionDisplayId, taskKey }
  config: Record<string, unknown>;  // The agent's adapterConfig blob
  context: Record<string, unknown>; // Runtime context (taskId, wakeReason, approvalId, etc.)
  onLog: (stream: "stdout" | "stderr", chunk: string) => Promise<void>;
  onMeta?: (meta: AdapterInvocationMeta) => Promise<void>;
  authToken?: string;
}

interface AdapterExecutionResult {
  exitCode: number | null;
  signal: string | null;
  timedOut: boolean;
  errorMessage?: string | null;
  usage?: UsageSummary;           // { inputTokens, outputTokens, cachedInputTokens? }
  sessionId?: string | null;      // Legacy — prefer sessionParams
  sessionParams?: Record<string, unknown> | null;  // Opaque session state persisted between runs
  sessionDisplayId?: string | null;
  provider?: string | null;       // "anthropic", "openai", etc.
  model?: string | null;
  costUsd?: number | null;
  resultJson?: Record<string, unknown> | null;
  summary?: string | null;        // Human-readable summary of what the agent did
  clearSession?: boolean;         // true = tell Paperclip to forget the stored session
}

interface AdapterSessionCodec {
  deserialize(raw: unknown): Record<string, unknown> | null;
  serialize(params: Record<string, unknown> | null): Record<string, unknown> | null;
  getDisplayId?(params: Record<string, unknown> | null): string | null;
}
```

### Module Interfaces

```ts
// Server — registered in server/src/adapters/registry.ts
interface ServerAdapterModule {
  type: string;
  execute(ctx: AdapterExecutionContext): Promise<AdapterExecutionResult>;
  testEnvironment(ctx: AdapterEnvironmentTestContext): Promise<AdapterEnvironmentTestResult>;
  sessionCodec?: AdapterSessionCodec;
  supportsLocalAgentJwt?: boolean;
  models?: { id: string; label: string }[];
  agentConfigurationDoc?: string;
}

// UI — registered in ui/src/adapters/registry.ts
interface UIAdapterModule {
  type: string;
  label: string;
  parseStdoutLine: (line: string, ts: string) => TranscriptEntry[];
  ConfigFields: ComponentType<AdapterConfigFieldsProps>;
  buildAdapterConfig: (values: CreateConfigValues) => Record<string, unknown>;
}

// CLI — registered in cli/src/adapters/registry.ts
interface CLIAdapterModule {
  type: string;
  formatStdoutEvent: (line: string, debug: boolean) => void;
}
```

---

## 2.1 Adapter Environment Test Contract types

```ts
type AdapterEnvironmentCheckLevel = "info" | "warn" | "error";
type AdapterEnvironmentTestStatus = "pass" | "warn" | "fail";

interface AdapterEnvironmentCheck {
  code: string;
  level: AdapterEnvironmentCheckLevel;
  message: string;
  detail?: string | null;
  hint?: string | null;
}

interface AdapterEnvironmentTestResult {
  adapterType: string;
  status: AdapterEnvironmentTestStatus;
  checks: AdapterEnvironmentCheck[];
  testedAt: string; // ISO timestamp
}

interface AdapterEnvironmentTestContext {
  companyId: string;
  adapterType: string;
  config: Record<string, unknown>; // runtime-resolved adapterConfig
}
```

---

## Environment variables the server always injects

| Variable | Source |
|----------|--------|
| `PAPERCLIP_AGENT_ID` | `agent.id` |
| `PAPERCLIP_COMPANY_ID` | `agent.companyId` |
| `PAPERCLIP_API_URL` | Server's own URL |
| `PAPERCLIP_RUN_ID` | Current run id |
| `PAPERCLIP_TASK_ID` | `context.taskId` or `context.issueId` |
| `PAPERCLIP_WAKE_REASON` | `context.wakeReason` |
| `PAPERCLIP_WAKE_COMMENT_ID` | `context.wakeCommentId` or `context.commentId` |
| `PAPERCLIP_APPROVAL_ID` | `context.approvalId` |
| `PAPERCLIP_APPROVAL_STATUS` | `context.approvalStatus` |
| `PAPERCLIP_LINKED_ISSUE_IDS` | `context.issueIds` (comma-separated) |
| `PAPERCLIP_API_KEY` | `authToken` (if no explicit key in config) |

---

## 6. Server-Utils Helpers

Import from `@paperclipai/adapter-utils/server-utils`:

| Helper | Purpose |
|--------|---------|
| `asString(val, fallback)` | Safe string extraction |
| `asNumber(val, fallback)` | Safe number extraction |
| `asBoolean(val, fallback)` | Safe boolean extraction |
| `asStringArray(val)` | Safe string array extraction |
| `parseObject(val)` | Safe `Record<string, unknown>` extraction |
| `parseJson(str)` | Safe JSON.parse returning `Record` or null |
| `renderTemplate(tmpl, data)` | `{{path.to.value}}` template rendering |
| `buildPaperclipEnv(agent)` | Standard `PAPERCLIP_*` env vars |
| `redactEnvForLogs(env)` | Redact sensitive keys for onMeta |
| `ensureAbsoluteDirectory(cwd)` | Validate cwd exists and is absolute |
| `ensureCommandResolvable(cmd, cwd, env)` | Validate command is in PATH |
| `ensurePathInEnv(env)` | Ensure PATH exists in env |
| `runChildProcess(runId, cmd, args, opts)` | Spawn with timeout, logging, capture |

---

## 9. TranscriptEntry Kinds Reference

The UI run viewer displays these entry kinds:

| Kind | Fields | Usage |
|------|--------|-------|
| `init` | `model`, `sessionId` | Agent initialization |
| `assistant` | `text` | Agent text response |
| `thinking` | `text` | Agent reasoning/thinking |
| `user` | `text` | User message |
| `tool_call` | `name`, `input` | Tool invocation |
| `tool_result` | `toolUseId`, `content`, `isError` | Tool result |
| `result` | `text`, `inputTokens`, `outputTokens`, `cachedTokens`, `costUsd`, `subtype`, `isError`, `errors` | Final result with usage |
| `stderr` | `text` | Stderr output |
| `system` | `text` | System messages |
| `stdout` | `text` | Raw stdout fallback |
