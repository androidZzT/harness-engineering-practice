# How to Extend the Harness: Skills, Config Directories, and Hooks

The previous post "What Is a Harness, Exactly" drew the harness boundary clearly — it is the platform layer, responsible for context management, memory, subagent orchestration, tool calls, hooks, and the run loop; business engineering is built on top of it and should not modify it. This post goes one level deeper: how does the harness itself get extended?

Three specific things: what skills should contain, what goes in config directories, and how to attach hooks. Both CC and Codex provide all three. Many people assume some capability is unique to one side; in practice both have counterparts for everything. The real difference is not "who has what" — it is **that given the same extension problem, the two took different engineering paths**.

---

## 1. Skills — What to Write, How Loading and Triggering Work

### What problem this solves

An LLM's default behavior is general-purpose. But in a real project you accumulate a lot of reusable workflows: domain-specific code review standards, best practices for a particular framework, a fixed commit message format, a checklist to go through every time.

There are three common ways to handle this: type it in each time (repetitive work), add it to the system prompt (global pollution — every conversation carries it), or wrap it in a shell script (disconnected from the agent context). Skills are a fourth option: package a reusable workflow, resource, or body of expertise into a self-contained unit that loads automatically or on demand at the right moment, without cluttering the conversation the rest of the time.

### How Codex does it

A Codex skill is a self-contained folder providing specialized workflows, tool integrations, domain expertise, and bundled resources (from `codex-rs/skills/src/assets/samples/skill-creator/SKILL.md`).

Each skill's entry point is `SKILL.md`, with three frontmatter fields:

```yaml
name: my-skill              # required; also the identifier used when calling it
description: |              # required; the content is "when to use this skill"
  This skill should be used when...
metadata:
  short-description: one-line summary  # optional; used in display lists
```

How you write the `description` field determines trigger quality. Its semantics are "under what conditions should you call me" — not "what I can do". The former is a trigger condition; the latter is a capability description. The agent uses the former to match scenarios; write the wrong one and trigger rate will be low.

Built-in system skills are compiled into the binary with the `include_dir!` macro (`skills/src/lib.rs:10`), and installed to `$CODEX_HOME/skills/.system` at startup (`:24`). To avoid reinstalling on every startup, a fingerprint marker file is used — if the hash of the embedded content has not changed, installation is skipped (`:32`). This is the distribution stability guarantee, keeping system skill behavior consistent across upgrades.

User-defined skills go in `$CODEX_HOME/skills/` with no additional registration required; they take effect after a restart.

### How CC does it

CC skills live at `.claude/skills/<name>/SKILL.md`, with three frontmatter fields: `name`, `description`, and `disallowed-tools`.

Triggering has two modes: explicit invocation by the user (`/skill-name`), or automatic invocation by Claude based on conversation context. The second mode depends entirely on the quality of the `description` — write it clearly and automatic triggering becomes accurate.

Beyond local skills, CC has a plugin skill mechanism: install through the plugin marketplace, manage with `/plugin`. Within a session you can use `/reload-skills` to rescan (supported since v2.1.152), or trigger a rescan via the `reloadSkills` parameter in a `SessionStart` hook.

`disallowed-tools` is a CC-only field: it lets you disable specific tools at the skill level. A "code review" skill that only needs to read files, not write them, can disable the write-file tool in `disallowed-tools` — during that skill's execution, write operations will not be triggered, reducing the risk of unintended changes.

### Comparison

| Dimension | Codex | CC |
|-----------|-------|----|
| Storage location | `$CODEX_HOME/skills/` | `.claude/skills/<name>/` |
| Frontmatter fields | name / description / metadata.short-description | name / description / disallowed-tools |
| Trigger modes | Context-based auto-trigger | Explicit `/skill-name` + context-based auto-trigger |
| Tool-level restriction | No dedicated field | `disallowed-tools` |
| Built-in skills | Compiled into binary, fingerprint-based reinstall optimization | Plugin marketplace |
| Hot reload | Updates with filesystem | `/reload-skills` or `SessionStart` hook |

