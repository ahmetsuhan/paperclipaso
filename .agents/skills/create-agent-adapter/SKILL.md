---
name: create-agent-adapter
description: >
  Create or modify Paperclip agent adapters across server, UI, and CLI surfaces.
  Use when adding support for a new CLI agent, API agent, custom process, or
  adapter package. Native Paperclip Runner harnesses use create-runner-adapter.
---

# Creating a Paperclip Agent Adapter

An adapter bridges Paperclip's orchestration layer to a specific AI agent runtime (Claude Code, Codex CLI, a custom process, an HTTP endpoint, etc.). Each adapter is a self-contained package that provides implementations for **three consumers**: the server, the UI, and the CLI.

Read the reference file the current step needs instead of loading all of them:

- [references/interfaces.md](references/interfaces.md): adapter-utils types,
  environment-test types, injected env vars, server-utils helpers, and
  TranscriptEntry kinds.
- [references/templates.md](references/templates.md): package.json, metadata,
  server/UI/CLI module skeletons, registry snippets, and the session-resume pattern.
- [references/skills-injection.md](references/skills-injection.md): how
  claude-local and codex-local inject Paperclip skills, with code.

---

## 1. Architecture Overview

```
packages/adapters/<name>/
  src/
    index.ts            # Shared metadata (type, label, models, agentConfigurationDoc)
    server/
      index.ts          # Server exports: execute, sessionCodec, parse helpers
      execute.ts        # Core execution logic (AdapterExecutionContext -> AdapterExecutionResult)
      parse.ts          # Stdout/result parsing for the agent's output format
    ui/
      index.ts          # UI exports: parseStdoutLine, buildConfig
      parse-stdout.ts   # Line-by-line stdout -> TranscriptEntry[] for the run viewer
      build-config.ts   # CreateConfigValues -> adapterConfig JSON for agent creation form
    cli/
      index.ts          # CLI exports: formatStdoutEvent
      format-event.ts   # Colored terminal output for `paperclipai run --watch`
  package.json
  tsconfig.json
```

Three separate registries consume adapter modules:

| Registry | Location | Interface |
|----------|----------|-----------|
| Server | `server/src/adapters/registry.ts` | `ServerAdapterModule` |
| UI | `ui/src/adapters/registry.ts` | `UIAdapterModule` |
| CLI | `cli/src/adapters/registry.ts` | `CLIAdapterModule` |

---

## 2. Shared Types (`@paperclipai/adapter-utils`)

All adapter interfaces live in `packages/adapter-utils/src/types.ts`. Import from `@paperclipai/adapter-utils` (types) or `@paperclipai/adapter-utils/server-utils` (runtime helpers). The core execution, session-codec, and server/UI/CLI module interfaces are in [references/interfaces.md](references/interfaces.md).

---

## 2.1 Adapter Environment Test Contract

Every server adapter must implement `testEnvironment(...)`. This powers the board UI "Test environment" button in agent configuration. The result and check types are in [references/interfaces.md](references/interfaces.md).

Guidelines:

- Return structured diagnostics, never throw for expected findings.
- Use `error` for invalid/unusable runtime setup (bad cwd, missing command, invalid URL).
- Use `warn` for non-blocking but important situations.
- Use `info` for successful checks and context.

Severity policy is product-critical: warnings are not save blockers.  
Example: for `claude_local`, an explicitly configured `ANTHROPIC_API_KEY` or selected managed API connection is `info`: the user chose that authentication. An ambient server key overriding subscription login remains `warn`, not `error`.

---

## 3. Step-by-Step: Creating a New Adapter

### 3.1 Create the Package

Create the layout shown in section 1.

**package.json** — must use the four-export convention (`.`, `./server`, `./ui`, `./cli`); see [references/templates.md](references/templates.md).

### 3.2 Root `index.ts` — Adapter Metadata

This file is imported by **all three** consumers (server, UI, CLI). Keep it dependency-free (no Node APIs, no React).

