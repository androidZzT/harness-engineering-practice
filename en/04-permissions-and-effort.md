# How the Harness Controls the Agent: Permissions and Effort

The previous post "How to Extend the Harness: Skills, Config Directories, and Hooks" was about adding things to the harness — packaging reusable workflows as skills, writing constraints into config directories, attaching monitoring logic to hooks. This post shifts perspective: once you have equipped an agent with tools, how do you keep it in check?

Two concrete questions. First, when the agent modifies files, executes commands, or makes network requests, how do you decide which operations require human approval and which are automatically allowed? Second, for the same model, how do you tune "how deeply to think" by scenario, so you neither waste tokens on simple tasks nor shortchange complex ones?

Both CC and Codex have solutions for these two questions. As in the previous posts, the difference is not in who has what — it is **that given the same control problem, the two took different engineering paths**.

---

## 1. Permissions

### What problem this solves

In the course of running, an agent modifies files, executes commands, makes network requests. Fully open is risky; prompting for every single action kills efficiency. The core tension is: **how to dial in a balance between "fewer interruptions" and "staying in control", and make that balance fixable at the personal / project / enterprise level**.

### How Codex does it

Codex splits authorization into two orthogonal dimensions, combined into a permission matrix.

**Dimension one: sandbox policy**, defines the agent's operational boundaries. The `SandboxPolicy` enum type (`codex-rs/protocol/src/protocol.rs:878`), 4 variants:

- `DangerFullAccess` (serialized as `"danger-full-access"`) — unrestricted
- `ReadOnly { network_access: bool }` — read-only, network off by default
- `ExternalSandbox { network_access }` — already in an external sandbox; disk access opened, network follows the incoming setting
- `WorkspaceWrite { writable_roots, network_access, exclude_tmpdir_env_var, ... }` — adds write access to the current workspace and listed `writable_roots` on top of `ReadOnly`

**Dimension two: approval policy**, determines when to stop and ask the user. The `AskForApproval` enum type (`codex-rs/protocol/src/protocol.rs:784`), 5 variants:

- `UnlessTrusted` (`"untrusted"`) — only commands known to be safe and read-only are auto-approved; everything else is asked
- `OnFailure` (marked DEPRECATED) — auto-approve everything inside the sandbox; escalate to user on failure
- `OnRequest` (`#[default]`, the default) — the model decides when to ask
- `Granular(GranularApprovalConfig)` — fine-grained per-category toggles; `true` allows, `false` auto-rejects (does not surface to user)
- `Never` — never asks; failures are returned directly to the model without escalation

The `Granular` variant's `GranularApprovalConfig` (same file `protocol.rs`, immediately following the `AskForApproval` definition) has 5 category fields: `sandbox_approval` (shell command approval, including inline privilege escalation), `rules` (prompts triggered by execpolicy `prompt` rules), `skill_approval` (skill script execution), `request_permissions` (`request_permissions` tool trigger), and `mcp_elicitations` (MCP elicitation prompts).

The two dimensions' division of labor: sandbox policy manages "boundaries" (file read/write scope + network toggle); approval policy manages "interruption rhythm" (when to stop and ask). The two are independent and can be combined freely.

### How CC does it

CC builds authorization in three stacked layers: mode (baseline) + rules (allow/ask/deny) + classifier (auto mode real-time review).

**6 permission modes** (permission-modes page), Shift+Tab cycles through `default→acceptEdits→plan`:

- `default` — only read-only operations are not asked
- `acceptEdits` — reads + file edits + `mkdir`/`touch`/`mv`/`cp`/`rm`/`rmdir`/`sed` within the working directory are not asked
- `plan` — read-only; research and propose a plan, no source code modification
- `auto` — almost nothing is asked, but an independent classifier model reviews each action in real time before execution; marked research preview; requires Claude Code v2.1.83+, model must be Opus 4.6+ or Sonnet 4.6
- `dontAsk` — only pre-approved tools run; everything else is auto-rejected; suitable for locked-down CI or scripts
- `bypassPermissions` — skips all checks for everything, equivalent to `--dangerously-skip-permissions`; only for isolated containers/VMs; `rm -rf /` and `rm -rf ~` still trigger circuit-breaker prompts