### Practical advice

The most common mistake when writing a skill is writing the `description` as a capability description rather than an invocation condition. A bad example:

> Capability description style: "This skill provides Python code review capabilities, including style checking and performance analysis."

Trigger rate will be low, because the agent sees "I have this capability" but does not know when to call it.

Rewritten as a condition-trigger description:

> "This skill should be used when reviewing Python code, checking for style issues, or analyzing performance bottlenecks in Python files."

A few more points:
- Use CC's `disallowed-tools` properly: give each skill an explicit tool scope to prevent skills from interfering with each other
- Built-in system skills carry a maintenance burden; Codex compiles them into the binary for distribution consistency, but for day-to-day use just put skills in the user directory
- Commit local skills to the repository and share them with the team — a skill is a reusable workflow asset; it should not live only on one person's machine

---

## 2. Config Directories — .codex vs .claude, What Goes Where

### What problem this solves

Agent behavior needs configuration: which model to use, which tools to allow, project-level code convention constraints, enterprise-level security boundaries. There is a natural tension here: personal preferences, project constraints, and enterprise policy all have different ownership and should not all go in a single file where "later entries override earlier ones" — an enterprise security boundary should not be overridable by a project config file.

The core design question for config directories: **whose config can override whose, and where is something locked down**.

### How Codex does it

Codex's project-level config is `.codex/config.toml` in TOML format. The layered loading order is fully commented in `config/src/loader/mod.rs:79-100`, split into two separate logics:

**Constraint layer** (for enterprise/security policy):

```
cloud → admin → system (/etc/codex/requirements.toml)
```

This layer uses "early-layer lock" semantics: `a constraint defined in an earlier layer cannot be overridden by a later layer` (direct quote). A constraint that a system administrator writes in `/etc/codex/requirements.toml` cannot be overridden by the project layer or the user layer.

**Config layer** (ordinary config, later layers override earlier ones):

```
admin → system (/etc/codex/config.toml)
     → user ($CODEX_HOME/config.toml)
     → profile ($CODEX_HOME/<name>.config.toml)
     → cwd (./config.toml)
     → tree (./.codex/config.toml, searches upward)
     → repo (git root/.codex/config.toml)
     → runtime (--config flag, etc.)
```

Project root detection is determined by `project_root_markers`, defaulting to `.git`. Config files closer to the project root have higher priority (runtime is highest).

Protected paths are another hard constraint: `PROTECTED_METADATA_PATH_NAMES = [".git", ".agents", ".codex"]` (`protocol/src/permissions.rs:27`). Even in danger-full-access sandbox mode, agents cannot write to these three directories. This is a system-level protection, not an optional configuration.

`.agents/` holds subagent definitions; `.codex/hooks.toml` holds hook config (expanded in the next section).

### How CC does it

CC's config directory is `.claude/`, containing:

- `settings.json`: JSON format, agent behavior config
- `CLAUDE.md`: Markdown, system prompt and context notes
- `skills/`: local skill directory
- `agents/`: sub-agent definitions
- `commands/`: custom slash commands
- `rules/`: rule files loaded by path scope
- `.mcp.json`: MCP tool config

Priority from high to low: managed (enterprise IT push) > command-line arguments > local > project (`.claude/`) > user (`~/.claude/`). Managed settings are the enterprise-level mandatory policy — highest priority, cannot be overridden by any other layer.

`rules/` has a feature worth mentioning separately: it supports a `paths:` field in frontmatter for path scope. For example, `rules/python-style.md` with `paths: ["**/*.py"]` in its frontmatter means this rule is only added to context when Claude reads a Python file — it takes up no tokens at other times. In large repositories this can meaningfully reduce token consumption from irrelevant rules.

### Comparison