**Required exports:**
- `type` — the adapter type key, stored in `agents.adapter_type`
- `label` — human-readable name for the UI
- `models` — available model options for the agent creation form
- `agentConfigurationDoc` — markdown describing all `adapterConfig` fields (used by LLM agents configuring other agents)

**Writing `agentConfigurationDoc` as routing logic:**

The `agentConfigurationDoc` is read by LLM agents (including Paperclip agents that create other agents). Write it as **routing logic**, not marketing copy. Include concrete "use when" and "don't use when" guidance so an LLM can decide whether this adapter is appropriate for a given task.

See the worked example in [references/templates.md](references/templates.md).

Adding explicit negative cases improves adapter selection accuracy. One concrete anti-pattern is worth more than three paragraphs of description.

### 3.3 Server Module

#### `server/execute.ts` — The Core

This is the most important file. It receives an `AdapterExecutionContext` and must return an `AdapterExecutionResult`.

**Required behavior:**

1. **Read config** — extract typed values from `ctx.config` using helpers (`asString`, `asNumber`, `asBoolean`, `asStringArray`, `parseObject` from `@paperclipai/adapter-utils/server-utils`)
2. **Build environment** — call `buildPaperclipEnv(agent)` then layer in `PAPERCLIP_RUN_ID`, context vars (`PAPERCLIP_TASK_ID`, `PAPERCLIP_WAKE_REASON`, `PAPERCLIP_WAKE_COMMENT_ID`, `PAPERCLIP_APPROVAL_ID`, `PAPERCLIP_APPROVAL_STATUS`, `PAPERCLIP_LINKED_ISSUE_IDS`), user env overrides, and auth token
3. **Resolve session** — check `runtime.sessionParams` / `runtime.sessionId` for an existing session; validate it's compatible (e.g. same cwd); decide whether to resume or start fresh
4. **Render prompt** — use `renderTemplate(template, data)` with the template variables: `agentId`, `companyId`, `runId`, `company`, `agent`, `run`, `context`
5. **Call onMeta** — emit adapter invocation metadata before spawning the process
6. **Spawn the process** — use `runChildProcess()` for CLI-based agents or `fetch()` for HTTP-based agents
7. **Parse output** — convert the agent's stdout into structured data (session id, usage, summary, errors)
8. **Handle session errors** — if resume fails with "unknown session", retry with a fresh session and set `clearSession: true`
9. **Return AdapterExecutionResult** — populate all fields the agent runtime supports

The `PAPERCLIP_*` variables the server always injects are listed in [references/interfaces.md](references/interfaces.md).

#### `server/parse.ts` — Output Parser

Parse the agent's stdout format into structured data. Must handle:

- **Session identification** — extract session/thread ID from init events
- **Usage tracking** — extract token counts (input, output, cached)
- **Cost tracking** — extract cost if available
- **Summary extraction** — pull the agent's final text response
- **Error detection** — identify error states, extract error messages
- **Unknown session detection** — export an `is<Agent>UnknownSessionError()` function for retry logic

**Treat agent output as untrusted.** The stdout you're parsing comes from an LLM-driven process that may have executed arbitrary tool calls, fetched external content, or been influenced by prompt injection in the files it read. Parse defensively:
- Never `eval()` or dynamically execute anything from output
- Use safe extraction helpers (`asString`, `asNumber`, `parseJson`) — they return fallbacks on unexpected types
- Validate session IDs and other structured data before passing them through
- If output contains URLs, file paths, or commands, do not act on them in the adapter — just record them

#### `server/index.ts` — Server Exports

Export `execute`, `testEnvironment`, the parse helpers, and a `sessionCodec` (required for session persistence); see [references/templates.md](references/templates.md).

#### `server/test.ts` — Environment Diagnostics

Implement adapter-specific preflight checks used by the UI test button.

Minimum expectations:

1. Validate required config primitives (paths, commands, URLs, auth assumptions)
2. Return check objects with deterministic `code` values
3. Map severity consistently (`info` / `warn` / `error`)
4. Compute final status:
   - `fail` if any `error`
   - `warn` if no errors and at least one warning
   - `pass` otherwise

This operation should be lightweight and side-effect free.

### 3.4 UI Module

