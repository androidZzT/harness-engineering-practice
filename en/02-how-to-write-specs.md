# How to Write Specs for Complex Tasks

The previous post drew a clear boundary between the harness and business engineering: the harness is the platform layer; business engineering is built on top of it and does not modify it. I left a question at the end: what exactly should the business-side specs contain?

This post answers that directly. For a simple task you can just throw a sentence at the model and be done. But when a task is complex enough to require multiple roles working together, spanning multiple phases, with each step auditable, a spec is no longer "a prompt" — it is a structured engineering asset. I'll break it into four questions: how multi-agent plays out in practice, what the orchestrator entry contains, how rules/docs/skills divide responsibilities, and how skills are layered.

## 1. Multi-Agent: One Orchestrator, a Team of Dedicated Executors

The first decision in a complex task is: do not let a single agent run the whole thing end to end.

The context window will blow up, responsibilities will blur, and an agent that does requirements analysis, then writes code, then does reviews will eventually lose track of which role it is playing and which rules it should be following. My approach is one orchestrator with a team of dedicated executors.

The orchestrator (entry agent) does only orchestration and closeout: assess what phase we are in, whether the gate has passed, who to dispatch, whether to continue. It does not write business code. The concrete work goes to dedicated sub-agents; each sub-agent starts in a fresh context with only the knowledge it needs for that run.

Executors are split by role. In my setup these are roughly:

- Requirements analyst: breaks down the requirements document and scope into structured inputs
- Architect: does technical design, defines cross-platform unified contracts, writes ADRs, but does not write implementation code
- Platform implementers: one per platform, each only writes for its own side
- Code reviewer: verifies from multiple dimensions — security, performance, standards, reference-implementation alignment
- Test engineer: adds test coverage, writes failing acceptance tests first
- Integration engineer: runs end-to-end builds, installs, runs, logs, cross-platform consistency checks
- Efficiency engineer: consolidates recurring patterns into reusable assets

There is one easily overlooked effect of role division: **picking the wrong role is not just a division-of-labor mistake — it also bypasses the constraint rules that role carries.** For example, platform implementers automatically load the hard constraint checklist for their respective platform, while the architect does not. If you mistakenly dispatch work that should go to an implementer to the architect, that entire implementation-layer check is skipped. I learned this the hard way: once I dispatched a front-end UI implementation task to the architect, bypassed the checks it was supposed to pass, and visual alignment issues resurfaced.

So dispatch cannot be intuition-driven. My rule is to select the role by phase first, then cross-verify the write scope against a "changed file path → role" mapping table. If the two conflict, stop and clarify — never silently expand a given agent's write scope. One more rule: cross-platform changes must never be handled by a single agent; the architect must first produce a unified contract, then each platform implementer works in isolation within its own write scope, in parallel.

Parallelism should also be controlled. I cap the active sub-agents on a single main thread at 5, with 4 as steady state and the 5th reserved. When dispatching each sub-agent, declare upfront: what artifact to deliver, in what format, written where, what the acceptance criteria are. After it finishes, verify that the artifact is present and meets spec before closing it and freeing the slot. Without this quota and artifact verification, multi-agent work quickly becomes a set of runaway concurrent processes.

## 2. The Orchestrator Entry (CLAUDE.md / AGENTS.md): What to Put In

The orchestrator's behavior is almost entirely determined by its entry file — `CLAUDE.md` or `AGENTS.md`. How well this file is written directly determines whether the whole collaboration is orderly or chaotic.

My first principle: **the entry file contains only startup routing and hard boundaries, not specific steps.**

Many people stuff all rules and all processes into `CLAUDE.md` until it reaches several thousand lines, requiring the model to read the whole thing at each startup — slow and hard to focus. The right approach is to keep the entry thin: it only tells the model "when a task arrives, first assess the type, then go read this file". Specific steps go into skills, hard rules go into rules, role contracts go into agents, background and decision records go into docs. The entry itself is just a map.

The second principle is to lock down the orchestrator's contract explicitly. My entry file has a section titled "Entry Agent Contract". The core is just a few lines:

- It is the orchestrator and closeout agent, not an executor
- It only assesses phases, gates, handoffs, whether to continue, and who to dispatch
- Small mechanical revisions, index updates, and running validators it can do itself; everything else defaults to dispatching sub-agents
- It does not directly modify production code; it does not bypass gates, reviews, or validators
- On non-fatal issues, first record owner, route, evidence, next step — do not pause and ask
- Only fatal errors, or a gate explicitly blocking the current action, warrant stopping

The last two points are especially important. The thing complex workflows fear most is an agent stopping to wait for your approval every time it hits a small snag — running for ten minutes and asking eight times. Write "what to record-and-continue vs. what warrants stopping" into the contract, and it can genuinely run an entire chain without interruption. Similarly, dispatching sub-agents is the orchestrator's internal operation — it should not come back to ask you "can I dispatch the next one" every single time. Only scope expansion, destructive operations, risk acceptance, and external publishing should come back to you for a decision.