Except for `bypassPermissions`, writes to protected paths (`.git`/`.claude`/`.vscode`/`.idea`, etc.) are never auto-approved.

**Three rule categories** (permissions page): `allow` (use without asking), `ask` (confirm each time), `deny` (prohibit). Evaluation order is deny → ask → allow; first match wins, deny always takes precedence.

One detail worth noting: a bare tool-name deny (e.g. `Bash`) removes the tool from context entirely; a scoped deny (e.g. `Bash(rm *)`) keeps the tool but blocks matching calls only.

**Settings 5-level precedence**: Managed (enterprise IT push, highest, cannot be overridden by any layer) > command-line arguments > Local (`.claude/settings.local.json`) > Project (`.claude/settings.json`) > User (`~/.claude/settings.json`). A deny at any layer cannot be overridden to allow at another layer.

**Auto mode classifier** (permission-modes page) has a few specifics:

An independent classifier model reviews each action before execution, intercepting three scenarios: escalations outside the requested scope, actions targeting unknown infrastructure, and actions driven by adversarial content encountered in context. The decision sequence is: first check allow/deny rules, directly approve read-only and in-working-directory edits, forward the rest to the classifier; when blocked, the reason is returned to the model.

Upon entering auto mode, broad allow rules are discarded (`Bash(*)`, wildcard interpreters, package manager run, `Agent` rules); they are restored on exit. The classifier sees only user messages, tool calls, and `CLAUDE.md` — tool results are stripped to prevent injection.

Fallback threshold: if blocked 3 consecutive times or 20 times cumulatively, auto mode pauses and falls back to per-request prompting (threshold is not configurable).

Boundaries stated verbally in the session ("don't push yet") are treated as block signals, but are not persisted — if compaction removes that message, they may no longer apply. For hard guarantees, write deny rules.

### Comparison

| Dimension | Codex | CC |
|-----------|-------|----|
| Core model | Two orthogonal dimensions: sandbox policy × approval policy | Three stacked layers: mode + rules + classifier |
| Granularity | 4 sandbox × 5 approval, combination matrix | 6 modes (including dontAsk full-pre-approve) |
| Finest-grain restriction | `Granular`: 5 independent category toggles | Scoped deny: `Bash(rm *)` granularity |
| Semantic review | None (deterministic rule combination) | Auto mode classifier, semantic judgment |
| Enterprise policy lockdown | Sandbox policy + approval policy constraint combination | Managed settings, highest in 5-level priority |

Codex leans toward "policy-based control": two deterministic, orthogonal dimensions, open-source verifiable, behavior predictable. CC leans toward "model-based control + policy fallback": auto mode uses an independent classifier for semantic judgment ("does this exceed scope / does this target unknown infrastructure / is this driven by injected content"), with deny rules as the hard safety net.

### Practical advice

**Codex**: day-to-day use `WorkspaceWrite` + `OnRequest` (the default combination); for unattended batch runs use `Never`, but always pair with a tightened sandbox (network off, `writable_roots` narrowed); `Granular` fits fine-grained scenarios like "allow shell but block MCP / skill scripts".

**CC**: review-oriented coding use `acceptEdits`, review changes after the fact with an editor or `git diff`; to reduce interruptions on long tasks use `auto`, but it is research preview — sensitive operations still need a human pass; for CI use `dontAsk` + a pre-approved allow list; `bypassPermissions` only in isolated containers.

**Universal**: lock enterprise security boundaries with Managed or the constraint layer — do not try to override with the project layer; write hard boundaries as deny rules, do not rely on verbal statements in the conversation (compaction may delete those messages).

---

## 2. Effort

### What problem this solves

The same model can "think briefly and answer quickly" or "think deeply and answer thoroughly". Too much thinking on a simple task wastes tokens and may overthink; too little on a complex task is insufficient. What is needed is a dial for "how deeply to think" that can be calibrated per model and set as a default per scenario.

### How Codex does it

**`ReasoningEffort` enum with 6 levels** (`codex-rs/protocol/src/openai_models.rs:45`):

