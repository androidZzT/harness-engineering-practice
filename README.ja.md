[English](README.md) ∙ [한국어](README.ko.md) ∙ **日本語** ∙ [中文](README.zh-CN.md)

<div align="center">

# Harness Engineering

**coding agent のプラットフォーム層（Claude Code & Codex）を実践的に解説するガイド。**

<img src="./assets/cover.png" alt="Harness Engineering" width="640">

</div>

---

## Harness Engineering とは何か

**Claude Code** や **Codex** といった coding agent を使って開発するとき、あなたのコードはその半分にすぎない。もう半分は *harness* だ——コンテキスト・メモリ・ツール呼び出し・sub-agent・skill・hook、そしてそれらすべてをつなぐループを管理するモデル周辺の実行時環境のことだ。

> *"the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."* — OpenAI, [Harness engineering](https://openai.com/index/harness-engineering/)

**Harness Engineering** とは、その層の上に構築するのであって、それに逆らわないという実践の規律だ。このシリーズの核心的な主張は一つの境界にある：

> **harness は coding agent のプラットフォーム層である。あなたの業務工学はその上に構築された一組の spec であり、harness 自体を書き直すべきではない。**

この境界を正しく引けば、agent に関する多くの痛みが消える：コンテキストウィンドウが溢れなくなり、sub-agent がフェーズをまたいでリークしなくなり、プラットフォームのアップグレードが 3 時間の互換性監査ではなく、無料のアップグレードになる。

これは、エンジニアが実際に使っている二つの harness——**Claude Code（Anthropic）** と **Codex（OpenAI）**——が同じ問題をどう解いているかを比較する、実践的でソースに基づいたシリーズだ：skill・設定ディレクトリ・hook・権限・reasoning effort。

## 対象読者

- Claude Code・Codex・Cursor、またはその他の agentic coding ツールの上で本格的なワークフローを構築しているエンジニア。
- 「agent にパッチを当てればいい」という衝動を感じたことがあり、*いつやるべきでないか*を知りたい人。
- **agentic engineering**・**context engineering**・**spec-driven development（SDD）** をバズワードではなく実践的な規律として理解したい読者。

## 目次

| # | 章 | 内容 |
|---|---------|----------------|
| 01 | [**「harness」とは何か**](./ja/01-what-is-harness.md) | プラットフォーム層と業務工学の境界；なぜ harness を変更すべきでないか |
| 02 | [**複雑なタスクの spec の書き方**](./ja/02-how-to-write-specs.md) | マルチ agent のオーケストレーション・オーケストレーターエントリポイント・rules / docs / skills の整理方法 |
| 03 | [**harness の拡張方法**](./ja/03-extending-the-harness.md) | skill・設定ディレクトリ・hook——Claude Code と Codex の二つの拡張モデル |
| 04 | [**agent の制御：権限と effort**](./ja/04-permissions-and-effort.md) | コントロールサーフェス——権限モードと reasoning effort——の Claude Code と Codex 比較 |
| 05 | [**Spec と知識ベース：エージェントが本当に読み、従っているか確かめる**](./ja/05-knowledge-base.md) | 知識ベースがエージェントに届く経路；紐付け ≠ ロード ≠ 読まれた ≠ 遵守、そして検証の仕方 |
| 06 | *長いタスクを乗り切る：Compaction・Memory・Goals* | 近日公開 |

## 関連トピック

- [**AI-Ready と AI-SDLC について**](./ja/ai-ready.md) —— Agents プラットフォームを煮詰める前に、まず工学を AI-Ready にする（独立した小編、シリーズ章ではない）

## ひとつのアイデア

覚えておくことが一つあるとすれば：

> **あなたの仕事は、spec を書き、harness が公開するプリミティブを組み合わせることだ——harness 自体を変更することではない。克制は工学規律の核心だ。**

## 参考文献

このシリーズは一次資料に基づいている。最も有用なもの：

- OpenAI — [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- Anthropic — [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- Anthropic — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Simon Willison — [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/)
- LangChain — [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)

## 翻訳

このガイドは **English**・**한국어**・**日本語**・**中文** で利用できる。中文版がオリジナルで、他は翻訳版だ。翻訳の問題を見つけた場合は PR を歓迎する——用語については [GLOSSARY.md](./GLOSSARY.md) を参照。

## ライセンス

[CC BY 4.0](./LICENSE) のもとでライセンス——帰属表示を付けて共有・改変自由。
