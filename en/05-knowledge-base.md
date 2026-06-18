# Spec and knowledge base: making sure the agent actually reads it, actually obeys it

You wrote a CLAUDE.md for your project, split out a few rules, built a couple of skills, wired up two MCP servers. And then the agent makes the same mistakes it always did, and treats the rules you set like they aren't there.

At that point you start to suspect one thing: did it even read them?

That question is worth more than it looks. Between "I configured a file" and "the content of that file actually shaped the model's decision at this step" sit several gates, and each one can quietly drop the ball. I split it into four things that usually get treated as one:

> **Linked ≠ Loaded ≠ Read ≠ Obeyed.**

Putting a file in `.claude/` is *linking*. Having it spliced into the context window for this session is *loading*. The model actually factoring it into its judgment at the current step is *reading*. Acting on it is *obeying*. Break any one of the four links and the result is the same — "the rule didn't take effect" — but where it broke determines how you debug it.

This chapter first walks through the ways to wire up a knowledge base, then covers when and in what form each one enters the context, and finally how to confirm it actually got in and actually constrained the agent. As before, Claude Code (CC below) and Codex side by side — the CC side per official docs and stated conservatively, the Codex side cited down to source line numbers wherever I could.

## 1. The ways a knowledge base reaches the agent

First, what "knowledge base" means here: the things you want the agent to keep obeying — project conventions, coding standards, workflows, domain knowledge, callable capabilities, external data. To wire these into the agent, CC and Codex each have a set of mechanisms that mostly map one-to-one:

| Mechanism | Claude Code | Codex |
|---|---|---|
| Main instruction file | CLAUDE.md: project / user / org scopes, plus CLAUDE.md in subdirectories | AGENTS.md: global / project / subdirectory, multi-level |
| Split-out rules | `.claude/rules/`: split across files, can be scoped by file path | instructions in config: directives that travel with the config |
| Auto memory | Auto memory: notes the model accumulates itself, entry point `MEMORY.md` | —— |
| Skill | Skill: a loadable unit of behavior with trigger conditions | skill: same concept, slightly different naming |
| External tools / data | MCP | MCP |
| Special injection points | `@import`, `--append-system-prompt` | —— |

These paths look similar; the differences that matter are all in the next section. *When* and *how* they enter the context differs, and that is exactly the root of "I configured it but it didn't take."

## 2. How each path enters the context

The difference hides in three things: when it enters the context, in what role it enters, and where it quietly leaks. Here is the skeleton in one table, with source citations below it:

| Path | When it enters the context | In what role / form | Easiest place to trip |
|---|---|---|---|
| CLAUDE.md / AGENTS.md | At startup, the chain from "launch directory upward" loads; subdirectory ones only enter once the agent actually goes and reads files in that directory | Injected as a user message, not a system prompt — it is context, not enforced configuration | A subdirectory rule you didn't touch this run is simply absent; AGENTS.md defaults to a 32 KiB cap, over which it is truncated (not ignored — just didn't get in) |
| `.claude/rules/` | Ones with `paths:` trigger by glob path; only ones without `paths` load in full at startup | Same as CLAUDE.md, lands on the user side | You wrote `paths` but didn't touch the matching directory → the rule doesn't fire; that's the design, not a bug |
| skill | At startup only metadata enters (name + `description`); the body loads only once triggered | progressive disclosure: costs almost no tokens unused, but once loaded it stays resident every following turn | `description` didn't match → not one word of the body enters; "discovered / loaded / triggered" are three separate things |
| MCP | Tool definitions (schemas) enter the context at startup | a token-costing tool list | Mount many servers, each with many tools → you pay a big context "rent" up front |
| permission / hook | Intercepts right before a tool actually executes, **bypassing the model's intent** | a client-side hard constraint, outside the "enter the context" chain above | If you want "block it no matter what the model thinks," don't write it into CLAUDE.md — use a deny rule or a PreToolUse hook |

A few details worth remembering, with sources:

- **CLAUDE.md**: officially characterized as context, not enforced configuration; the "always / never" you write into it is fundamentally a request, not a switch. `@import` expands at startup, up to 4 hops; block-level HTML comments are stripped before injection, so you can use them to leave notes for humans at no token cost.
- **AGENTS.md**: multi-level discovery is in `core/src/agents_md.rs`; the 32 KiB cap is `AGENTS_MD_MAX_BYTES` (`core/src/config/mod.rs:186`); it is wrapped as user instructions tagged `# AGENTS.md instructions ...` / `</INSTRUCTIONS>` (`core/src/context/user_instructions.rs`), then merged with config instructions.
- **skill**: metadata in the list has a cap — `description` + `when_to_use` combined are truncated past 1536 characters; on the Codex side the metadata is rendered to a budget and spliced into the developer message (`build_available_skills` in `core/src/session/mod.rs`).
- **MCP**: tools are listed from the connection manager, filtered to the accessible-and-enabled ones, then spliced into the instructions (`core/src/connectors.rs`).
- **permission**: CC has three arrays `allow` / `ask` / `deny`, resolved deny > ask > allow, then layered with a permission mode (default / acceptEdits / plan / bypassPermissions); Codex bakes it into the type system — `SandboxPolicy` with four values (`protocol/src/protocol.rs:878`), `AskForApproval` with five (`:784`), and a fine-grained `GranularApprovalConfig` (`:817-854`) that can separately govern whether skill / rules / MCP need per-use confirmation.

