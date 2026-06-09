# harness の拡張方法：skill・設定ディレクトリと hook

前稿「harness とは何か」で harness の境界を明確にした——プラットフォーム層であり、コンテキスト管理・メモリ・subagent オーケストレーション・ツール呼び出し・hook・実行ループを担い、業務工学はその上に構築するものであって変更すべきではない。本稿はさらに一層掘り下げる：harness 自体はどう拡張するのか。

具体的に三点論じる：skill に何を書くべきか・設定ディレクトリに何を置くか・hook をどう挂ける。CC と Codex の両方がこれら三つの能力を提供しており、どちらかにしかない能力だと思っている人も多いが、実際には両者にそれぞれ対応するものがある。真の差異は「どちらが持っているか」ではなく、**同じ拡張の問いに対して、二者が工学的に異なるアプローチを取ったこと**にある。

---

## 一、Skill —— 何を書くか、どうロード・トリガーするか


### 解決する問題

LLM のデフォルト動作は汎用だ。しかし具体的なプロジェクトで作業していると、大量の再利用可能なワークフローが蓄積される：特定ドメインのコードレビュー基準・あるフレームワークのベストプラクティス・固定のコミットメッセージフォーマット・毎回チェックすべき項目リスト。

これらのコンテンツを扱う一般的な方法は三つある：毎回手打ちする（繰り返しの手間）・システムプロンプトに入れる（グローバル汚染、全会話に付きまとう）・シェルスクリプトにラップする（agent コンテキストから切り離れる）。Skill は第四の解法だ：再利用可能なワークフロー・リソース・専門知識を自己完結したユニットにパッケージし、適切なタイミングで自動または手動でロードし、それ以外の時間は会話を干渉しない。

### Codex はどうするか

Codex の skill は自己完結したフォルダで、specialized workflows・tool integrations・domain expertise・bundled resources を提供する（`codex-rs/skills/src/assets/samples/skill-creator/SKILL.md` より）。

各 skill のエントリは `SKILL.md` で、frontmatter には三つのフィールドがある：

```yaml
name: my-skill              # 必須、呼び出し時の識別子
description: |              # 必須、「この skill をいつ使うべきか」の内容
  This skill should be used when...
metadata:
  short-description: 一行要約  # 省略可、リスト表示に使う
```

`description` フィールドの書き方がトリガーの品質を決める。その意味は「どんな状況で呼び出すか」であり、「何ができるか」ではない——前者がトリガー条件で、後者は能力説明だ。agent は前者でシナリオをマッチングするので、書き方が違えばトリガー率が大幅に下がる。

システム組み込み skill は `include_dir!` マクロでバイナリにコンパイルされ（`skills/src/lib.rs:10`）、起動時に `$CODEX_HOME/skills/.system` にインストールされる（`:24`）。毎回起動のたびに再インストールしないよう、fingerprint マーカーファイルで判定する——組み込みコンテンツのハッシュが変わっていなければインストールをスキップする（`:32`）。これが配布の安定性を保証し、システム skill の動作がアップグレード前後で一貫するようにしている。

ユーザー定義 skill は `$CODEX_HOME/skills/` に置き、追加登録は不要、再起動後に有効になる。

### CC はどうするか

CC の skill は `.claude/skills/<name>/SKILL.md` に置き、frontmatter には三つのフィールドがある：`name`・`description`・`disallowed-tools`。

トリガー方法は二つ：ユーザーが明示的に `/skill-name` で呼び出すか、Claude が会話のコンテキストから自動判断して呼び出すか。後者は完全に `description` の記述品質に依存する——明確に書けば書くほど、自動トリガーが正確になる。

ローカル skill のほかに、CC には plugin skill 機構がある：plugin マーケットプレイスからインストールし、`/plugin` で管理する。セッション内で `/reload-skills` で再スキャンできる（v2.1.152 以降サポート）、または `SessionStart` hook の `reloadSkills` パラメータで再スキャンをトリガーできる。

`disallowed-tools` は CC 側独自のフィールドだ：skill レベルで特定のツールを無効化できる。例えば「コードレビュー」skill はファイルの読み取りだけ必要でファイルの書き込みは不要なので、`disallowed-tools` で書き込みツールを無効化すると、この skill の実行中は書き込み操作がトリガーされず、誤操作のリスクを減らせる。

### 比較

| 次元 | Codex | CC |
|------|-------|----|
| 保存場所 | `$CODEX_HOME/skills/` | `.claude/skills/<name>/` |
| frontmatter フィールド | name / description / metadata.short-description | name / description / disallowed-tools |
| トリガー方法 | コンテキスト自動トリガー | 明示的 `/skill-name` ＋ コンテキスト自動トリガー |
| ツールレベルの制限 | 専用フィールドなし | `disallowed-tools` |
| 組み込み skill | バイナリにコンパイル、fingerprint 再インストール最適化 | plugin マーケットプレイス |
| ホットリロード | ファイルシステムの更新に追随 | `/reload-skills` または `SessionStart` hook |