#### `ui/parse-stdout.ts` — Transcript Parser

Converts individual stdout lines into `TranscriptEntry[]` for the run detail viewer. Must handle the agent's streaming output format and produce entries of these kinds:

- `init` — model/session initialization
- `assistant` — agent text responses
- `thinking` — agent thinking/reasoning (if supported)
- `tool_call` — tool invocations with name and input
- `tool_result` — tool results with content and error flag
- `user` — user messages in the conversation
- `result` — final result with usage stats
- `stdout` — fallback for unparseable lines

Return `[{ kind: "stdout", ts, text: line }]` for those lines.

#### `ui/build-config.ts` — Config Builder

Converts the UI form's `CreateConfigValues` into the `adapterConfig` JSON blob stored on the agent.

#### UI Config Fields Component

Create `ui/src/adapters/<name>/config-fields.tsx` with a React component implementing `AdapterConfigFieldsProps`. This renders adapter-specific form fields in the agent creation/edit form.

Use the shared primitives from `ui/src/components/agent-config-primitives`:
- `Field` — labeled form field wrapper
- `ToggleField` — boolean toggle with label and hint
- `DraftInput` — text input with draft/commit behavior
- `DraftNumberInput` — number input with draft/commit behavior
- `help` — standard hint text for common fields

The component must support both `create` mode (using `values`/`set`) and `edit` mode (using `config`/`eff`/`mark`).

### 3.5 CLI Module

#### `cli/format-event.ts` — Terminal Formatter

Pretty-prints stdout lines for `paperclipai run --watch`. Use `picocolors` for coloring.

---

## 4. Registration Checklist

After creating the adapter package, register it in all three consumers. Snippets for each are in [references/templates.md](references/templates.md):

- `server/src/adapters/registry.ts` — a `ServerAdapterModule` with `execute`, `sessionCodec`, `models`, `supportsLocalAgentJwt`, and `agentConfigurationDoc`, added to `adaptersByType`.
- `ui/src/adapters/registry.ts` — a `UIAdapterModule` assembled in `ui/src/adapters/<name>/index.ts`.
- `cli/src/adapters/registry.ts` — a `CLIAdapterModule` with `formatStdoutEvent`.

---

## 5. Session Management — Designing for Long Runs

Sessions allow agents to maintain conversation context across runs. The system is **codec-based** — each adapter defines how to serialize/deserialize its session state.

**Design for long runs from the start.** Treat session reuse as the default primitive, not an optimization to add later. An agent working on an issue may be woken dozens of times — for the initial assignment, approval callbacks, re-assignments, manual nudges. Each wake should resume the existing conversation so the agent retains full context about what it has already done, what files it has read, and what decisions it has made. Starting fresh each time wastes tokens on re-reading the same files and risks contradictory decisions.

**Key concepts:**
- `sessionParams` is an opaque `Record<string, unknown>` stored in the DB per task
- The adapter's `sessionCodec.serialize()` converts execution result data to storable params
- `sessionCodec.deserialize()` converts stored params back for the next run
- `sessionCodec.getDisplayId()` extracts a human-readable session ID for the UI
- **cwd-aware resume**: if the session was created in a different cwd than the current config, skip resuming (prevents cross-project session contamination)
- **Unknown session retry**: if resume fails with a "session not found" error, retry with a fresh session and return `clearSession: true` so Paperclip wipes the stale session

If the agent runtime supports any form of context compaction or conversation compression (e.g. Claude Code's automatic context management, or Codex's `previous_response_id` chaining), lean on it. Adapters that support session resume get compaction for free — the agent runtime handles context window management internally across resumes.

**Pattern** (from both claude-local and codex-local): resume only when the stored cwd matches, and retry fresh with `clearSessionOnMissingSession` when resume reports an unknown session. Code is in [references/templates.md](references/templates.md).

---

## 6. Server-Utils Helpers

Import safe extraction, template, env, validation, and process helpers from `@paperclipai/adapter-utils/server-utils`; the table is in [references/interfaces.md](references/interfaces.md).

---

## 7. Conventions and Patterns