To close this section in one line: **the words in CLAUDE.md / a rule / a skill are suggestions for the model to see; permissions and hooks are gates that block actions on your behalf. If you want "definitely," use the latter.**

## 3. How to confirm the agent actually read it and is actually constrained

This is the question this chapter most needs to answer. Take that "Linked ≠ Loaded ≠ Read ≠ Obeyed" chain apart, and the verification splits into two layers too: one looks at whether something entered the context window (static), the other at what the agent actually did (behavioral).

### First: did the thing actually enter the context

CC has a few ready-made windows.

- **`/context`**: shows the token breakdown of the current context window — how much goes to the system prompt, tools, MCP, memory files, messages. The most direct test: if something you thought was loaded has no share here, it didn't get in.
- **`/memory`**: lists which CLAUDE.md, CLAUDE.local.md, and rules files this session actually loaded. A file not on this list is one the model simply can't see. Debugging "CLAUDE.md isn't working" starts with running this to confirm the file is on the list.
- **`InstructionsLoaded` hook**: for precise logs, this hook records exactly which instruction files were loaded, when, and why — especially handy for debugging path-scoped rules and those lazily loaded subdirectory files.

A few more checks that follow directly from the mechanisms above: subdirectory CLAUDE.md, `MEMORY.md` content past line 200, and topic files do not enter the context at startup; to confirm they take effect, watch whether the model actually goes and reads the files in the matching directory. After `/compact`, the project-root CLAUDE.md is re-read from disk and reinjected, but subdirectory ones are not — so rules "loosening" after compaction is often exactly this.

### Then: what the agent actually did

A static check tells you "whether the thing is in the window," but can't answer "did that skill of yours actually trigger this time" or "did the sub-agent you dispatched inherit your rules." Those are behavioral questions, and they need the execution trace of a real run.

For this layer you can use an open-source profiler, for example [cctrace](https://github.com/androidZzT/cctrace) (written in Go, MIT, runs locally, no network). It wraps your agent (`cctrace claude -- claude` or `cctrace codex -- codex`) and lays the run's events out as a local waterfall timeline. Mapped to this chapter's questions, a few of its features line up exactly:

- At session startup it collects "which skills were discovered on this machine, and their descriptions," and then marks on the timeline "which skill actually triggered this time." Compare the two and you can judge: if a skill is in the list but never fired all session, its `description` didn't route the model to it — the problem is in the description, not the body.
- It marks **sub-agent fan-out** separately. This matters: a sub-agent runs in its own isolated context and does not automatically inherit the CLAUDE.md from your main session. Many "I clearly wrote the rule but the sub-task ignored it" cases are exactly because the work was done by a sub-agent that never read that rule. Seeing the fan-out on the timeline is what makes you realize the rule needs to sink down to where the sub-agent can reach it.
- It puts tool calls, model thinking, stalls, and retries on the same time axis, so "what exactly this step called and where the time went" is plain to see.

In short, the built-in `/context` and `/memory` answer "what is in the context," and the profiler answers "what the agent actually did." Confirmation means both sides line up.

### On the Codex side: dump what was sent to the model, verbatim

Codex offers a more thorough path. Set the environment variable `CODEX_ROLLOUT_TRACE_ROOT` (`rollout-trace/src/thread.rs`) and it writes the raw events and payloads of every inference — request, response, tool calls — to disk as a local bundle, which an offline reducer later reconstructs into "the exact string of conversation items the model actually saw." This is the hardest ground truth: not inferring whether it read something, but storing exactly what was sent to the model and checking it item by item. Session-level rollout persistence lives around `core/src/session/mod.rs`.

### If you want "guaranteed to take effect," there are only two roads

After the whole loop, the conclusion is short: text in the context is not guaranteed to be strictly enforced. If you truly need a rule to "definitely" take effect, there are only two roads: lift it to the system-prompt level (CC's `--append-system-prompt`, or org-level managed config), or skip the "persuade the model" route entirely and block the action at the client with hooks and permission rules. The former raises the probability of being obeyed; the latter gives the model no chance to choose at all.

## Summary

> **Configured doesn't mean loaded; loaded doesn't mean read; read doesn't mean it will obey.**

Each way of wiring up a knowledge base gets stuck at a different point on this chain: CLAUDE.md and rules get stuck on "when, and in what role, they enter the context"; skills on "did the description route to it"; MCP on "is it worth the context it costs"; permissions step out of the chain entirely and block at the client.

When debugging, first use `/context`, `/memory`, and `InstructionsLoaded` to confirm the thing got into the window, then use the execution trace of a real run to confirm what the agent actually did — whether the skill triggered, whether the sub-agent inherited the rules. For certainty, don't bet on the text in the context being strictly enforced; lift it to the system prompt, or hand it to hooks and permissions.

The next chapter returns to the long-task thread: how the harness survives a session spanning many turns — how compaction, memory, and goal work together.