One more pitfall worth calling out specifically: **do not directly edit the generated entry files.** My entry's true source of authority is a template file; `CLAUDE.md` and `AGENTS.md` are both projected from the template using a tool. Edit the projected file directly and it gets overwritten on the next sync. This is easy to stumble on in multi-agent setups — you think you changed a rule, but you changed an ephemeral artifact.

## 3. How to Organize rules / docs / skills

Spec is not just process — it also covers "where knowledge, constraints, and steps each live, and how they reference each other". I use three directory types to answer three different questions:

- skills: how to do something (specific steps and procedures)
- rules: what must be followed (hard constraints, checklists)
- docs: why things are the way they are (architecture, context, long-term decision records)

The core benefit of this split is that each piece of knowledge has exactly one authoritative location. A procedure changes: update the skill. A constraint changes: update the rule. Need to trace why a design decision was made: look in docs. No duplication, no contradictory copies scattered around.

### What a skill's file structure looks like

Each skill is a directory with a simple standard structure:

```
some-skill/
  SKILL.md       Entry: routing + responsibilities + boundaries + output contract
  references/    Long content loaded on demand: methodology, templates, check sections
  scripts/       Deterministic scripts (only in script-backed skills)
```

Test cases do not live in the skill directory; they go in a parallel `evals/` directory with one subdirectory per skill name (discussed in the next post).

`SKILL.md` itself has a fixed skeleton, with each section kept short:

- frontmatter: name, a description for routing (must include "applicable / not applicable / typical trigger phrases"), layer, risk level, whether manual review is required
- Scope of responsibility: what this skill manages
- Applicable and not applicable: when to invoke it, when not to
- How it works: minimal procedure, plus "at what point go read which reference"
- On-demand reference loading: a list mapping each scenario to its reference file
- Output and verification: what is delivered, what counts as passing

The key point: **SKILL.md must be thin.** It is a routing card for the model to quickly determine "should I use this, and how" — not a knowledge base. Dumping platform details, anti-patterns, and long templates into it means the model has to read an essay every time it is invoked, drowning out the routing signal.

### How a skill references rules, docs, and the knowledge base

The skill body is thin — so where does the actual long knowledge live, and how does it get used? Through three different referencing mechanisms, each handling a different type.

**References are the skill's own long content, loaded by name on demand.** I do not write methodology and templates into `SKILL.md` — I split them into separate files under `references/`, then write "at this step, go read this reference" in the how-it-works section. The model only reads that content when it actually reaches that step; it does not occupy context at other times.

**Rules are not copied into skill bodies — they are attached via path and role automatically.** One kind is directory-level rules: each rule declares which paths it governs, and the rule is automatically loaded into context when an agent enters that path. Another kind is role-binding: when a role is dispatched, the checklists that role is supposed to follow are loaded automatically. The hardest constraints are not path-bound at all — they are always loaded from startup. This way the authoritative source for rules is a single copy; skills and roles "reference" it rather than each maintaining their own copy.

**Docs are a consumed knowledge base — references must be explicit.** Docs hold long-term knowledge and decision records; skills read them when they need context. One discipline I care about especially: **every piece of long-term knowledge in docs should have an explicit consumer (some skill, rule, or role) that references it.** Knowledge with no consumer is orphaned knowledge — it gets written and never read, changes without anyone knowing, and gradually diverges from reality. When adding a new docs entry, immediately note who will use it; only then does that knowledge actually enter the system rather than pile up in a folder nobody opens.

The three together form a reference network: skill bodies are thin, name their own references explicitly, passively pick up path-scoped and role-bound rules, and actively fetch background from docs — while each piece of knowledge has exactly one authoritative location.

### How to organize scripts

The `scripts/` directory holds deterministic tools: validators, command wrappers, procedural helper scripts. Its purpose is to **reduce the space for the agent to improvise.** For anything with a single correct approach, do not have the model figure it out from scratch each time — write a script and nail it down.

A few hard requirements I hold for scripts: prefer standard libraries and POSIX shell, minimize external dependencies; any script that writes a state file must use atomic writes and locking for concurrency; command wrappers only invoke entry points declared in the config, with full logs written to disk and only a summary returned to the main session. One that is easy to skip: new scripts must have tests; changes to procedural scripts require running regression tests, not just a syntax pass. Scripts exist to pin down deterministic behavior — if the scripts themselves are unreliable, they serve no purpose.

## 4. How to Layer Skills, and Why

The last layer of organization is how to layer within the skill system.

I use three layers, with strictly non-overlapping responsibilities:

- Orchestration layer (orchestrator): typically just one, maintains the state machine and delegates control, does not do concrete work
- Phase layer (phase): corresponds to each segment of the delivery chain — requirements, design, development, review, testing, release — each managing its own inputs, outputs, and gate
- Atomic layer (atom): the bottom level, single deterministic capabilities, each does exactly one thing, does not orchestrate other skills, does not invent commands

