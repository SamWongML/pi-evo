# Pi 0.83.0 Harness Reality Check

_Verification of `docs/researches/self-evolving-pi-agents.md` (v2) against the installed Pi 0.83.0 package · resolves issue #2_

## Method and sources

Verified against the actually-installed package, not web docs, unless noted:

- `~/.bun/install/global/node_modules/@earendil-works/pi-coding-agent/` — version **0.83.0** (`pi --version`)
- Type definitions (highest-trust source for event shapes): `dist/core/extensions/types.d.ts` (1271 lines — the full `ExtensionAPI`/event union), `dist/core/session-manager.d.ts`, `dist/core/compaction/compaction.d.ts`, `dist/core/tools/bash.d.ts`
- Implementation (grepped when docs/types were ambiguous): `dist/core/agent-session.js`, `dist/core/extensions/runner.js`, `dist/core/extensions/loader.js`, `dist/core/system-prompt.js`, `dist/core/tools/bash.js`, `~/.bun/install/global/node_modules/@earendil-works/pi-ai/dist/api/anthropic-messages.js`
- Docs shipped with the package: `docs/extensions.md`, `docs/session-format.md`, `docs/sessions.md`, `docs/compaction.md`, `docs/skills.md`, `docs/security.md`, `docs/settings.md`, `docs/packages.md`, `docs/usage.md`
- A real session JSONL, generated locally via `SessionManager.create()` against a throwaway `--session-dir` (see §3) — this machine had zero prior Pi sessions, confirmed by `~/.pi/agent/sessions` being absent before this work
- A working extension actually loaded and run by `pi -e` (see §5, native modules) — this is an executed test, not a read of documentation
- npm registry lookups for ecosystem packages (§6)

Every claim below cites a file path and line number, or a doc quote, or an executed command's output.

---

## 1. Verdict table — hooks

| Hook / API | Verdict | Evidence |
|---|---|---|
| `before_agent_start` returns `{ message }` to inject content | **VERIFIED**, with a load-bearing nuance on placement (see §2) | `BeforeAgentStartEventResult` — `types.d.ts:800-804`; consumed at `agent-session.js:881-906` |
| `before_agent_start` can chain `systemPrompt` overrides | **VERIFIED** | `types.d.ts:802-803`: "Replace the system prompt for this turn. If multiple extensions return this, they are chained."; runner chains it across extensions at `runner.js:862-865` |
| `tool_result` shape: `isError`, `content`, `details`, `input`, `toolName` | **VERIFIED**, but `details.exitCode` is **DIFFERENT** (see §2.2) | `ToolResultEventBase` — `types.d.ts:691-699`; per-tool `details` narrowing `types.d.ts:700-733` |
| `session_before_compact` exposes `preparation.messagesToSummarize`, `tokensBefore`, `fileOps` | **VERIFIED**, exact field names | `CompactionPreparation` — `compaction.d.ts:111-127` |
| `agent_settled` vs `agent_end` vs `turn_end` | **VERIFIED** — v2's framing is exactly right | See §2.3 |
| `session_shutdown` reason values `quit`/`reload`/`new`/`resume`/`fork` | **VERIFIED**, exact union | `SessionShutdownEvent` — `types.d.ts:463-468` |
| `input` hook, `{ action: "continue" }` | **VERIFIED**, and richer than v2 knew | `InputEvent`/`InputEventResult` — `types.d.ts:627-647`. Result is a 3-way union: `continue` \| `transform` (rewrite text/images) \| `handled` (suppress default handling entirely) — v2 only used `continue` |
| `context` — pre-LLM message list mutable | **VERIFIED**, and it's a genuine deep copy | `emitContext()` calls `structuredClone(messages)` — `runner.js:746`; handlers mutate in place or return `{messages}` to replace, chained — `runner.js:751-757` |
| `tool_call` fires before execution, `event.input` mutable, `{ block, reason }` | **VERIFIED** verbatim | `types.d.ts:684-690` comment: "`event.input` is mutable. Mutate it in place... Later `tool_call` handlers see earlier mutations. No re-validation is performed after mutation."; result type `types.d.ts:778-782` |
| `pi.registerTool` — `promptSnippet`/`promptGuidelines` fields exist | **VERIFIED** | `ToolDefinition` — `types.d.ts:350-353` |
| Setting `promptSnippet`/`promptGuidelines` rebuilds the system prompt | **DIFFERENT** — the trigger is narrower than v2 implies (see §2.4) | `agent-session.js:625-644` |
| `pi.appendEntry` persists extension state outside LLM context | **VERIFIED** verbatim | `types.d.ts:922-923`: "not sent to LLM"; `CustomEntry` — `session-manager.d.ts:59-73`: "Does NOT participate in LLM context (ignored by buildSessionContext)" |
| `resources_discover` returns `{ skillPaths, promptPaths, themePaths }` | **VERIFIED** exact shape | `ResourcesDiscoverEvent`/`Result` — `types.d.ts:402-413`; confirmed to trigger a system-prompt rebuild picking up new skills at `agent-session.js:1764-1780` |
| `resources_discover` reason values | **DIFFERENT (narrower)** — only `"startup" \| "reload"`, not the 5-value `session_start` union | `types.d.ts:406` vs `SessionStartEvent.reason` `types.d.ts:418` |
| `pi.exec` | **VERIFIED** | `ExtensionAPI.exec` — `types.d.ts:930-931` |
| `ctx.sessionManager` (read-only), `.getSessionFile()`, `.getSessionId()` | **VERIFIED** | `ExtensionContext.sessionManager: ReadonlySessionManager` — `types.d.ts:219`; `ReadonlySessionManager` Pick-list includes both — `session-manager.d.ts:140` |
| `ctx.getContextUsage()` | **VERIFIED**, `.tokens` can be `null` | `ContextUsage` — `types.d.ts:193-199`: "or null if unknown (e.g. right after compaction...)" — v2's code (`ctx.getContextUsage()?.tokens`) doesn't handle the null case, only the undefined one |
| `ctx.reload()` | **DIFFERENT (narrower scope)** — not callable from event hooks or tools at all (see §2.5) | `ExtensionCommandContext.reload()` — `types.d.ts:289-290`, absent from base `ExtensionContext`; `extensions.md:1299` |
| `withFileMutationQueue` | **VERIFIED** | Signature `withFileMutationQueue<T>(filePath: string, fn: () => Promise<T>): Promise<T>` — `dist/core/tools/file-mutation-queue.d.ts:5`; usage note at `extensions.md:1867` |
| `SessionManager.listAll()` / `.open()` | **VERIFIED** as static methods | `session-manager.d.ts:325,353-354` |

