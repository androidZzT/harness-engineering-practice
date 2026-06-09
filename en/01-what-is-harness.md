# What Is a Harness, Exactly

## 1. Defining the Harness

### A few authoritative definitions from the field

The term "Harness Engineering" went mainstream in 2026, but "harness" itself has been used with increasing looseness. Let me lay out the primary sources first, then give my own reading.

**OpenAI**, in [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/) (2026-02, Ryan Lopopolo), defines the harness as:

> "the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."

In OpenAI's framing, the Codex harness is specifically `codex-core`, the shared Rust library — the agent loop, thread lifecycle, config/auth, and sandboxed tool execution all live there. The UI shells (VS Code extension, CLI) are not part of the harness.

**Anthropic**, in [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk), writes:

> "the agent harness that powers Claude Code (the Claude Code SDK) can power many other types of agents, too."

The November 2025 piece [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) treats the harness as a standalone engineering object: context compaction, structured handoff, and tool sandboxing are all listed as key harness design concerns.

**Simon Willison**, in [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/), uses the term most broadly:

> "A coding agent is a piece of software that acts as a harness for an LLM, extending that LLM with additional capabilities that are powered by invisible prompts and implemented as callable tools."

He counts the entire Claude Code, Cursor, and Codex CLI — UI shells included — as the harness.

**LangChain**, in [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness), proposes the formula:

> "Agent = Model + Harness."

The model is the engine; the harness is the vehicle that wraps the engine so it can do sustained work.

The **overlap** across all these definitions is clear: the harness is the runtime layer outside the model — scaffolding, constraints, feedback loops, tool calls, context and memory management, agent loop. The **difference** is scope: OpenAI draws it tightest (SDK/core only), Simon draws it broadest (includes UI surface), Anthropic sits in between.

### My reading

This series uses the narrower, more OpenAI-adjacent definition, and settles on this one-sentence version:

> **The harness is the platform layer of a coding agent — context management, memory, subagent orchestration, the skill mechanism, tool calls, hooks, and the run loop — that layer. Business engineering is built on top of it and should not modify it.**

Why the narrow reading? Because the boundary question this piece is about — "what is the harness, what is business engineering" — only becomes meaningful under a narrow definition. If the entire CLI counts as the harness, the business engineering boundary falls outside the CLI and there is nothing worth discussing. Conversely, if the harness is just the LLM call interface, then context, memory, and tools all become the business layer's problem, which effectively hands the entire engineering discipline down to every individual project team — also unrealistic. The middle ground: **the SDK/core layer of the coding agent**.

### SDD is not Harness Engineering; a spec is not a harness

These two pairs get conflated often, but they sit at entirely different levels.

**SDD (Spec-Driven Development)** was popularized by OpenAI's Sean Grove in [The New Code](https://www.darekm101.com/articles/the-new-code-sean-grove-openai) and related talks. It focuses on the **engineering paradigm at the business layer**: use specs rather than code directly as the primary artifact of human–AI collaboration — requirements spec, design spec, acceptance spec in sequence, with code as a derivative of the spec. SDD is about **what to build**.

**Harness Engineering** was formally named by OpenAI's Ryan Lopopolo in the February 2026 post. It focuses on the **engineering paradigm at the agent platform layer**: how to design context compaction, how to manage the agent loop, how to sandbox, how to stay stable across long sessions. Harness Engineering is about **how the agent runs**.

The relationship between them: **SDD is the higher-level paradigm, Harness Engineering is the lower-level one**. SDD asks "what spec does the business need to write for delivery"; Harness Engineering asks "how does the coding agent execute those specs reliably". Without a harness, specs cannot run; without specs, a harness just spins in place.

**Spec and harness are also two different things.** A spec is a product of business engineering — workflow backbone, phase contracts, atomic tools, domain knowledge, all the things a business team writes itself, in various forms of spec. The harness is the platform those specs run on. The relationship is "business specs call harness capabilities" — **not "specs are part of the harness", and not "the harness is a kind of spec"**.

Conflate these two layers and the question "what belongs to the business, what belongs to the platform" becomes permanently unanswerable.

## 2. The Harness Is the Coding Agent Platform Layer

I currently work with Claude Code and Codex. From an engineering perspective the two are not fundamentally different: a local CLI entry point, a remote model, and a runtime mechanism that lets the model complete tasks in a controlled way. That runtime mechanism is the harness.

The seven items below are my own synthesis, using Claude Code / Claude Agent SDK as the primary reference. "Skill" and "hook" are Claude ecosystem terminology; the Codex counterparts are "permissions / sandbox / agent loop hooks" — same concept, different names. These seven are not an official taxonomy. They come from actual orchestration experience and cover the vast majority of agent behaviors you will encounter in practice.

**1. Context window management.** Decides what information enters the current conversation window, when to compact, when to truncate, when to restore historical state. When a run spans multiple phases, the harness moves stale details out and keeps the facts the current phase needs. The model sees only the current window; all the tradeoffs behind it are invisible to it.

