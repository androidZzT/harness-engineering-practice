[English](README.md) ∙ **한국어** ∙ [日本語](README.ja.md) ∙ [中文](README.zh-CN.md)

<div align="center">

# Harness Engineering

**Coding agent의 플랫폼 층에 대한 실용 가이드 — Claude Code와 Codex.**

<img src="./assets/cover.png" alt="Harness Engineering" width="640">

</div>

---

## Harness Engineering이란 무엇인가?

**Claude Code**나 **Codex** 같은 coding agent로 작업할 때, 당신의 코드는 이야기의 절반에 불과하다. 나머지 절반은 *harness*다—컨텍스트、메모리、도구 호출、sub-agent、skill、hook, 그리고 이 모든 것을 하나로 묶는 루프를 관리하는 모델 주변의 런타임.

> *"the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."* — OpenAI, [Harness engineering](https://openai.com/index/harness-engineering/)

**Harness Engineering**은 그 층과 싸우는 것이 아니라 그 위에 구축하는 규율이다. 이 시리즈의 핵심 주장은 하나의 경계다：

> **Harness는 coding agent의 플랫폼 층이다. 당신의 비즈니스 엔지니어링은 그 위에 구축된 spec 세트다—harness 자체를 다시 작성해서는 안 된다.**

그 경계를 올바르게 설정하면 많은 agent의 고통이 사라진다：컨텍스트 창이 더 이상 넘치지 않고, sub-agent가 단계를 가로질러 누수되지 않으며, 모든 플랫폼 업그레이드가 세 시간짜리 호환성 감사 대신 무료 업그레이드가 된다.

이것은 실제로 대부분의 엔지니어가 사용하는 두 가지 harness—**Claude Code (Anthropic)**와 **Codex (OpenAI)**—가 같은 문제를 어떻게 해결하는지를 비교하는 실용적이고 소스에 근거한 시리즈다：skill、설정 디렉토리、hook、권한、reasoning effort.

## 이 시리즈는 누구를 위한 것인가

- Claude Code、Codex、Cursor 또는 다른 agentic coding 도구로 본격적인 워크플로를 구축하는 엔지니어.
- "agent를 패치하고 싶다"는 충동을 느껴본 적이 있고, 언제 *하지 말아야 할지* 알고 싶은 사람.
- **agentic engineering**、**context engineering**、**spec-driven development (SDD)**에 버즈워드가 아닌 실용적 규율로서 관심 있는 독자.

## 목차

| # | 장 | 다루는 내용 |
|---|---------|----------------|
| 01 | [**"Harness"란 정확히 무엇인가**](./ko/01-what-is-harness.md) | 플랫폼 층과 비즈니스 엔지니어링의 경계；왜 harness를 수정하면 안 되는가 |
| 02 | [**복잡한 작업의 Spec은 어떻게 쓰는가**](./ko/02-how-to-write-specs.md) | 멀티 agent 편성、편성자 진입점、rules / docs / skills 구성 방법 |
| 03 | [**Harness는 어떻게 확장하는가**](./ko/03-extending-the-harness.md) | skill、설정 디렉토리、hook — Claude Code와 Codex의 두 가지 확장 모델 |
| 04 | [**Harness는 어떻게 agent를 제어하는가：권한과 Effort**](./ko/04-permissions-and-effort.md) | 제어 면 — 권한 모드와 reasoning effort — Claude Code와 Codex 비교 |
| 05 | *긴 작업을 버티는 방법：Compaction、Memory、Goals* | 출시 예정 |

## 핵심 아이디어

한 가지만 기억한다면：

> **당신의 일은 spec을 작성하고 harness가 노출하는 원시 요소를 조합하는 것이다—harness 자체를 수정하는 것이 아니다. 절제가 엔지니어링 규율의 핵심이다.**

## 더 읽을거리

이 시리즈는 일차 출처에 근거한다. 가장 유용한 것들：

- OpenAI — [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- Anthropic — [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- Anthropic — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Simon Willison — [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/)
- LangChain — [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)

## 번역

이 가이드는 **English**、**한국어**、**日本語**、**中文** 네 가지 언어로 제공된다. 중국어 버전이 원본이며, 나머지는 번역이다. 번역 오류를 발견했다면? PR 환영—잠긴 용어는 [GLOSSARY.md](./GLOSSARY.md)를 참조하라.

## 라이선스

[CC BY 4.0](./LICENSE) 라이선스—귀속 표시와 함께 공유 및 수정 가능.
