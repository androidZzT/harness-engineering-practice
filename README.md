**English** ∙ [한국어](README.ko.md) ∙ [日本語](README.ja.md) ∙ [中文](README.zh-CN.md)

<div align="center">

# Harness Engineering

**A practical guide to the platform layer of coding agents — Claude Code & Codex.**

<img src="./assets/cover.png" alt="Harness Engineering" width="640">

</div>

---

## What is Harness Engineering?

When you build with a coding agent like **Claude Code** or **Codex**, your code is only half the story. The other half is the *harness* — the runtime around the model that manages context, memory, tool calls, sub-agents, skills, hooks, and the loop that holds it all together.

> *"the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."* — OpenAI, [Harness engineering](https://openai.com/index/harness-engineering/)

**Harness Engineering** is the discipline of building *on top of* that layer instead of fighting it. The central claim of this series is a boundary:

> **The harness is the coding agent's platform layer. Your business engineering is a set of specs built on top of it — and you should not rewrite the harness itself.**

Get that boundary right and a lot of agent pain disappears: context windows stop overflowing, sub-agents stop leaking across phases, and every platform upgrade is a free upgrade instead of a three-hour compatibility audit.

This is a hands-on, source-grounded series comparing how the two harnesses most engineers actually use — **Claude Code (Anthropic)** and **Codex (OpenAI)** — solve the same problems: skills, configuration directories, hooks, permissions, and reasoning effort.

## Who is this for

- Engineers building serious workflows on Claude Code, Codex, Cursor, or any agentic coding tool.
- Anyone who has felt the urge to "just patch the agent" and wants to know when *not* to.
- Readers interested in **agentic engineering**, **context engineering**, and **spec-driven development (SDD)** as practical disciplines rather than buzzwords.

## Contents

| # | Chapter | What it covers |
|---|---------|----------------|
| 01 | [**What "Harness" Actually Means**](./en/01-what-is-harness.md) | The boundary between the platform layer and business engineering; why you shouldn't modify the harness |
| 02 | [**How to Write Specs for Complex Tasks**](./en/02-how-to-write-specs.md) | Multi-agent orchestration, the orchestrator entry point, and how to organize rules / docs / skills |
| 03 | [**Extending the Harness**](./en/03-extending-the-harness.md) | Skills, configuration directories, and hooks — the two extension models of Claude Code vs. Codex |
| 04 | [**Controlling the Agent: Permissions & Effort**](./en/04-permissions-and-effort.md) | The control surface — permission modes and reasoning effort — compared across Claude Code and Codex |
| 05 | [**Spec and Knowledge Base: Making Sure the Agent Reads and Obeys**](./en/05-knowledge-base.md) | The ways a knowledge base reaches the agent; Linked ≠ Loaded ≠ Read ≠ Obeyed, and how to verify |
| 06 | *Surviving Long Tasks: Compaction, Memory, Goals* | Coming soon |

## Adjacent topics

- [**On AI-Ready and AI-SDLC**](./en/ai-ready.md) — before grinding on an Agents platform, make your engineering AI-Ready first (a standalone piece, not a series chapter)

## The one idea

If you remember one thing:

> **Your job is to write the specs and compose the primitives the harness exposes — not to modify the harness itself. Restraint is the core of the engineering discipline.**

## Further reading

The series is grounded in primary sources. The most useful ones:

- OpenAI — [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- Anthropic — [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- Anthropic — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Simon Willison — [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/)
- LangChain — [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)

## Translations

This guide is available in **English**, **한국어**, **日本語**, and **中文**. The Chinese version is the original; the others are translations. Spotted a translation issue? PRs are welcome — see [GLOSSARY.md](./GLOSSARY.md) for the locked terminology.

## License

Licensed under [CC BY 4.0](./LICENSE) — share and adapt with attribution.