## 2. The five highest-priority items, in depth

### 2.1 `before_agent_start` injection and the prompt cache — the load-bearing question

**Mechanism, traced end-to-end in source:**

1. On every user-submitted turn, Pi builds `messages = [userMessage, ...pendingNextTurnMessages]` (`agent-session.js:864-880`).
2. It then calls `emitBeforeAgentStart(prompt, images, this._baseSystemPrompt, this._baseSystemPromptOptions)` (`agent-session.js:882`). The **already-built** system prompt string is passed in read-only; the handler does not see it being constructed.
3. Each extension's `before_agent_start` handler may return `{ message }`. The runner collects one message per handler into a `messages: [...]` array (`runner.js:856-866`).
4. Back in `agent-session.js:884-895`, each returned message is pushed onto the **same `messages` array, after the user's own message**: `messages.push({ role: "custom", customType, content, display, details, timestamp })`.
5. If a handler also returned `systemPrompt`, that replaces `agent.state.systemPrompt` for this turn only (`agent-session.js:898-901`); otherwise the base prompt is (re-)applied, undoing any override from a previous turn (`agent-session.js:902-906`).
6. Before the wire call, `convertToLlm()` normalizes `role: "custom"` messages into plain `role: "user"` text blocks (`dist/core/messages.js:89-96`).

**Consequence — this directly answers the ticket's question:**

- The injected content lands **as an additional `user`-role message appended after the user's own prompt**, not spliced immediately after the system block. "After the cached prefix" is the right mental model in the sense that it doesn't touch the system prompt at all — but it also isn't literally adjacent to the system prompt in the message list; it's the new tail of the conversation for that turn.
- Using `message` (not `systemPrompt`) **does not invalidate the provider prompt cache**, traced concretely for Anthropic in `pi-ai/dist/api/anthropic-messages.js`:
  - The system prompt gets its own `cache_control` breakpoint, independent of everything else (`anthropic-messages.js:740-748`).
  - Conversation history gets a second, separate rolling breakpoint: "Add cache_control to the last user message to cache conversation history" (`anthropic-messages.js:968` comment, logic at 968-990).
  - Since the injected block becomes the **new last user-role message** for this request, it receives *this turn's* cache breakpoint — it doesn't disturb the system-prompt breakpoint, and it doesn't require the previously-cached prefix (system + prior turns) to be recomputed. It's simply new, necessarily-uncached tail content, which is exactly how growing-conversation caching is supposed to work.