### Naming
- Adapter type: `snake_case` (e.g. `claude_local`, `codex_local`)
- Package name: `@paperclipai/adapter-<kebab-name>`
- Package directory: `packages/adapters/<kebab-name>/`

### Config Parsing
- Never trust `config` values directly — always use `asString`, `asNumber`, etc.
- Provide sensible defaults for every optional field
- Document all fields in `agentConfigurationDoc`

### Prompt Templates
- Support `promptTemplate` for every run
- Use `renderTemplate()` with the standard variable set
- Default prompt should use `DEFAULT_PAPERCLIP_AGENT_PROMPT_TEMPLATE` from `@paperclipai/adapter-utils/server-utils` so local adapters share Paperclip's execution contract: act in the same heartbeat, avoid planning-only exits unless requested, leave durable progress and a next action, use child issues instead of polling, mark blockers with owner/action, and respect governance boundaries.

### Error Handling
- Differentiate timeout vs process error vs parse failure
- Always populate `errorMessage` on failure
- Include raw stdout/stderr in `resultJson` when parsing fails
- Handle the agent CLI not being installed (command not found)

### Logging
- Call `onLog("stdout", ...)` and `onLog("stderr", ...)` for all process output — this feeds the real-time run viewer
- Call `onMeta(...)` before spawning to record invocation details
- Use `redactEnvForLogs()` when including env in meta

### Paperclip Skills Injection