`None` / `Minimal` / `Low` / `Medium` (`#[default]`) / `High` / `XHigh`, serialized as `"none"/"minimal"/"low"/"medium"/"high"/"xhigh"`, default `Medium`.

**The client does exactly one thing: pass it through.** `build_reasoning` (`codex-rs/core/src/client.rs:715`) assembles the effort into the request; the key line is `effort: effort.or(model_info.default_reasoning_level)` (`:722`), which ultimately becomes `{"reasoning":{"effort":"high"}}` in the OpenAI Responses API request body.

The Codex client does not modify the system prompt, does not alter the tool list, does not adjust the context window, and adds no local reasoning enhancements. It sends a string to the server; how "high" translates to actual compute is the cloud model's concern.

**Three-level fallback** (`effective_reasoning_effort`, `codex-rs/core/src/session/turn_context.rs:125`): user-set value for the current turn → model default level (`default_reasoning_level`) → if the model does not support reasoning, the entire `reasoning` field is omitted (note: the field is omitted, not sent as `"none"`).

**Level mapping across model switches**: `effort_rank` (`openai_models.rs:557-562`, None=0 / Minimal=1 / Low=2 / Medium=3 / High=4 / XHigh=5) combined with `nearest_effort` (`:566`), maps by minimum absolute rank difference to the closest level the target model supports.

**Plan mode independent effort**: `plan_mode_reasoning_effort` (`codex-rs/core/src/config/mod.rs:873`) is an independent field from the execution phase; the plan phase can separately be configured to a higher level.

### How CC does it

**Levels vary by model** (model-config page):

- Opus 4.8 / Opus 4.7: `low` / `medium` / `high` / `xhigh` / `max`
- Opus 4.6 / Sonnet 4.6: `low` / `medium` / `high` / `max`
- Defaults: Opus 4.8, Opus 4.6, Sonnet 4.6 all default to `high`; Opus 4.7 defaults to `xhigh`
- Setting a level the current model does not support falls back to the highest supported level not exceeding that level (e.g., setting `xhigh` on Opus 4.6 actually runs `high`)

**Persistence differences**: `low`/`medium`/`high`/`xhigh` persist across sessions; `max` is current-session-only (unless set via the `CLAUDE_CODE_EFFORT_LEVEL` environment variable). `max` is the deepest reasoning level with no token cap; the official documentation notes it is "prone to overthinking, test before adopting broadly".

**Two additional things that are not model levels — they are CC's own additions:**

`ultracode` deserves extra attention, because it is often treated as "an extra reasoning level for Opus 4.8". It is not. It is a session-level CC setting: when enabled, it sends `xhigh` to the model and simultaneously has Claude automatically orchestrate a **dynamic workflow** for each substantive task — a JavaScript orchestration script written by Claude in real time, run in the background, breaking the work into dozens to hundreds of sub-agents running in parallel (dynamic workflows introduced in v2.1.154).

The key difference from ordinary subagent orchestration: the script holds its own loops, branches, and intermediate results; only the final answer stays in Claude's context. This makes it suitable for tasks that a single conversation context cannot coordinate — full-repo bug scans, migrations across hundreds of files, multi-source cross-validated research. Scripts can also encode quality routines, for example having multiple agents adversarially review each other's conclusions before reporting.

Two ways to trigger: state the request directly in the prompt (in your own words, or with the keyword `ultracode`), or `/effort ultracode` to have Claude auto-plan a full orchestration for every substantive task in the session. It only appears in the `/effort` menu for models that support `xhigh` (i.e. Opus 4.8 / 4.7); other models do not offer it. Current session only — it is not part of the `effortLevel` setting, the `--effort` flag, or `CLAUDE_CODE_EFFORT_LEVEL`.

`ultrathink` keyword: write `ultrathink` anywhere in the prompt to request deeper reasoning for that turn. It does not change the session effort level; the effort value sent to the API is unchanged — it only adds an in-context instruction for that turn. "think" / "think hard" and similar phrases are not recognized as keywords.

**Entry points**: `/effort` (no argument opens a slider / `/effort <level>` sets directly / `/effort auto` reverts to model default); left/right keys on the slider in `/model`; the `--effort` flag; the `CLAUDE_CODE_EFFORT_LEVEL` environment variable; `effortLevel` settings (accepts only low~xhigh; `max` and `ultracode` are not accepted); the `effort` field in skill / subagent frontmatter. Priority: environment variable > settings > model default.