| Dimension | Codex | CC |
|-----------|-------|----|
| Project config file | `.codex/config.toml` (TOML) | `.claude/settings.json` (JSON) + `CLAUDE.md` (Markdown) |
| Enterprise mandatory policy | requirements.toml, constraint layer locked, later layers cannot override | managed settings, highest priority |
| Layering logic | Two independent logics: constraint layer (locked) + config layer (overridable) | Unified priority chain, managed at the top |
| Protected paths | `.git` / `.agents` / `.codex`, not writable even in sandbox | No equivalent mechanism |
| Path-scoped rules | None | `rules/` frontmatter `paths:` field |
| Subagent definitions | `.agents/` | `.claude/agents/` |

Both sides agree that enterprise policy should not be overridable, but the implementation differs: Codex writes the constraint layer and config layer merge logic separately, with explicit "early-layer lock" semantics in code; CC relies on the priority chain, with managed at the highest priority to enforce this.

### Practical advice

The core value of layering is **making configuration ownership clear**:

- Personal preferences (which model, what style) go in the user layer (`~/.codex/config.toml` or `~/.claude/settings.json`) — do not commit to the repository
- Project constraints (tool permission scope, code convention notes) go in the project layer, committed to the repository so team members share the same constraints
- Enterprise security boundaries (prohibit certain tool calls, network access restrictions) use the managed layer — do not try to override them with the project layer
- Make full use of CC's `rules/` path scope: instead of piling all rules into one large `CLAUDE.md`, split by language or directory so only relevant rules load each time

---

## 3. Hooks — Events, Types, Trust

### What problem this solves

During execution, an agent passes through many key checkpoints: before and after tool calls, session start and end, when the user submits a prompt, during context compaction, when a sub-agent starts or stops. Inserting custom logic at these checkpoints enables a lot:

- Confirm before writing a file
- Log an audit record after a tool call
- Run a completion check at session end
- Auto-decide when a permission request arrives

Both sides use an event-driven lifecycle model, but the mount configuration, handler types, and trust mechanisms differ.

### How Codex does it

Codex hook config lives in `.codex/hooks.toml` (TOML format).

Full event list (from `config/src/hook_config.rs`, 10 total):

```
PreToolUse         Before a tool call
PostToolUse        After a tool call
PermissionRequest  When a permission request arrives
PreCompact         Before context compaction
PostCompact        After context compaction
SessionStart       Session startup
UserPromptSubmit   User submits a prompt
SubagentStart      Sub-agent starts
SubagentStop       Sub-agent stops
Stop               Agent stops
```

Handler types (`HookHandlerConfig` enum, `hook_config.rs:137-156`):

```rust
enum HookHandlerConfig {
    command { command, commandWindows, timeout, async, statusMessage },
    prompt {},
    agent {},
}
```

Three types — `command` (shell command), `prompt` (LLM evaluation), `agent` (sub-agent verification).

The trust mechanism is implemented via the `trusted_hash: Option<String>` field in `HookStateToml` (`hook_config.rs:29`). Each hook must declare a hash in its `state` section; only when the hash matches the current file content will the hook run automatically. A mismatch requires user approval.

```toml
[state.my-audit-hook]
trusted_hash = "sha256:abc123"

[hooks.PostToolUse]
[[hooks.PostToolUse]]
matcher = "write_file"
[[hooks.PostToolUse.hooks]]
type = "command"
command = "echo 'file written' >> /tmp/audit.log"
```

The motivation behind `trusted_hash` is defense against malicious hooks: in a shared repository, if `.codex/hooks.toml` is tampered with, the hash invalidates and the hook does not execute silently — it triggers an approval step. The hook file itself is treated as a potential attack surface.

### How CC does it