- **Using `systemPrompt` override is the actually dangerous path**: since it directly replaces the string that gets its own `cache_control` block, returning a different `systemPrompt` on a turn *will* miss the system-prompt cache for that request (a new prefix hash). v2 already warned "use sparingly" for this path — that warning is correct and should be stronger: prefer `message` exclusively for anything that varies turn to turn.
- Caveat: this cache-topology analysis is traced against the **Anthropic** provider file specifically. OpenAI and Google Vertex/Generative AI have their own caching code paths (`openai-completions.js`, `google-vertex.js`, `google-generative-ai.js` all reference caching) that were not traced in the same depth; the general principle (new tail content doesn't retroactively invalidate an already-cached prefix) should hold for any prefix-based cache, but provider-specific breakpoint placement wasn't independently confirmed for non-Anthropic providers.

**Verdict: VERIFIED**, with the correction that the injected message is a new trailing `user` turn, not a prefix insertion — and with confirmation that this is the safe path precisely because it avoids the `systemPrompt`-override footgun.

### 2.2 `tool_result` shape — `details.exitCode` for bash does not exist

`ToolResultEventBase` (`types.d.ts:691-699`) is:
```ts
{ type: "tool_result"; toolCallId: string; input: Record<string, unknown>;
  content: (TextContent | ImageContent)[]; isError: boolean; usage?: Usage }
```
narrowed per tool with a `details` field. For bash specifically:
```ts
export interface BashToolResultEvent extends ToolResultEventBase {
  toolName: "bash";
  details: BashToolDetails | undefined;
}
```
and `BashToolDetails` (`dist/core/tools/bash.d.ts:10-13`) is:
```ts
export interface BashToolDetails { truncation?: TruncationResult; fullOutputPath?: string; }
```
**There is no `exitCode` field anywhere in `BashToolDetails`.** Tracing the actual implementation (`dist/core/tools/bash.js:319-346`) confirms why: on a non-zero exit, the tool `throw`s a plain `Error` whose **message text** is the accumulated output with `\n\nCommand exited with code ${exitCode}` appended (line 344). That thrown error becomes `isError: true` with `content: [{type:"text", text: <that string>}]` — exit code is baked into the *text*, never a structured field. Confirmed independently in the locally-generated real session probe (§3): `"content":[{"type":"text","text":"hello\n\nCommand exited with code 1"}],"details":{},"isError":true"`.

**Verdict: DIFFERENT.** v2's `evolve-capture.ts` does `const exitCode = (event.details as any)?.exitCode;` — this is always `undefined` for bash. The failure-detection logic still works via `event.isError`, but any code path branching specifically on a numeric exit code needs to regex the text (`/Command exited with code (\d+)/`) instead of reading a field. This is a concrete bug in v2's reference implementation, not just a documentation gap.

### 2.3 `agent_settled` vs `agent_end` vs `turn_end` — v2's framing is correct

- `AgentEndEvent` (`types.d.ts:540-543`, comment "Fired when an agent loop ends") is emitted once per **internal** loop iteration at `agent-session.js:432-433`, inside a `while (await this._handlePostAgentRun())` loop (`agent-session.js:748`) that keeps going across retries, auto-compaction, and queued continuations.
- `AgentSettledEvent` (`types.d.ts:544-547`, comment: "Fired after an agent run has fully settled and no automatic retry, compaction, or queued continuation will run") is emitted exactly once, **after** that while-loop exits (`agent-session.js:755`, `_emitAgentSettled()` at line 314-318).
- `TurnEndEvent` (`types.d.ts:554-560`) carries `turnIndex`, the terminal `message`, and `toolResults` — fired once per individual assistant-message-plus-tool-round, i.e. potentially many times within one settle cycle.

**Verdict: VERIFIED.** `agent_end` can and does fire multiple times per user turn; `agent_settled` is the correct one-shot "truly done" signal, exactly as v2 states, and this is confirmed structurally (not just by doc comment) by the loop that wraps `agent_end` emission but not `agent_settled` emission.

### 2.4 `promptSnippet`/`promptGuidelines` and system-prompt rebuild — narrower trigger than implied

The system prompt is built by `buildSystemPrompt()` (`dist/core/system-prompt.js:7`) from `selectedTools`, `toolSnippets`, `promptGuidelines`, skills, context files, etc. `_rebuildSystemPrompt(toolNames)` (`agent-session.js:710-739`) reassembles the whole string from scratch, and only three call sites trigger it:

1. Initial construction/tool-set binding (`agent-session.js:643`).
2. `setActiveToolsByName()` — doc comment on the method itself: "Also rebuilds the system prompt to reflect the new tool set. Changes take effect on the next agent turn." (`agent-session.js:628-629,643-644`).
3. `extendResourcesFromExtensions()`, called right after `session_start` at startup and on `/reload` — i.e. driven by `resources_discover` (`agent-session.js:1764-1780`).

**There is no per-turn rebuild.** A tool registered once at extension load, with `promptSnippet`/`promptGuidelines` set at registration time and never toggled, is baked into the system prompt exactly once and stays byte-identical for the rest of the session — it does **not** get rebuilt on every turn.

**Verdict: DIFFERENT (refinement, not contradiction).** v2's mitigation — "set once at load, never toggled" — is exactly correct and is validated by this trace. But the trigger v2's prose implies ("activating a tool... rebuilds the system prompt") is better stated as: the rebuild happens when something calls `setActiveTools`/`ctx.setActiveTools()` (toggling the *active* tool set, e.g. via a `/tools` command or an extension dynamically enabling/disabling tools) or when `resources_discover` adds new skills/prompts on `/reload` — not from the mere existence of `promptSnippet`/`promptGuidelines` on a tool that stays active throughout.

### 2.5 `ctx.reload()` — only callable from a registered command, not from any hook or tool

`reload(): Promise<void>` exists only on `ExtensionCommandContext` (`types.d.ts:289-290`), which extends the base `ExtensionContext` — it is **absent** from the plain `ExtensionContext` passed to every `pi.on(...)` event handler (`ExtensionHandler<E,R> = (event: E, ctx: ExtensionContext) => ...`, `types.d.ts:851`) and from the `ctx` passed to `ToolDefinition.execute()` (also typed `ExtensionContext`, `types.d.ts:371`). The doc confirms this explicitly: "Tools run with `ExtensionContext`, so they cannot call `ctx.reload()` directly. Use a command as the reload entrypoint, then expose a tool that queues that command as a follow-up user message." (`docs/extensions.md:1299`, worked example follows at 1301-1324).

**Verdict: DIFFERENT (narrower than v2's gotcha table implies).** v2's gotcha ("`ctx.reload()` is terminal for its handler" / "`await ctx.reload(); return;`") is real advice for the one place `reload()` actually exists (a `registerCommand` handler) but reads as if it's callable from any hook. It isn't — a `tool_call`/`agent_settled`/etc. handler, or a tool's `execute()`, cannot call `ctx.reload()` at all; the only path is register a command, then have a tool queue that command via `pi.sendUserMessage("/your-reload-cmd", { deliverAs: "followUp" })` as the doc's example shows.

---

## 3. Session JSONL format — verified by generating real sessions

This machine had **zero** Pi sessions before this work (`~/.pi/agent/sessions` did not exist). Rather than guess, a throwaway session was generated by calling the real `SessionManager` API directly against every append method it exposes (`SessionManager.create()`, `.appendMessage()`, `.appendCustomEntry()`, `.appendCustomMessageEntry()`, `.appendLabelChange()`, `.appendSessionInfo()`, `.appendModelChange()`, `.appendThinkingLevelChange()`, `.appendCompaction()`, `.branch()`, `.branchWithSummary()`), writing to an isolated `--session-dir`, never touching the user's real session store. Resulting JSONL (line-numbered):

```jsonc
{"type":"session","version":3,"id":"019fb8b8-...","timestamp":"...","cwd":"/tmp/probe-cwd-example"}
{"type":"message","id":"32945ab8","parentId":null,"timestamp":"...","message":{"role":"user","content":"Say the single word: hello","timestamp":1785510512112}}
{"type":"message","id":"70e912b5","parentId":"32945ab8","timestamp":"...","message":{"role":"assistant","content":[{"type":"text","text":"hello"}],"api":"messages","provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop","timestamp":...}}
{"type":"message","id":"adb25ddb","parentId":"70e912b5","timestamp":"...","message":{"role":"toolResult","toolCallId":"call_probe_1","toolName":"bash","content":[{"type":"text","text":"hello\n\nCommand exited with code 1"}],"details":{},"isError":true,"timestamp":...}}
{"type":"message","id":"cf74e1ac","parentId":"adb25ddb","timestamp":"...","message":{"role":"bashExecution","command":"echo hi","output":"hi","exitCode":0,"cancelled":false,"truncated":false,"timestamp":...}}
{"type":"custom","customType":"evolve-probe","data":{"count":42,"note":"outside LLM context"},"id":"9bc19596","parentId":"cf74e1ac","timestamp":"..."}
{"type":"custom_message","customType":"evolve-recall","content":"<prior-experience>...","display":true,"details":{"lesson_ids":["abc123"]},"id":"36cb3cb2","parentId":"9bc19596","timestamp":"..."}
{"type":"label","id":"4724055d","parentId":"36cb3cb2","timestamp":"...","targetId":"32945ab8","label":"checkpoint-1"}
{"type":"session_info","id":"64c85790","parentId":"4724055d","timestamp":"...","name":"Probe session"}
{"type":"model_change","id":"68ef6f07","parentId":"64c85790","timestamp":"...","provider":"anthropic","modelId":"claude-sonnet-4-5"}
{"type":"thinking_level_change","id":"745e8679","parentId":"68ef6f07","timestamp":"...","thinkingLevel":"high"}
{"type":"compaction","id":"66f618ce","parentId":"745e8679","timestamp":"...","summary":"...","firstKeptEntryId":"70e912b5","tokensBefore":5000,"details":{"readFiles":["a.ts"],"modifiedFiles":["b.ts"]},"fromHook":true}
{"type":"branch_summary","id":"4e4c9648","parentId":"70e912b5","timestamp":"...","fromId":"70e912b5","summary":"Abandoned approach: tried X, didn't work.","details":{...},"fromHook":true}
```

Cross-checked field-for-field against `dist/core/session-manager.d.ts:1-140` and `docs/session-format.md` (which enumerates the identical 9 entry types at its own `## Entry Types` section).

| Item | Verdict | Detail |
|---|---|---|
| Path layout `~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl` | **VERIFIED verbatim** | `docs/session-format.md:8` |
| `type` values: `session`, `message`, `compaction`, `branch_summary`, `custom`, `label` | **DIFFERENT (incomplete)** — real format has **9** top-level entry types, not 6 | Missing from v2: `custom_message`, `session_info`, `model_change`, `thinking_level_change` — all confirmed real, `SessionEntry` union at `session-manager.d.ts:105` |
| `custom` handles both in-context and out-of-context extension data | **DIFFERENT** — these are two distinct entry types | `CustomEntry` (`type:"custom"`) explicitly does **not** participate in LLM context (`session-manager.d.ts:59-67`); `CustomMessageEntry` (`type:"custom_message"`) explicitly **does** — "content is converted to a user message in `buildSessionContext()`" (`session-manager.d.ts:85-96`). `pi.appendEntry()` produces the former; `pi.sendMessage()` produces the latter. |
| `id`/`parentId` tree | **VERIFIED** | Every entry extends `SessionEntryBase { type, id, parentId, timestamp }` (`session-manager.d.ts:17-22`) |
| Message roles `user`, `assistant`, `toolResult`, `bashExecution`, `custom`, `branchSummary`, `compactionSummary` | **VERIFIED at the `AgentMessage` level** — but these are in-memory/context roles, not JSONL entry `type`s (see row above); the mapping from JSONL entry to `AgentMessage.role` happens via `sessionEntryToContextMessages()` (`session-manager.d.ts:151`) and `convertToLlm()` (`dist/core/messages.js:75-122`, handles exactly this role set at lines 78-119) | — |
| `label` shape `{targetId, label}` | **VERIFIED verbatim** | `LabelEntry` — `session-manager.d.ts:75-79` |
| `compaction` shape | **DIFFERENT** — no `retainedTail` field; real field is `firstKeptEntryId` (a single UUID pointer, not an array of retained messages) | `CompactionEntry` — `session-manager.d.ts:36-47` |

---

## 4. Gotchas table — confirmed against the shipped docs

| Gotcha | Verdict | Evidence |
|---|---|---|
| `defaultProjectTrust` default `"ask"` silently ignores project resources in non-interactive mode (`-p`, `--mode json`/`rpc`) | **VERIFIED verbatim** | `docs/security.md:29`: "Non-interactive modes (`-p`, `--mode json`, and `--mode rpc`) do not show a trust prompt. Without an applicable saved trust decision, `defaultProjectTrust: "ask"` and `"never"` ignore such resources, while `"always"` trusts them." |
| No background bash, no built-in cron/daemon | **VERIFIED verbatim** (see §7 for full treatment) | `docs/usage.md:296`: "It intentionally does not include built-in MCP, sub-agents, permission popups, plan mode, to-dos, or background bash." |
| Tool results truncated to 2000 chars during compaction | **VERIFIED verbatim** | `docs/compaction.md:269`: "Tool results are truncated to 2000 characters during serialization." |
| Tools run in parallel by default | **VERIFIED verbatim** | `docs/extensions.md:1867`: "tool calls run in parallel by default"; also `extensions.md:757`: "In the default parallel tool execution mode, sibling tool calls from the same assistant message are preflighted sequentially, then executed concurrently." |
| `withFileMutationQueue` is the fix | **VERIFIED** | `docs/extensions.md:1867`; signature confirmed §1 |
| `session_shutdown` fires on `/reload`, `/new`, `/resume`, `/fork` too | **VERIFIED** | `SessionShutdownEvent.reason` union, `types.d.ts:465` |
| Skill name collisions warn and keep the first found | **VERIFIED verbatim** | `docs/skills.md:188`: "Name collisions (same name from different locations) warn and keep the first skill found." |
| Native module ABI mismatch (`better-sqlite3`) | **VERIFIED as a real-world risk, mechanism refined** | See §5 — the mechanism is Node ABI (`NODE_MODULE_VERSION`) mismatch between whatever Node built the addon and whatever Node runs Pi, not a Bun-vs-Node issue in this install |
| Sessions are local to each device | **VERIFIED** | Confirmed by path layout (§3); no shared/sync facility exists in Pi itself |
| `ctx.reload()` terminal-for-handler advice | **VERIFIED but scope is narrower** — only reachable from commands | See §2.5 |

---

## 5. Can a Pi extension load a native Node module? — YES, tested directly

**Established facts, confirmed:**
- `pi`'s bin entry is `dist/cli.js` (`package.json: "bin": {"pi": "dist/cli.js"}`), and `dist/cli.js` opens with `#!/usr/bin/env node` (`dist/cli.js:1`). Despite `bun` being the install mechanism (`~/.bun/bin/pi`), **Pi runs under Node**, not as a Bun process, for this install.
- `package.json: "engines": {"node": ">=22.19.0"}`. Installed Node is **v22.17.0** (`node --version`) — below the declared minimum. Pi runs anyway; nothing in the launcher enforces the engines field.
- `jiti` `2.7.0` is a direct dependency (`package.json:53`) and is exactly what loads extensions: `dist/core/extensions/loader.js:2` ("Extension loader - loads TypeScript extension modules using jiti"), `loader.js:14` (`import { createJiti } from "jiti/static"`), `loader.js:332` (`jiti.import(extensionPath, ...)`).
- **Critically**, `loader.js:325-331` branches on `isBunBinary`:
  ```js
  const jiti = createJiti(import.meta.url, {
    moduleCache: false,
    ...(isBunBinary ? { virtualModules: VIRTUAL_MODULES, tryNative: false } : { alias: getAliases() }),
  });
  ```
  `isBunBinary` (`dist/config.js:16`) is `import.meta.url.includes("$bunfs")` etc. — true **only** for a compiled Bun single-file binary. This bun-global install is not that; `isBunBinary` is false, so jiti runs in its normal Node/dev mode with `tryNative` left at its default (not disabled) and dependency resolution via `alias` (a map to real `node_modules` paths), not `virtualModules`.
- Practical consequence: since `tryNative` is not disabled in this configuration, jiti falls through to Node's native `require`/`import` for anything that doesn't need TypeScript transformation — which is exactly the path a `.node` binary addon or a Node builtin module takes.

**Executed test** (not just inferred from source): wrote an extension at a scratch path that does, inside `session_start`:
```ts
const { DatabaseSync } = require("node:sqlite");
const db = new DatabaseSync(":memory:"); db.exec("CREATE TABLE t (id INTEGER)");
// ...
const bsq = require("better-sqlite3");
const db2 = new bsq(":memory:"); db2.exec("CREATE TABLE t (id INTEGER)");
```
`better-sqlite3` was `npm install`ed fresh (with its prebuilt binary compiled for the exact Node running Pi) right next to the extension file. Ran:
```
$ pi --no-session --session-dir ./sessions -e ./native-test-ext.ts -p "reply with the word done" --offline
NATIVE_TEST_RESULTS: {
  "node:sqlite via require()": "OK",
  "better-sqlite3 via require()": "OK"
}
```
Both loaded successfully — a real `.node` N-API addon (`better-sqlite3`) and Node's built-in `node:sqlite` both work, required from inside a Pi extension loaded the normal way (`pi -e`, which uses the same `loader.js` path as auto-discovered extensions).

**`node:sqlite` standalone** also confirmed directly in plain Node: works, but emits `(node:xxxx) ExperimentalWarning: SQLite is an experimental feature and might change at any time` — it is functional on Node 22.17.0 but its API is explicitly unstable upstream, a real consideration for a store meant to survive Pi/Node upgrades.

**Verdict on the gating question:**
- **Native `.node` addons load fine** from a Pi extension in this installation, because Pi runs as plain Node (`#!/usr/bin/env node`) and jiti's native-import fallback is only disabled in the Bun-compiled-binary distribution mode, which this is not.
- The real risk is the one hermes-memory's own README already documents and this doesn't contradict: **ABI mismatch** — `better-sqlite3` (or any native addon) must be built against the exact Node `NODE_MODULE_VERSION` that ends up running Pi. Their README states this plainly (fetched during this work): "`better-sqlite3` is a native addon. If Pi is installed via Homebrew and the extension was compiled for a different Node ABI, session search may warn: `was compiled against a different Node.js version using NODE_MODULE_VERSION ...`" and documents a `npm rebuild better-sqlite3` auto-recovery plus a manual fallback. This is an install-topology risk (which Node built the addon vs which Node launches Pi), not a jiti/extension-loading limitation.
- `node:sqlite` sidesteps the ABI-mismatch risk entirely (it's compiled into the Node binary, not a separate addon needing a rebuild) but is Experimental and its surface has shifted across Node versions; `better-sqlite3` is the more battle-tested choice at the cost of the ABI-pinning operational burden.
- Given the target scale (1-3 machines, single operator), the ABI risk is manageable (one or a few machines to keep Node/Pi pinned consistently on), and the sibling ticket's SQLite + `sqlite-vec` recommendation is **not blocked** by extension loading. `sqlite-vec` itself ships as a loadable extension for both `better-sqlite3` and `node:sqlite`, so either substrate is viable; the native-addon path was not independently tested here but follows the same `better-sqlite3` loading mechanism just confirmed.
- Caveat/limit of this test: this only demonstrates the **current bun-global install topology** (`#!/usr/bin/env node`, not `isBunBinary`). If Pi ever ships or gets invoked as a compiled Bun single-file binary, `tryNative: false` would be forced and native `.node` addons would very likely fail to load through jiti in that mode (untested — no compiled Bun binary of Pi was available to verify this negative case directly, but it follows directly from `loader.js:325-331`, and `VIRTUAL_MODULES` in that mode is a fixed allowlist that does not include any native-addon-backed package).

---

## 6. Does Pi have any scheduling or background execution? — ABSENT, confirmed

No hits for cron, daemon, timer, or scheduler as a Pi *feature* anywhere in the shipped docs (grepped all of `docs/*.md`). The one explicit, on-point statement:

> "It intentionally does not include built-in MCP, sub-agents, permission popups, plan mode, to-dos, or background bash. You can build or install those workflows as extensions or packages, or use external tools such as containers and tmux." — `docs/usage.md:296`

Cross-checked against the CLI surface (`pi --help`) and the full `ExtensionAPI` (`types.d.ts`): there is no `pi run --daemon`, no cron-like registration API, no equivalent of a "background task" hook. The closest thing to backgrounding is:
- `pi.exec()` — a promise-returning subprocess call, awaited synchronously by whatever handler calls it; not fire-and-forget scheduling.
- Extensions can spawn detached child processes themselves via Node's `child_process` (nothing stops this), but Pi provides no primitive for it and no supervision if the parent `pi` process exits.

**Verdict: ABSENT, exactly as documented.** A periodic (nightly, hourly, etc.) curator/consolidation job needs an OS-level scheduler external to Pi: `cron`, a `systemd` timer/user service, `launchd` on macOS, or (per v2's own AWS framing, out of scope at 1-3-machine solo scale) EventBridge Scheduler. Nothing here contradicts v2's own gotcha table entry ("No background bash, no built-in cron / Deliberate"), which was already correctly ABSENT-flagged.

---

## 7. Ecosystem survey — what already exists

Five current packages were examined (READMEs fetched from their npm listings / GitHub during this work; versions confirmed live via `npm view <pkg> version`).

### 7.1 `pi-hermes-memory` (chandra447) — v0.9.2 — the strongest existing base

Ported directly from Nous Research's Hermes agent (`tools/memory_tool.py`, `run_agent.py`, `agent/memory_provider.py`, `agent/memory_manager.py` — credited in its own README). Config file `~/.pi/agent/hermes-memory-config.json`.

| Loop | Coverage |
|---|---|
| **Recall** | Two-tier (global `~/.pi/agent/pi-hermes-memory/` + project `~/.pi/agent/projects-memory/<project>/`). Default `memoryMode: "policy-only"` injects only a `<memory-policy>` block telling the agent to call `memory_search`/`session_search`, not the raw memory files — deliberately keeps first-turn tokens low. `legacy-inject` mode restores full injection. Session history is FTS5-searchable (SQLite, `better-sqlite3`). |
| **Reflect** | "Background Learning" reviews every N turns (default 10) OR every M tool calls (default 15), whichever comes first — an activity-based nudge, not a fixed cadence. Correction detection saves immediately on phrases like "no, use pnpm". Failure/correction/insight/preference/convention/tool-quirk are distinct categories. |
| **Curate** | Auto-consolidation: when a Markdown store (5,000-char cap) is full, spawns a one-shot LLM call to merge/prune before retrying the write; falls back to error if consolidation can't free space (does **not** silently spill to SQLite-only storage — a real limitation the README states outright). No cross-device merge logic — this is explicitly per-instance. |
| **Cache preservation** | `reviewTransport: "direct"` (default) uses an in-process `completeSimple()` side-channel instead of spawning a child `pi -p` process for review/flush/correction/consolidation, explicitly to keep "the main session's system prompt, tools, and LLM prefix cache intact" — this is a real, working claim, not aspirational; falls back to the legacy subprocess path only on failure. |
| **Skills** | `skill_manage` tool (create/view/patch/update/delete), discovered via `resources_discover` for project-scoped skills — the exact hook v2 relies on for dynamic skill paths, confirmed in active third-party use. Duplicate/collision guards on skill creation (exact-slug block, near-name+similar-description block, near-name+different-description rename prompt). |
| **Security** | All memory/skill writes pass a content scanner before acceptance, explicitly to prevent prompt-injection-via-stored-memory. |

**Verdict:** implements a large fraction of Recall and Reflect, a meaningful (but single-device) slice of Curate. **Viable base for the single-device layer**, not a fleet/multi-scope solution — it has no concept of scope predicates, provenance chains, or supersession (all the things v2's §1.7 fleet-memory research says matter). At 1-3-machine solo scale, most of that gap may not matter; the real gap for pi-evo's v0 is (per v2's own §11) failure→fix *pairing* (Hermes stores failures, not failure-then-later-success pairs) and eval-gated promotion.

### 7.2 `pi-self-learning` (Matteo Collina) — v0.5.0 — git-backed digest, exactly as v2 described

`daily/YYYY-MM-DD.md` → `monthly/YYYY-MM.md` → `core/CORE.md` (frequency+recency-scored via `core/index.json`) → `long-term-memory.md`, all committed to a dedicated git repo at `.pi/self-learning-memory/` (project) or `~/.pi/agent/self-learning-memory/` (global). Runs `autoAfterTask` (after each fully-completed task; explicitly skips user-aborted/incomplete turns), extracting what went wrong and how it was fixed, including "blocked commands and permission denials as operational constraints." Injects `CORE.md` into context by default (`context.includeCore`), with an `instructionMode: "strict"` option that adds explicit policy telling the agent to consult `CORE.md` first, then daily/monthly logs, then `long-term-memory.md` if stuck.

**Verdict:** covers Reflect (task-level, automatic) and a real Curate step (frequency/recency-ranked promotion into `CORE.md`, git-committed — i.e., versioned and revertible for free). No Recall retrieval beyond "always inject CORE.md" (no query-based search, no scoping) and no cross-device sharing (the git repo is local unless the operator pushes it somewhere, which the extension itself doesn't do). **A viable model for the T1/T2-adjacent "durable rules" layer**, not a lesson-store replacement.

### 7.3 `pi-observational-memory` — v3.0.3 — a different problem, worth knowing about

Not a lesson/memory-across-sessions store at all — it targets **within-session continuity across compactions**. Captures "Observations" (timestamped events) and distills "Reflections" (durable facts) continuously *during* the session, so that when compaction fires, the summary is already prepared instead of being computed synchronously at the moment of context pressure ("Compaction becomes a fast rendering step instead of a slow summarization event"). V3 is a breaking rewrite from V2 per its own README warning.

**Verdict:** doesn't touch Recall/Reflect/Curate as v2 defines them (those are cross-session); it's a complementary answer to a different pain point (compaction quality/latency within one long session) that v2's own §1.6(d) and gotcha table flag as a real problem ("Context rot", "Compaction destroys evidence"). Worth citing as prior art for *how* to make `session_before_compact` handling cheap, not as a competing design for the lesson store.

### 7.4 `@remnic/plugin-pi` — thin connector to an external always-on daemon, not self-contained

Requires a separately-running `remnic daemon` (`remnic daemon start`) reachable over HTTP/MCP (default `http://127.0.0.1:4318`); the Pi extension itself is a thin client that recalls via Pi's `context` hook (not `before_agent_start`), observes turns, flushes memory before compaction, and exposes Remnic's own MCP tools as Pi tools. Config supports `recallMode` (`auto`/`minimal`/`full`/`graph_mode`/`no_recall`), `recallTopK`, `recallBudgetChars`, and a self-disabling circuit breaker (`recallTimeoutThreshold` over a rolling `recallTimeoutWindow`) that permanently turns off recall after repeated daemon timeouts — a real production safeguard worth stealing regardless of what store gets built.

Notable side-finding: its README documents a **Pi-API-compatible fork called "Oh My Pi" (`omp`, at omp.sh)** that "preserves Pi's extension API" — i.e., there is at least one fork in the wild extensions must sometimes account for, and it currently has a documented loader bug (`omp` builds ≥v16.3.5 can't resolve an npm-installed extension's own `node_modules`, e.g. `@sinclair/typebox`) that this package works around by pre-bundling with `bun build`. Not directly relevant to pi-evo's Pi-only v0 scope, but worth knowing the fork exists.

**Verdict:** architecturally interesting (recall via `context` rather than `before_agent_start`, external daemon rather than in-process store) but not a viable base for a 1-3-machine *single-operator* setup unless the operator wants to also stand up and operate a separate always-on Remnic daemon — which is exactly the kind of extra moving part v2's own solo-scale framing argues against.

### 7.5 `@ryan_nookpi/pi-extension-memory-layer` (part of `Jonghakseo/pi-extension`) — v0.3.4 — minimal, explicit-only

`remember`/`recall`/`forget`/`memory_list` tools plus `/remember` and `/memory` slash commands. Storage is plain Markdown under `~/.pi/memory/{user,projects/<project-id>}/`. Project identity resolves via git remote → root commit hash → cwd-hash fallback. The memory index (not full content) is injected into the system prompt every turn so the agent knows what's stored without an explicit recall call.

**Verdict:** essentially zero automatic capture (Reflect) — everything is an explicit tool call the LLM must decide to make — and no curation beyond manual `forget`. Useful only as a UX reference (the `/memory` overlay browser, the project-ID resolution fallback chain) — not a base.

### 7.6 Net read on "buy vs. build" at solo scale

v2's own week-1 recommendation (`pi install npm:pi-hermes-memory` + steal `pi-self-learning`'s layout) holds up well and is **strengthened**, not weakened, by this survey: nothing found since v2 was written closes the specific gaps v2 already named (cross-device sharing, governed scope/provenance/supersession, T4 code-artifact promotion, failure→fix pairing, eval-gated promotion). At 1-3-machine solo scale, though, "cross-device sharing" and "governed scope" are largely moot (single operator, 1-3 machines is not a fleet), which means the actual delta v0 needs to build shrinks to: **failure→fix pairing** and **eval-gated promotion** — both of which no surveyed package does, hermes-memory included.

---

## 8. Summary — what v2 got right, what needs correction

**Right, and now independently confirmed against source (not just docs):** `before_agent_start`'s `message` injection path and its cache-safety, `agent_settled` as the correct terminal signal, `session_before_compact`'s exact preparation fields, `resources_discover`'s shape and its dynamic-skill-path effect, `tool_call`'s mutate-and-block semantics, `context`'s deep-copy mutability, `session_shutdown`'s reason union, the session path layout, the parallel-tool-execution default and `withFileMutationQueue` fix, the 2000-char compaction truncation, skill name-collision shadowing, `defaultProjectTrust` in non-interactive mode, and the "no background bash / no cron" absence. This is a high hit rate for a document that had never touched a real install.

**Needs correction:**
1. `details.exitCode` on bash `tool_result` does not exist — exit code is embedded in the result *text*, not a structured field. This breaks a specific code path in v2's own reference extension.
2. The real session JSONL has 9 entry types, not 6 — missing `custom_message` (distinct from, and more relevant to context-injection than, `custom`), `session_info`, `model_change`, `thinking_level_change`. The `custom`/`custom_message` split matters specifically for anything building on `pi.appendEntry` vs `pi.sendMessage`.
3. `compaction` entries carry `firstKeptEntryId` (a pointer), not a `retainedTail` array.
4. `ctx.reload()` is reachable only from a registered command, never from an event hook or a tool — a stronger constraint than "treat as terminal for its handler" implies.
5. The system-prompt rebuild that threatens the prompt cache is triggered specifically by `setActiveTools`/`resources_discover`, not by a tool merely carrying `promptSnippet`/`promptGuidelines`.

**The single most consequential finding:** native `.node` addons load without issue from Pi extensions in the current install topology (`#!/usr/bin/env node`, jiti in non-Bun-binary mode) — directly confirmed by getting `better-sqlite3` to open an in-memory database inside a real `pi -e` run. This fully un-gates the sibling ticket's SQLite + `sqlite-vec` store recommendation for v0; the only real operational risk is Node-ABI pinning (documented independently by `pi-hermes-memory`'s own README), which is a non-issue at 1-3-machine solo scale. The one caveat is that this guarantee is specific to the current bun-global/Node-launcher distribution — a hypothetical future compiled-Bun-binary distribution of Pi would force `tryNative: false` and likely break this.