### Comparison

| Dimension | Codex | CC |
|-----------|-------|----|
| Number of levels | 6 (None/Minimal/Low/Medium/High/XHigh) | By model, up to 5 (low/medium/high/xhigh/max) |
| Default level | `Medium` | Most models `high` (Opus 4.7 is `xhigh`) |
| Client-side processing | No processing (pure pass-through, changes only one API field) | Additional wrapping (`ultracode` triggers background workflow orchestration) |
| Single-turn depth boost | No dedicated keyword | `ultrathink` keyword (does not change session level) |
| max persistence | N/A | Current session only (when not set via environment variable) |
| Level mapping across model switches | `nearest_effort`: minimum absolute rank (may map higher) | Takes the highest supported level not exceeding the target (only down, never up) |
| Plan phase independent effort | Yes (`plan_mode_reasoning_effort` independent field) | No dedicated field |

Both sides share the low/medium/high/xhigh naming in the middle range, both have model defaults and fallback mechanisms. The difference: Codex treats effort as a field that is only sent to the API without local processing — verifiable in source; CC adds a few things on top of the same levels — upward with `max` (uncapped), `ultracode` (effort plus background workflow orchestration), and `ultrathink` (single-turn depth boost without changing the session level).

The fallback algorithms also differ: Codex `nearest_effort` maps by minimum absolute rank difference and may land higher than the target; CC takes the highest supported level not exceeding the target — only down, never up.

### Practical advice

**Codex**: `Medium` (default) fits most scenarios; dial up to `High` / `XHigh` when deep reasoning is needed; configure a higher level separately for plan mode (planning is worth thinking harder about; the execution phase does not need the same level); remember that changing the level only changes one string sent to the server — the client does not "try harder".

**CC**: `high` (Opus 4.8 default) is sufficient for day-to-day use; for hard problems, write `ultrathink` in the prompt for a temporary boost without touching session settings; use `ultracode` to have Claude break tasks into parallel orchestration automatically; use `max` with care — potential for overthinking, test on a small scope first.

**Universal**: effort is calibrated per model; the same level name does not represent the same compute on different models. After switching models, confirm the level actually in effect.

---

## Summary

Across both sections, each side's control surface can be summarized in one sentence:

**Codex**: permissions are a "policy matrix" (two deterministic, orthogonal dimensions, open-source verifiable); effort is "levels passed through unchanged" (whatever level you set is exactly what gets sent, no local processing). Predictability first, behavior inspectable.

**CC**: permissions are "model-based control + policy fallback" (auto mode classifier for semantic judgment, deny rules as the hard safety net); effort is "levels plus extras" (max uncapped, ultracode with background orchestration, ultrathink for temporary single-turn depth). Flexibility first, control surface made thick.

Neither approach is right or wrong — each fits different scenarios. When you need strongly predictable behavior and machine-verifiable security boundaries, Codex's combination matrix is more direct. When you want to reduce manual intervention and rely on semantic judgment for complex edge cases, CC's classifier path is less work to maintain.

---

## Harness Engineering Series

A step-by-step breakdown of the platform layer (harness) and business engineering for coding agents:

1. [What Is a Harness, Exactly](./01-what-is-harness.md) — the boundary between platform layer and business engineering
2. [How to Write Specs for Complex Tasks](./02-how-to-write-specs.md) — multi-agent, orchestrator entry, rules / docs / skills organization
3. [How to Extend the Harness](./03-extending-the-harness.md) — skills, config directories, and hooks: the two mechanisms in CC and Codex
4. [How the Harness Controls the Agent](./04-permissions-and-effort.md) — permissions and effort: the control surface in CC and Codex
5. How the Harness Handles Long Tasks — compaction, memory, goal (in progress)

> Other languages: [English](../en/04-permissions-and-effort.md) · [한국어](../ko/04-permissions-and-effort.md) · [日本語](../ja/04-permissions-and-effort.md) · [中文](../zh/04-permissions-and-effort.md)