**2. Memory management.** Persistent information across runs: project rules, user preferences, auditable evidence. `CLAUDE.md`, `AGENTS.md`, `.inbox`, run state — these are all memory infrastructure. The harness decides when to inject them into the window; business engineering decides what goes inside them.

**3. Spawning subagents.** The entry agent does not do the concrete work itself. It breaks tasks into subagents, each with an isolated context, and its own job is orchestration, evidence convergence, and closing out. This is critical: subagents are a primitive provided by the harness. Business engineering can only "dispatch" them — it should not reinvent the dispatch mechanism.

**4. The skill mechanism.** Claude Code skills, Codex plugins/presets — they are fundamentally the same thing: a loadable behavior unit with a trigger condition, input constraints, an output contract, and a risk label. The harness decides how a skill is discovered, loaded, and composed; business engineering decides what a skill contains.

**5. Tool calls.** `git`, `worktree`, shell, build commands, test commands, MCP server, third-party SaaS — all side effects happen through tool calls. The harness owns the protocol layer (registration, scheduling, timeout handling, output truncation); business engineering owns "which tools should be registered".

**6. The hook mechanism.** `SessionStart`, `Stop`, `PreCompact`, `PostCompact` — these hooks are the harness's "cut in at a lifecycle checkpoint" interface for business use. Injecting `.inbox`, running a workflow closure check before Stop, preserving key evidence before compaction — all of this is done through hooks.

**7. The run loop.** This is the meta-capability that ties the previous six together: read state → assess phase → dispatch subagent → converge evidence → sync state → decide continue or block. A mature harness guarantees this loop always converges; it never silently walks away mid-task.

What all seven share: **they are things the coding agent team (Anthropic / OpenAI) is continuously refining. They are not things you should rewrite for your project.** Your project can decide how to *use* these capabilities, but should not bypass them and implement its own copy.

## 3. What Business Engineering Is: a Set of Specs Built on Top of the Harness

Back to the thread from the first section: the output of business engineering is, at its core, a set of specs — executable, agent-dispatchable engineering contracts. Specs run because of the harness; what the harness runs depends on what the specs say.

How this set of specs is layered, organized, and laid out in the repository is worth a dedicated post — I won't expand here. This section just wants to state abstractly: what a complete business spec typically needs to cover, or in other words, what questions it needs to answer.

**Where does this delivery start, what phases does it pass through, and when is it done?** That is the workflow backbone. It does not "think" — it only declares: which phase we are in, who to dispatch next, what condition causes it to block. A project typically needs only one backbone spec. It is the entry orchestrator, but it is not itself an agent — the actual "thinking" is done by subagents the harness provides.

**What does each phase do, what does it produce, and when does it gate through?** That is the phase contract. Each phase has explicit inputs, outputs, gate conditions, and blocking rules. Phases hand off to each other declaratively without running ahead — this is the biggest engineering-discipline difference between SDD and traditional agent loops. How many phases to cut and how to cut them varies by project. "Requirements / Design / Implementation / Review / Test / Release" is a common skeleton but not the only combination.

**What atomic capabilities does a phase use when doing its work?** That is the toolbox: repo scanning, evidence extraction, build commands, E2E test runs, visual diff, task system integration, and so on. Each atomic capability is only responsible for its own inputs and outputs; **it should have no knowledge of which phase is calling it** — that is what makes it reusable across phases without being hard-coupled to any one.

**What background does a spec rely on when making judgments?** That is domain knowledge: business architecture, known pitfalls, reference implementations. This category is not directly invoked, but every other spec implicitly depends on it to make judgments like "is this good design", "does this step on a known landmine", "what does the reference implementation look like".

These four categories together define "what business engineering needs to write for a project". They share a few common traits:

- **All are written by the business itself** — they are part of the project repository and evolve with the project.
- **All run through primitives exposed by the harness** — the business side does not implement its own agent loop, context management, or subagent scheduling; all of those go through the harness.
- **They are organized through reference relationships** — the backbone references phases, phases reference tools, all of them reference domain knowledge.

At this point it becomes clear: **business engineering = writing this set of specs well; harness = the runtime that makes these specs run.** The relationship between them is "caller and callee", not "modifier and modified".

## 4. Why You Should Not Modify the Harness

I have seen projects where the first reaction to a problem is "can I tweak Claude Code's such-and-such mechanism": take over context compaction themselves, build a custom subagent scheduling layer, bypass hooks with an external injection shim. These ideas all come from a reasonable engineering instinct: I have more specific business requirements; the platform defaults aren't good enough.

But working around the harness is almost always not worth it, for three reasons.

**First, the harness itself is continuously iterating.** Claude Code has changed a lot in the past year — context compaction strategy, skill graph, subagent isolation, hook invocation timing, tool result trimming — and you get all of that for free, as long as you haven't patched over it. Once you have, every upgrade requires re-evaluating compatibility; a three-second upgrade becomes three hours of debugging.