### 実践的なアドバイス

skill を書くときの最もありがちな落とし穴は、`description` を能力紹介として書くことで、呼び出し条件として書かないことだ。悪い例：

> 能力紹介の書き方："This skill provides Python code review capabilities, including style checking and performance analysis."

トリガー率が低い。agent が「この能力がある」と理解するだけで、いつ呼び出すべきかわからないからだ。

条件トリガーの書き方に変える：

> "This skill should be used when reviewing Python code, checking for style issues, or analyzing performance bottlenecks in Python files."

その他の点：
- CC の `disallowed-tools` を活用する：各 skill のツール範囲を明確に制限し、skill 間の相互干渉を防ぐ
- システム組み込み skill は保守負担になる。Codex がそれらをバイナリにコンパイルするのは配布の一貫性のためであり、日常使いするものは直接ユーザーディレクトリに置けばよい
- ローカル skill はリポジトリにコミットし、チームで共有する——skill 自体が再利用可能なワークフロー資産であり、個人のマシンにだけ存在すべきではない

---

## 二、設定ディレクトリ —— .codex と .claude、何を置くか


### 解決する問題

Agent の動作には設定が必要だ：どのモデルを使うか・どのツールを許可するか・プロジェクトレベルのコード規約制約・企業レベルのセキュリティ境界。ここには自然な衝突がある：個人設定・プロジェクト制約・企業ポリシーの設定権限がそれぞれ異なり、すべてを一つのファイルに置いて「後勝ち」で単純に処理できない——企業のセキュリティ境界はプロジェクトの設定ファイルで上書きできてはならない。

設定ディレクトリの核心的な設計問題は：**誰の設定が誰の設定を上書きでき、どこでロックするか**だ。

### Codex はどうするか

Codex のプロジェクトレベル設定は `.codex/config.toml`（TOML 形式）だ。分層ロード順は `config/src/loader/mod.rs:79-100` に完全なコメントがあり、二つのロジックに分かれる：

**制約層**（constraint、企業・セキュリティポリシー用）：

```
cloud → admin → system (/etc/codex/requirements.toml)
```

この層は「早層ロック」意味論を使う：`a constraint defined in an earlier layer cannot be overridden by a later layer`（原文）。システム管理者が `/etc/codex/requirements.toml` に書いた制約は、プロジェクト層でも上書きできず、ユーザー層でも上書きできない。

**設定層**（通常設定、後の層が早い層を上書き）：

```
admin → system (/etc/codex/config.toml)
     → user ($CODEX_HOME/config.toml)
     → profile ($CODEX_HOME/<name>.config.toml)
     → cwd (./config.toml)
     → tree (./.codex/config.toml、上へ順に探す)
     → repo (git root/.codex/config.toml)
     → runtime (--config パラメータなど)
```

プロジェクトルートの判断は `project_root_markers` で決まり、デフォルトは `.git` だ。設定ファイルはプロジェクトルートに近いほど優先度が高い（runtime が最高）。

保護パスはもう一つのハード制約だ：`PROTECTED_METADATA_PATH_NAMES = [".git", ".agents", ".codex"]`（`protocol/src/permissions.rs:27`）。danger-full-access サンドボックスモードでも、agent はこの三つのディレクトリに書き込めない。これはシステムレベルの保護であり、オプションの設定ではない。

`.agents/` に subagent 定義を置き；`.codex/hooks.toml` に hook 設定を置く（次節で展開）。

### CC はどうするか

CC の設定ディレクトリは `.claude/` で、以下を含む：

- `settings.json`：JSON 形式、agent の動作設定
- `CLAUDE.md`：Markdown、システムプロンプトとコンテキスト説明
- `skills/`：ローカル skill ディレクトリ
- `agents/`：サブ agent 定義
- `commands/`：カスタムスラッシュコマンド
- `rules/`：パスのスコープに応じてロードするルールファイル
- `.mcp.json`：MCP ツール設定

優先度の高い順：managed（企業 IT プッシュ）> コマンドライン引数 > local > project（`.claude/`）> user（`~/.claude/`）。managed settings は企業層の強制ポリシーであり最高優先度、プロジェクト層では上書きできない。

`rules/` には個別に言及する価値のある特性がある：frontmatter の `paths:` フィールドでパスのスコープをサポートする。例えば `rules/python-style.md` に `paths: ["**/*.py"]` を書くと、このルールは Claude が Python ファイルを読むときだけコンテキストに入り、それ以外の時間はトークンを消費しない。大規模なリポジトリでは、この特性が無関係なルールのトークン消費を明確に削減できる。

### 比較