Paperclip ships shared skills (in the repo's top-level `skills/` directory) that agents need at runtime — things like the `paperclip` API skill and the `paperclip-create-agent` workflow skill. Each adapter is responsible for making these skills discoverable by its agent runtime **without polluting the agent's working directory**.

**The constraint:** never copy or symlink skills into the agent's `cwd`. The cwd is the user's project checkout — writing `.claude/skills/` or any other files into it would contaminate the repo with Paperclip internals, break git status, and potentially leak into commits.

**The pattern:** create a clean, isolated location for skills and tell the agent runtime to look there.

**Examples:** claude-local uses a tmpdir + `--add-dir`; codex-local symlinks into its own global skills home and skips existing entries. Code is in [references/skills-injection.md](references/skills-injection.md).

**For a new adapter:** figure out how your agent runtime discovers skills/plugins, then choose the cleanest injection path:

1. **Best: tmpdir + flag** (like claude-local) — if the runtime supports an "additional directory" flag, create a tmpdir, symlink skills in, pass the flag, clean up after. Zero side effects.
2. **Acceptable: global config dir** (like codex-local) — if the runtime has a global skills/plugins directory separate from the project, symlink there. Skip existing entries to avoid overwriting user customizations.
3. **Acceptable: env var** — if the runtime reads a skills/plugin path from an environment variable, point it at the repo's `skills/` directory directly.
4. **Last resort: prompt injection** — if the runtime has no plugin system, include skill content in the prompt template itself. This uses tokens but avoids filesystem side effects entirely.

**Skills as loaded procedures, not prompt bloat.** The Paperclip skills (like `paperclip` and `paperclip-create-agent`) are designed as on-demand procedures: the agent sees skill metadata (name + description) in its context, but only loads the full SKILL.md content when it decides to invoke a skill. This keeps the base prompt small. When writing `agentConfigurationDoc` or prompt templates for your adapter, do not inline skill content — let the agent runtime's skill discovery do the work. The descriptions in each SKILL.md frontmatter act as routing logic: they tell the agent when to load the full skill, not what the skill contains.

**Explicit vs. fuzzy skill invocation.** For production workflows where reliability matters (e.g. an agent that must always call the Paperclip API to report status), use explicit instructions in the prompt template: "Use the paperclip skill to report your progress." Fuzzy routing (letting the model decide based on description matching) is fine for exploratory tasks but unreliable for mandatory procedures.

---

## 8. Security Considerations

Adapters sit at the boundary between Paperclip's orchestration layer and arbitrary agent execution. This is a high-risk surface.

### Treat Agent Output as Untrusted

The agent process runs LLM-driven code that reads external files, fetches URLs, and executes tools. Its output may be influenced by prompt injection from the content it processes. The adapter's parse layer is a trust boundary — validate everything, execute nothing (see `server/parse.ts` above).

### Secret Injection via Environment, Not Prompts

Never put secrets (API keys, tokens) into prompt templates or config fields that flow through the LLM. Instead, inject them as environment variables that the agent's tools can read directly:

- `PAPERCLIP_API_KEY` is injected by the server into the process environment, not the prompt
- User-provided secrets in `config.env` are passed as env vars, redacted in `onMeta` logs
- The `redactEnvForLogs()` helper automatically masks any key matching `/(key|token|secret|password|authorization|cookie)/i`

This follows the "sidecar injection" pattern: the model never sees the real secret value, but the tools it invokes can read it from the environment.

### Network Access

If your agent runtime supports network access controls (sandboxing, allowlists), configure them in the adapter:

- Prefer minimal allowlists over open internet access. An agent that only needs to call the Paperclip API and GitHub should not have access to arbitrary hosts.
- Skills + network = amplified risk. A skill that teaches the agent to make HTTP requests combined with unrestricted network access creates an exfiltration path. Constrain one or the other.
- If the runtime supports layered policies (org-level defaults + per-request overrides), wire the org-level policy into the adapter config and let per-agent config narrow further.

### Process Isolation

- CLI-based adapters inherit the server's user permissions. The `cwd` and `env` config determine what the agent process can access on the filesystem.
- `dangerouslySkipPermissions` / `dangerouslyBypassApprovalsAndSandbox` flags exist for development convenience but must be documented as dangerous in `agentConfigurationDoc`. Production deployments should not use them.
- Timeout and grace period (`timeoutSec`, `graceSec`) are safety rails — always enforce them. A runaway agent process without a timeout can consume unbounded resources.

---

## 9. TranscriptEntry Kinds Reference

The full kinds/fields table is in [references/interfaces.md](references/interfaces.md).

---

## 10. Testing

Create tests in `server/src/__tests__/<adapter-name>-adapter.test.ts`. Test:

1. **Output parsing** — feed sample stdout through your parser, verify structured output
2. **Unknown session detection** — verify the `is<Agent>UnknownSessionError` function
3. **Config building** — verify `buildConfig` produces correct adapterConfig from form values
4. **Session codec** — verify serialize/deserialize round-trips

Run only the new adapter test file and `pnpm --filter @paperclipai/adapter-<name> typecheck` while iterating; leave repo-wide checks for the PR-ready hand-off described in `AGENTS.md`.

---

## 11. Minimal Adapter Checklist

- [ ] `packages/adapters/<name>/package.json` with four exports (`.`, `./server`, `./ui`, `./cli`)
- [ ] Root `index.ts` with `type`, `label`, `models`, `agentConfigurationDoc`
- [ ] `server/execute.ts` implementing `AdapterExecutionContext -> AdapterExecutionResult`
- [ ] `server/test.ts` implementing `AdapterEnvironmentTestContext -> AdapterEnvironmentTestResult`
- [ ] `server/parse.ts` with output parser and unknown-session detector
- [ ] `server/index.ts` exporting `execute`, `testEnvironment`, `sessionCodec`, parse helpers
- [ ] `ui/parse-stdout.ts` with `StdoutLineParser` for the run viewer
- [ ] `ui/build-config.ts` with `CreateConfigValues -> adapterConfig` builder
- [ ] `ui/src/adapters/<name>/config-fields.tsx` React component for agent form
- [ ] `ui/src/adapters/<name>/index.ts` assembling the `UIAdapterModule`
- [ ] `cli/format-event.ts` with terminal formatter
- [ ] `cli/index.ts` exporting the formatter
- [ ] Registered in `server/src/adapters/registry.ts`
- [ ] Registered in `ui/src/adapters/registry.ts`
- [ ] Registered in `cli/src/adapters/registry.ts`
- [ ] Added to workspace in root `pnpm-workspace.yaml` (if not already covered by glob)
- [ ] Tests for parsing, session codec, and config building