**Second, harness complexity is much deeper than it looks.** Context management appears simple; in practice it involves window budgets, cache hit rates, compaction algorithms, information fidelity, and cross-run consistency. "I'll write a simplified version" sounds like 200 lines of code; in practice you discover one corner case after another. This is the eternal asymmetry between platform and application layers — what you see is the platform's default behavior; what you do not see is the several hundred edge cases it has already handled.

I ran into a concrete example of this recently. Someone in their own project built a "memory mechanism": a `memory/` directory in the repository, with a hook at session startup that bulk-injects all files from `memory/` into the context. On the surface it looks like "a self-built memory system". After a few runs the context window blows up, the run gets truncated mid-way, and important information gets pushed out.

This is the canonical "messing with the harness".

Memory management is one of the seven harness capabilities to begin with. The Anthropic piece [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) spends the whole post on how they approach it: structured progress file handoffs, compaction strategy, preserving necessary information while discarding redundancy — all oriented toward not blowing the window. Claude Code also provides primitives like `CLAUDE.md`, `.inbox`, and run state, with deliberate injection tradeoffs: what enters the window, what gets compacted, what gets truncated, what goes to external storage for lazy loading — all governed by rules.

Bulk-injecting `memory/` skips all of those tradeoffs. It effectively takes a capability a platform team has repeatedly refined and reduces it to "regardless of your window size, stuff it all in" via a single hook line. The result is that the window budget problem the harness had already handled for you gets dragged back onto your plate, without the platform's compaction and recovery mechanisms as a safety net.

Doing the same thing with harness primitives — writing project rules to `CLAUDE.md`, using hooks to inject `.inbox` at the right moment, using run state to sync necessary facts across phases — costs a few lines of config. Reimplementing it yourself costs a blown context window, lost platform upgrade benefits, and ongoing maintenance burden. Leave professional concerns to the professionals; business engineering should not modify harness mechanisms — this is not a lecture, it is engineering economics.

**Third, bypassing the harness means you are off the SDD track.** SDD's engineering discipline comes from the hard constraints the harness exposes: subagents cannot leak information across contexts, phases cannot gate-skip, tool calls must leave evidence. Once you bypass the harness for some business requirement, those constraints all collapse together. Short-term it looks like working around a limitation; long-term you have demolished the foundation of the entire engineering discipline.

The right approach: **when the platform layer is insufficient, first ask whether the business spec layer can absorb it.**

One positive example from my own practice. When I was building a harness-based workflow, I ran into this: the model would sometimes "want to take one more step" across phases — it was only supposed to produce requirements evidence, but it would start doing design work. My first thought was "can I patch the subagent scheduling to force it to stay in the current phase". I then realized this is actually solvable on the business spec side: lock down the current phase's boundaries tightly in the phase contract, add a gate check, and use a hook to force convergence when the subagent should close out. The harness was untouched, and the behavior was constrained.

This ability to "absorb platform limitations in business engineering" is the core muscle an SDD engineer should be training.

## 5. One Sentence to Remember

> **The job of business engineering = write this set of specs well, compose the primitives the harness exposes; do not modify the harness itself.**

That sentence has two directions.

Looking up: use everything the harness gives you. Context management, subagents, skills, hooks, tool calls — these are free "engineering capability add-ons". The platform team understands how to implement them better than you do.

Looking down: the differentiation in your business delivery lives in how the specs are written and composed, not in how deeply the harness is modified. Two teams doing similar projects will have different specs — the difference is in each team's phase segmentation, atomic tool choices, and accumulated domain knowledge — not in one side having hacked the Claude Code internals.

This is the strongest intuition I've developed doing Harness Engineering: **restraint is the core of engineering discipline.** Not touching what should not be touched frees up all that energy for the work that actually matters: writing the business specs well, and writing them precisely.

The next post will expand on two related questions: **what exactly should a spec contain, and how should skills be layered?** Bringing workflow backbone, phase contracts, atomic tools, and domain knowledge — where each goes in a concrete project structure, how they reference each other — down to an actionable layout.

---

## Harness Engineering Series

A step-by-step breakdown of the platform layer (harness) and business engineering for coding agents:

1. [What Is a Harness, Exactly](./01-what-is-harness.md) — the boundary between platform layer and business engineering
2. [How to Write Specs for Complex Tasks](./02-how-to-write-specs.md) — multi-agent, orchestrator entry, rules / docs / skills organization
3. [How to Extend the Harness](./03-extending-the-harness.md) — skills, config directories, and hooks: the two mechanisms in CC and Codex
4. [How the Harness Controls the Agent](./04-permissions-and-effort.md) — permissions and effort: the control surface in CC and Codex
5. How the Harness Handles Long Tasks — compaction, memory, goal (in progress)

> Other languages: [English](../en/01-what-is-harness.md) · [한국어](../ko/01-what-is-harness.md) · [日本語](../ja/01-what-is-harness.md) · [中文](../zh/01-what-is-harness.md)