| 次元 | Codex | CC |
|------|-------|----|
| プロジェクト設定ファイル | `.codex/config.toml`（TOML） | `.claude/settings.json`（JSON）+ `CLAUDE.md`（MD） |
| 企業強制ポリシー | requirements.toml、制約層ロック、後の層は上書き不可 | managed settings、最高優先度 |
| 分層ロジック | 制約層（ロック）＋ 設定層（上書き）の二つの独立ロジック | 統一優先度チェーン、managed 最高 |
| 保護パス | `.git` / `.agents` / `.codex`、サンドボックスでも書き込み不可 | 同様の機構なし |
| パススコープルール | なし | `rules/` frontmatter `paths:` フィールド |
| subagent 定義 | `.agents/` | `.claude/agents/` |

二者は「企業ポリシーを上書きできないようにする」という方向は一致しているが、実装が異なる：Codex は制約層と設定層のマージロジックを分けて書き、制約層にはコードで明確な「早層ロック」意味論がある；CC は優先度チェーンに頼り、managed の最高優先度で保証する。

### 実践的なアドバイス

分層の核心的な価値は**設定の帰属を明確にすること**だ：

- 個人設定（どのモデルを使うか・スタイル）はユーザー層（`~/.codex/config.toml` または `~/.claude/settings.json`）に置き、リポジトリにコミットしない
- プロジェクト制約（ツールの権限範囲・コード規約の説明）はプロジェクト層に置いてリポジトリにコミットし、チームメンバーが同じ制約を共有できるようにする
- 企業のセキュリティ境界（特定ツールの呼び出し禁止・ネットワークアクセス制限）は管理層を使い、プロジェクト層で上書きしようとしない
- CC の `rules/` パススコープを十分に活用する：すべてのルールを一つの大きな `CLAUDE.md` に積み上げるより、言語やディレクトリ別に分割し、関連するものだけを毎回ロードする方がよい

---

## 三、Hook —— イベント・タイプ・信頼


### 解決する問題

Agent の実行中には多くの重要なポイントがある：ツール呼び出しの前後・セッションの開始と終了・ユーザーが prompt を送信したとき・コンテキスト圧縮時・サブ agent の起動停止時。これらのポイントでカスタムロジックを差し込むことで、様々なことができる：

- ファイルを書く前に確認を取る
- ツール呼び出し後に監査ログを記録する
- セッション終了時に完了度チェックを行う
- 権限申請時に自動決定する

二者はともにイベント駆動のライフサイクルモデルを使っているが、マウント設定・ハンドラータイプ・信頼機構はそれぞれ異なる。

### Codex はどうするか

Codex の hook 設定は `.codex/hooks.toml`（TOML 形式）にある。

完全なイベントリスト（`config/src/hook_config.rs` より、計 10 個）：

```
PreToolUse         ツール呼び出し前
PostToolUse        ツール呼び出し後
PermissionRequest  権限申請時
PreCompact         コンテキスト圧縮前
PostCompact        コンテキスト圧縮後
SessionStart       セッション起動
UserPromptSubmit   ユーザーが prompt を送信したとき
SubagentStart      サブ agent 起動
SubagentStop       サブ agent 停止
Stop               agent 停止
```

ハンドラータイプ（`HookHandlerConfig` enum、`hook_config.rs:137-156`）：

```rust
enum HookHandlerConfig {
    command { command, commandWindows, timeout, async, statusMessage },
    prompt {},
    agent {},
}
```

三種類——`command`（シェルコマンド）・`prompt`（LLM 評価）・`agent`（サブ agent 検証）。

信頼機構は `HookStateToml` 内の `trusted_hash: Option<String>` フィールドで実装されている（`hook_config.rs:29`）。各 hook は `state` セクションでハッシュを宣言する必要があり、ハッシュが現在のファイル内容と一致した場合のみ hook は自動実行される；不一致の場合はユーザーの承認が必要になる。

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

`trusted_hash` の設計の動機は悪意ある hook に対する防御だ：複数人で共有しているリポジトリで `.codex/hooks.toml` が改ざんされると、ハッシュが無効になり、hook はサイレントに実行されず承認をトリガーする——hook 自体を潜在的な攻撃対象として扱っている。

### CC はどうするか

CC の hook 設定は `settings.json` にある。公式ドキュメントは 32 個のイベントを列挙しており、以下を含む：SessionStart / SessionEnd / UserPromptSubmit / PreToolUse / PostToolUse / PostToolUseFailure / Stop / PermissionRequest / Notification / MessageDisplay など。

ハンドラータイプ：`command`（シェル）/ `http`（HTTP リクエスト）/ `mcp_tool`（MCP ツール）/ `prompt`（LLM 評価）/ `agent`（サブ agent 検証）。

