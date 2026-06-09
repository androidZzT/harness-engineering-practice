# Translation Glossary

To keep the four language editions consistent, translators must follow these rules.

## Keep verbatim (do NOT translate)

Product and tool names, code identifiers, CLI commands, file names, and config keys:

`Claude Code` · `Codex` · `Cursor` · `Claude Agent SDK` · `codex-core` · `codex-rs` ·
`CLAUDE.md` · `AGENTS.md` · `.claude` · `.codex` · `.inbox` · `MCP` · `git` · `worktree` ·
`SessionStart` · `Stop` · `PreCompact` · `PostCompact` · `SandboxPolicy` · `AskForApproval` ·
`ReasoningEffort` · and any inline `code`, file paths, or `file.rs:123` source citations.

## Key terms — translate consistently

| English | Note |
|---------|------|
| **harness** | Keep the English word "harness" (it is the subject of the whole series). Do not translate it to a generic word for "framework". Transliterate only if that is the natural convention in the target language. |
| **Harness Engineering** | Proper noun — keep in English, optionally with a parenthetical gloss on first use. |
| platform layer | Translate consistently; this is the harness. |
| business engineering | The spec layer built on top of the harness. Translate consistently. |
| coding agent | Translate consistently (or keep English where natural). |
| skill / hook / spec / sub-agent / orchestrator | Translate consistently; keep the English term in parentheses on first use if the concept is unfamiliar. |
| permission mode | The CC/Codex control surface for tool permissions. |
| reasoning effort | The model effort/thinking-budget control. |
| context window / compaction / memory | Standard LLM-runtime terms. |
| SDD (Spec-Driven Development) | Keep the acronym; gloss on first use. |

## Style

- Preserve every Markdown heading level, list, blockquote, link, and the source-citation links verbatim — only translate the prose.
- Keep all external links (OpenAI / Anthropic / Simon Willison / LangChain) exactly as they are.
- Match the original's plain, direct, engineer-to-engineer voice. Do not add marketing tone, do not embellish.
- Do **not** introduce any project names, company names, or personal identifiers. The text is deliberately generic.