The phase layer has a key design element called the gate. The end of a phase is not simply "done" — it produces a four-state conclusion: `pass`, `blocked` (with responsible owner and fallback path), `not_required` (this phase does not apply to this module), or `risk_accepted` (issue exists but has been explicitly accepted). For example, a module that does not render any UI directly marks its UI fidelity phase as `not_required` and proceeds. But if a required design reference is missing, it does not push through on guesswork — it marks `blocked` and returns upstream to fill the gap. **When evidence is missing, it is better to block than to pretend completion.**

Keeping three layers only in your head is useless — they need to become machine-readable, machine-verifiable artifacts. I materialize the entire graph as a manifest file, which is the single source of truth for all skills. Each node declares fixed fields: which layer it belongs to, how it is executed (script-backed or model-driven), a routing description (must include "applicable / not applicable / typical trigger phrases"), inputs and outputs, available tools, risk level, and whether manual review is required.

Nodes are connected by three types of edges:

- handoff: inter-phase transitions, driven by the orchestrator
- dynamic_load: based on the current task type, dynamically load the relevant capability into the sub-agent's fresh context — load what you use, do not bulk-load everything
- semantic: evidence dependency — for example, a "find candidates" step only produces a candidate map; actual pinpoint evidence must be converged by a downstream evidence skill; seeing a candidate is not the same as reaching a conclusion

**Why bother layering at all?** Because these things change at different rates and fail in different modes. Orchestration logic changes rarely, but when it does it is global. Phase contracts change at medium frequency, in a single segment's inputs and outputs. Atomic capabilities change most often, but each change touches only one point. Mix them together and you cannot change one layer without worrying about collapsing the others. With layers, each capability can be routed, tested, and changed independently without silently pulling other parts along.

## One Sentence to Remember

> **The spec for a complex task comes down to three assets: how the workflow is orchestrated, how skills are organized, how the knowledge base is connected. Getting these three things right separately is far more useful than writing a longer prompt.**

Looking back at this post, it maps cleanly onto those three assets.

**Workflow orchestration** is the first two sections: multi-agent determines who gets what work and what each is bound by; the orchestrator entry determines where to enter, who is in charge, and how to record issues and run a full chain without stopping to check in repeatedly.

**Skill organization** runs through sections three and four: how a skill's files are laid out (thin entry + on-demand long content + deterministic scripts), and how the full skill set is divided into orchestration, phase, and atomic layers so that adding more capabilities does not create chaos.

**Knowledge base wiring** is the reference network in section three: rules attach via path and role automatically, every docs entry must have a consumer, skills only name their own references. Each piece of knowledge has one authoritative location — change it once and it takes effect everywhere, no contradictory copies scattered around.

Together these three make "what the AI is actually doing" traceable: who does each step, what they must follow, what evidence they must produce, where to fall back when blocked — all written in the spec, not buried in some improvised session response.

This is the same theme as the first post — restraint. Do not let capabilities grow unchecked; every addition must state its position and contract.

But writing specs well only solves "is the organization clear". It does not answer "is each skill actually working well". A routing description written beautifully — will the model reliably invoke it at the right moment? A phase that claims to output a gate conclusion — does it actually produce one every time? Those require testing.

The next post covers how to test whether a skill is working well. Three things mainly.

First, how to run a comparison. Take the same question, run it once with the skill loaded and once without, compare where the outputs differ. Only then can you tell whether the improvement came from this skill or was something the general rules would have done anyway — you will not misattribute credit.

Second, how to persist results. Every actual model response goes into a file and is committed to the repository, so anyone can pull it up and review it later — not "run and forget, start fresh next time".

Third, why I do not use another large model to score the output. Having an AI act as judge sounds convenient, but the judge itself is unstable — scoring 8 today and 6 tomorrow. I prefer a fixed set of pass/fail checks that are unambiguous.

---

## Harness Engineering Series

A step-by-step breakdown of the platform layer (harness) and business engineering for coding agents:

1. [What Is a Harness, Exactly](./01-what-is-harness.md) — the boundary between platform layer and business engineering
2. [How to Write Specs for Complex Tasks](./02-how-to-write-specs.md) — multi-agent, orchestrator entry, rules / docs / skills organization
3. [How to Extend the Harness](./03-extending-the-harness.md) — skills, config directories, and hooks: the two mechanisms in CC and Codex
4. [How the Harness Controls the Agent](./04-permissions-and-effort.md) — permissions and effort: the control surface in CC and Codex
5. How the Harness Handles Long Tasks — compaction, memory, goal (in progress)

> Other languages: [English](../en/02-how-to-write-specs.md) · [한국어](../ko/02-how-to-write-specs.md) · [日本語](../ja/02-how-to-write-specs.md) · [中文](../zh/02-how-to-write-specs.md)