覚えておく価値がある実行の約束事がいくつかある：
- `matcher` フィールドは正規表現をサポートし、ツール名またはイベントコンテンツにマッチする
- exit code 0 = 成功、2 = ブロック（Claude に現在の操作を停止するよう通知する）
- JSON は stdin/stdout を通じてコンテキストと結果を受け渡す

`prompt` タイプの実際の使い方：`Stop` hook の中で、小モデルに「タスクが本当に完了しているか」を判断させる prompt を書き、完了していなければ非 0 を返して Claude に継続させる。これで hook の役割が「監視＋遮断」から「監視＋判断＋フィードバック」に拡張され、agent loop の末尾に自動 QA 層を追加するのに等しい。

`http` タイプは CC 独自だ：シェルスクリプトを介さず直接 HTTP エンドポイントを呼べる。hook を外部の承認システムやログプラットフォームに接続するのに適している。

### 比較

| 次元 | Codex | CC |
|------|-------|----|
| 設定ファイル | `.codex/hooks.toml`（TOML） | `settings.json`（JSON） |
| イベント数 | 10 個 | 32 個 |
| ハンドラータイプ | command / prompt / agent | command / http / mcp_tool / prompt / agent |
| 独自イベント | SubagentStart / SubagentStop / PreCompact / PostCompact | SessionEnd / PostToolUseFailure / Notification / MessageDisplay / Setup / PermissionDenied など |
| 信頼機構 | `trusted_hash`：ハッシュが一致した場合のみ自動実行 | ファイルシステム権限に依存 |
| HTTP タイプ | なし（command で間接実装が必要） | あり、直接 HTTP エンドポイントを呼び出せる |

二者はともに `prompt` と `agent` タイプの hook を持っている——これは多くの人の印象とは異なる点だが、ソースコードの `HookHandlerConfig` enum にこの二つの variant が明確に存在する。差異は主に：Codex には `trusted_hash` によるファイルレベルの信頼機構がある（セキュリティ重視）；CC には HTTP タイプとより多くのイベントノードがある（統合重視）。

### 実践的なアドバイス

使用頻度から言えば、最もよく使う組み合わせ：

- `PreToolUse` ＋ 危険なツールにマッチする `matcher` ＋ `command` タイプ：`rm -rf`・`git push --force` の前にログを入れるか確認する
- `PostToolUse` ＋ `command`：ツール呼び出しの記録を監査ファイルに書き込み、事後に確認できるようにする
- `Stop` ＋ `prompt` タイプ：LLM に「今回のタスク目標が達成されたか」を判断させ、達成していなければ exit code 2 を返して Claude を継続させる
- Codex ユーザーは `trusted_hash` の保守に注意する：hook スクリプトを変更するたびにハッシュが無効になり、再度信頼する必要がある。これは期待される動作であってバグではないが、hook を頻繁に変更する場合は、ハッシュの更新をワークフローに組み込む必要がある

---

## まとめ

三つの拡張能力について、二者の設計方向をそれぞれ一文で言い表せる：

**Codex**：ビルド時にロックし（システム skill をバイナリにコンパイル）、実行時に検証する（trusted_hash）、制約層の早層ロックで上書き不可——セキュリティ境界を優先し、制約を前倒しにする。

**CC**：ファイルシステムを中心とし、パススコープルール・plugin マーケットプレイス・HTTP hook で拡張点をより柔軟に——組み合わせ可能性を優先し、柔軟性を外に延ばす。

二者とも「自分だけが持っている能力」はない。真の分岐は同じ拡張ニーズに対する工学的態度にある：Codex は「制約は早く確定するほど安全だ」と考え、CC は「組み合わせが柔軟なほど有用だ」と考える。

どちらの harness を使い、どの拡張方法を選ぶかは最終的にシナリオ次第だ——チームの規模・セキュリティ要件・外部システムとの統合の深さ。二つの哲学に優劣はなく、適するかどうかだけがある。

---

## Harness Engineering シリーズ

coding agent の「プラットフォーム層（harness）と業務工学」について、一記事ずつ解説する：

1. [harness とは何か](./01-what-is-harness.md) —— プラットフォーム層と業務工学の境界
2. [複雑なタスクの spec の書き方](./02-how-to-write-specs.md) —— マルチ Agent・オーケストレーターエントリポイント・rules / docs / skills の整理
3. [harness の拡張方法](./03-extending-the-harness.md) —— skill・設定ディレクトリと hook：CC と Codex の二つの機構
4. [harness で agent を制御する](./04-permissions-and-effort.md) —— 権限と effort：CC と Codex のコントロールサーフェス
5. harness で長いタスクを乗り切る —— compact・memory・goal（執筆中）

> 他の言語：[English](../en/03-extending-the-harness.md) · [한국어](../ko/03-extending-the-harness.md) · [日本語](../ja/03-extending-the-harness.md) · [中文](../zh/03-extending-the-harness.md)