CC hook config is in `settings.json`. The official documentation lists 32 events, including: `SessionStart` / `SessionEnd` / `UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `Stop` / `PermissionRequest` / `Notification` / `MessageDisplay`, and more.

Handler types: `command` (shell) / `http` (HTTP request) / `mcp_tool` (MCP tool) / `prompt` (LLM evaluation) / `agent` (sub-agent verification).

A few execution conventions worth knowing:
- The `matcher` field supports regex, matching tool names or event content
- Exit code 0 = success, 2 = block (signal Claude to stop the current operation)
- JSON is passed as context and results through stdin/stdout

The practical use of the `prompt` type: you can write a `Stop` hook with a prompt that asks a small model "was the task objective actually achieved" — if not, return a non-zero exit code and have Claude continue. This extends the hook's role from "listen + intercept" to "listen + judge + return control", effectively adding an automatic QA layer at the end of the agent loop.

The `http` type is CC-only: it calls an HTTP endpoint directly, without going through a shell script intermediary. Suitable for attaching hooks to external approval systems or logging platforms.

### Comparison

| Dimension | Codex | CC |
|-----------|-------|----|
| Config file | `.codex/hooks.toml` (TOML) | `settings.json` (JSON) |
| Total events | 10 | 32 |
| Handler types | command / prompt / agent | command / http / mcp_tool / prompt / agent |
| Unique events | SubagentStart / SubagentStop / PreCompact / PostCompact | SessionEnd / PostToolUseFailure / Notification / MessageDisplay / Setup / PermissionDenied, etc. |
| Trust mechanism | `trusted_hash`: auto-executes only if hash matches | Relies on filesystem permissions |
| HTTP type | None (must go through command indirectly) | Yes, calls HTTP endpoints directly |

Both sides have `prompt` and `agent` type hooks — this surprises many people, but the `HookHandlerConfig` enum in source explicitly has both variants. The main differences are: Codex has `trusted_hash` file-level trust (security focus); CC adds the HTTP type and more event nodes (integration focus).

### Practical advice

By usage frequency, the most common combinations:

- `PreToolUse` + `matcher` targeting dangerous tools + `command` type: insert a log line or confirmation before `rm -rf`, `git push --force`
- `PostToolUse` + `command`: write tool call records to an audit file for later review
- `Stop` + `prompt` type: have an LLM assess "was the task objective achieved" — if not, return exit code 2 to trigger Claude to continue
- Codex users should account for `trusted_hash` maintenance: every time you modify a hook script, the hash invalidates and you need to re-approve it. This is expected behavior, not a bug — but if you modify hooks frequently, make updating the hash part of your workflow

---

## Summary

Three extension capabilities, and each side's design direction can be summarized in one sentence:

**Codex**: lock down at build time (system skills compiled into the binary), verify at runtime (`trusted_hash`), constraint layer early-lock not overridable — security boundary first, push constraints earlier.

**CC**: filesystem-centric, path-scoped rules, plugin marketplace, HTTP hooks make extension points more flexible — composability first, push flexibility outward.

Neither side has capabilities that are truly exclusive. The real divide is the engineering attitude toward the same extension requirement: Codex tends to trust "the earlier a constraint is fixed, the safer it is"; CC tends to trust "the more composable, the more useful".

Which harness to use and which extension approach to take ultimately depends on your context — team size, security requirements, and depth of integration with external systems. Neither philosophy is right or wrong; it is a matter of fit.

---

## Harness Engineering Series

A step-by-step breakdown of the platform layer (harness) and business engineering for coding agents:

1. [What Is a Harness, Exactly](./01-what-is-harness.md) — the boundary between platform layer and business engineering
2. [How to Write Specs for Complex Tasks](./02-how-to-write-specs.md) — multi-agent, orchestrator entry, rules / docs / skills organization
3. [How to Extend the Harness](./03-extending-the-harness.md) — skills, config directories, and hooks: the two mechanisms in CC and Codex
4. [How the Harness Controls the Agent](./04-permissions-and-effort.md) — permissions and effort: the control surface in CC and Codex
5. How the Harness Handles Long Tasks — compaction, memory, goal (in progress)

> Other languages: [English](../en/03-extending-the-harness.md) · [한국어](../ko/03-extending-the-harness.md) · [日本語](../ja/03-extending-the-harness.md) · [中文](../zh/03-extending-the-harness.md)
