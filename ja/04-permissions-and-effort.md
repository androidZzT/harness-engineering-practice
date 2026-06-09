# harness で agent を制御する：権限と effort

前稿「harness の拡張方法：skill・設定ディレクトリと hook」は「harness に何かを追加する」話だった——再利用可能なワークフローを skill としてパッケージし、制約を設定ディレクトリに書き込み、監視ロジックを hook に挂ける。本稿では角度を変える：agent にツールを揃えた後、どうやって管理するか。

二つの具体的な問いがある：一つ、agent がファイルを変更したり、コマンドを実行したり、ネットワークリクエストを送ったりするとき、どの操作は確認を求め、どれは自動で通すかをどう決めるか；二つ、同じモデルで、シナリオに応じて「深く考える度合い」をどう調整し、トークンを無駄にせず、かつ複雑なタスクでも手を抜かないようにするか。

CC と Codex の両方にこれら二つの問いへの解法がある。前稿と同様、差異は「どちらが持っているか」ではなく、**同じコントロールの問いに対して、二者が工学的に異なるアプローチを取ったこと**にある。

---

## 一、権限


### 解決する問題

Agent は実行中にファイルを変更し、コマンドを実行し、ネットワークリクエストを送る。完全に開放するとリスクがあり、毎回確認を求めると効率が下がる。核心的な矛盾は：**「割り込みの少なさ」と「制御可能性」のバランスをどこで取るか、そしてそのバランスを「個人 / プロジェクト / 企業」という層別で固定できるかどうか**だ。

### Codex はどうするか

Codex は認可を二つの直交する次元に分割し、組み合わせて認可マトリクスを構成する。

**次元一：サンドボックスポリシー**、agent の操作境界を定義する。列挙型 `SandboxPolicy`（`codex-rs/protocol/src/protocol.rs:878`）、4 つのバリアント：

- `DangerFullAccess`（シリアライズ `"danger-full-access"`）—— 制限なし
- `ReadOnly { network_access: bool }` —— 読み取りのみ、ネットワークはデフォルトオフ
- `ExternalSandbox { network_access }` —— 外部サンドボックス内、ディスクを開放するが渡されたネットワーク設定に従う
- `WorkspaceWrite { writable_roots, network_access, exclude_tmpdir_env_var, ... }` —— ReadOnly の上に現在のワークスペース＋列挙した `writable_roots` への書き込みを追加許可

**次元二：承認ポリシー**、いつ停止してユーザーに確認するかを決める。列挙型 `AskForApproval`（`codex-rs/protocol/src/protocol.rs:784`）、5 つのバリアント：

- `UnlessTrusted`（`"untrusted"`）—— 「安全かつ読み取り専用」と知られているコマンドだけ自動承認、それ以外はすべて確認
- `OnFailure`（DEPRECATED 済み）—— サンドボックス内で全自動承認、失敗時のみユーザーにエスカレート
- `OnRequest`（`#[default]`、デフォルト）—— モデルが確認すべきタイミングを判断する
- `Granular(GranularApprovalConfig)` —— カテゴリ別に細かく開閉する、あるカテゴリを `true` にすれば通過、`false` にすれば自動拒否（ユーザーには表示しない）
- `Never` —— 一切確認しない、失敗はモデルに直接返し、エスカレートしない

`Granular` バリアント内の `GranularApprovalConfig`（同ファイル `protocol.rs`、`AskForApproval` 定義直後）には 5 つのカテゴリフィールドがある：`sandbox_approval`（シェルコマンド承認、インライン昇格を含む）・`rules`（execpolicy `prompt` ルールがトリガーするプロンプト）・`skill_approval`（skill スクリプトの実行）・`request_permissions`（`request_permissions` ツールがトリガー）・`mcp_elicitations`（MCP elicitation プロンプト）。

二つの次元の分担：サンドボックスポリシーが「境界」（ファイル読み書き範囲＋ネットワーク開閉）を管理し、承認ポリシーが「割り込みリズム」（いつ停止して確認するか）を管理する。両者は独立しており、任意に組み合わせられる。

### CC はどうするか

CC は認可を三層の積み重ねで構成する：モード（ベースライン）＋ ルール（allow/ask/deny）＋ 分類器（auto モードがリアルタイム審査）。

**6 種類の権限モード**（permission-modes ページ）、Shift+Tab で `default→acceptEdits→plan` を循環：

- `default` —— 読み取りのみの操作は確認しない
- `acceptEdits` —— 読み取り＋ファイル編集＋作業ディレクトリ内の `mkdir`/`touch`/`mv`/`cp`/`rm`/`rmdir`/`sed` は確認しない
- `plan` —— 読み取りのみ、調査してプランを提示、ソースコードは変更しない
- `auto` —— ほぼ確認しないが、独立した分類器モデルがアクション実行前にリアルタイム審査する；research preview と注記あり；Claude Code v2.1.83+ が必要、モデルは Opus 4.6+ または Sonnet 4.6 が必要
- `dontAsk` —— 事前承認されたツールだけ実行、それ以外はすべて自動拒否；CI やスクリプトのロックに適している
- `bypassPermissions` —— すべてのチェックをスキップして一切確認しない、`--dangerously-skip-permissions` 相当；アイソレートされたコンテナ/VM でのみ使用；`rm -rf /` と `rm -rf ~` は引き続きフェイルセーフプロンプトがある

`bypassPermissions` を除き、protected paths（`.git`/`.claude`/`.vscode`/`.idea` など）への書き込みは自動承認されない。

**三種類のルール**（permissions ページ）：`allow`（確認なしで使用）・`ask`（毎回確認）・`deny`（禁止）。評価順は deny → ask → allow で、最初にマッチしたものが勝ち、deny は常に優先される。

細部として覚えておく価値がある：ベアのツール名 deny（例：`Bash`）はツールをコンテキストから完全に除去する；スコープ付き deny（例：`Bash(rm *)`）はツールを残し、マッチする呼び出しだけを遮断する。

**settings の 5 段階 precedence**：Managed（企業 IT プッシュ、最高、いかなる層でも上書き不可）> コマンドライン引数 > Local（`.claude/settings.local.json`）> Project（`.claude/settings.json`）> User（`~/.claude/settings.json`）。いずれかの層で deny されると、他の層では allow できない。

**auto モードの分類器**（permission-modes ページ）にはいくつかの詳細がある：

独立した分類器モデルがアクション実行前に審査し、三種類のシナリオを遮断する：要求範囲を超えた昇格・知らないインフラへのアクセス・読み込んだ敵意コンテンツに駆動されたアクション。判断の順序は、まず allow/deny ルールを適用し、読み取りおよび作業ディレクトリ内の編集はそのまま通過させ、それ以外は分類器に渡し、遮断時は理由をモデルに返す。

auto モードに入ると、広すぎる allow ルール（`Bash(*)`・ワイルドカードインタープリタ・パッケージマネージャの run・`Agent` ルール）は削除され、離れると復元される。分類器は user メッセージ・ツール呼び出し・`CLAUDE.md` だけを見る、ツール結果は除去される（注入防止）。

フォールバック閾値：連続 3 回または累計 20 回遮断されると、auto モードが一時停止し、逐一確認モードに戻る（この閾値は設定不可）。

セッション中に口頭で伝えた境界（「まだ push しないで」）はブロックシグナルとして扱われるが、永続化されない。compaction でそのメッセージが削除されると失効する可能性があるので、ハードな保証が必要なら deny ルールを書くこと。

### 比較

| 次元 | Codex | CC |
|------|-------|----|
| 核心モデル | 二つの直交する次元：サンドボックスポリシー × 承認ポリシー | 三層の積み重ね：モード＋ルール＋分類器 |
| 段階の粒度 | 4 サンドボックス × 5 承認、組み合わせマトリクス | 6 種類のモード（dontAsk 全事前承認を含む） |
| 最も細かい粒度の制限 | `Granular`：5 つのカテゴリフィールドを独立して開閉 | スコープ付き deny：`Bash(rm *)` レベル |
| セマンティック審査 | なし（確定的なルール組み合わせ） | auto モード分類器、セマンティックな審査 |
| 企業ポリシーのロック | サンドボックスポリシー＋承認ポリシー制約の組み合わせ | Managed settings、5 段階優先度の最高 |

Codex は「ポリシー審査」寄り：二つの確定的次元の直交組み合わせ、オープンソースで検証可能、動作が予測可能。CC は「モデル審査＋ポリシーのフォールバック」寄り：auto モードで独立した分類器がセマンティック判断をし（「要求範囲を超えているか / 外部インフラかどうか / 注入に駆動されているか」を判断）、deny ルール層がフォールバックする。

### 実践的なアドバイス

**Codex**：日常は `WorkspaceWrite` ＋ `OnRequest`（デフォルトの組み合わせ）；無人バッチ実行は `Never`、ただし必ずサンドボックスを絞り込む（ネットワークオフ、`writable_roots` を制限）；`Granular` は「シェルは通すが MCP / skill スクリプトはブロックする」といった精細なシナリオに適している。

**CC**：レビュー付きコーディングは `acceptEdits`、エディタや `git diff` で事後確認する；長いタスクで割り込みを減らすには `auto`、ただし research preview なので機密性の高い操作は引き続き人が確認する；CI は `dontAsk` ＋ 事前承認 allow リスト；`bypassPermissions` はアイソレートされたコンテナのみ。

**共通**：企業のセキュリティ境界は Managed または制約層でロックし、プロジェクト層で上書きしようとしない；ハードな境界は deny ルールを書き、会話で口頭で言うだけに頼らない（compaction でそのメッセージが削除される可能性がある）。

---

## 二、effort


### 解決する問題

同じモデルで、「少し考えて素早く答える」こともでき、「じっくり考えて深く答える」こともできる。単純なタスクで考えすぎるとトークンを浪費し、overthinking になる可能性もある；複雑なタスクで考えなさすぎると不十分になる。だから「どれだけ深く考えるか」を調整できる設定が必要で、モデルごと・シナリオごとにデフォルト値を設定できる必要がある。

### Codex はどうするか

**`ReasoningEffort` 列挙型 6 段階**（`codex-rs/protocol/src/openai_models.rs:45`）：

`None` / `Minimal` / `Low` / `Medium`（`#[default]`）/ `High` / `XHigh`、シリアライズは `"none"/"minimal"/"low"/"medium"/"high"/"xhigh"`、デフォルトは `Medium`。

**クライアントは一つのことだけする：透過的に転送する**。`build_reasoning`（`codex-rs/core/src/client.rs:715`）は effort をリクエストに組み立て、キーとなる行は `effort: effort.or(model_info.default_reasoning_level)`（`:722`）で、最終的に OpenAI Responses API リクエストボディの `{"reasoning":{"effort":"high"}}` になる。

Codex クライアントはシステムプロンプトを変えず、ツールリストを変えず、コンテキストウィンドウを調整せず、ローカルでの推論強化は一切行わない。文字列をサーバーに送るだけで、`"high"` を実際の算力にどう変換するかはクラウドモデルの仕事だ。

**三層のフォールバック**（`effective_reasoning_effort`、`codex-rs/core/src/session/turn_context.rs:125`）：現在のターンでユーザーが設定した値 → モデルのデフォルト段階（`default_reasoning_level`）→ モデルが reasoning をサポートしていなければ `reasoning` フィールドを送らない（注意：`"none"` を送るのではなく、フィールドを送らない）。

**モデル切り替え時の段階マッピング**：`effort_rank`（`openai_models.rs:557-562`、None=0 / Minimal=1 / Low=2 / Medium=3 / High=4 / XHigh=5）と `nearest_effort`（`:566`）を組み合わせて、rank の差の絶対値が最小になるよう、対象モデルがサポートする最も近い段階にマッピングする。

**Plan モード独立 effort**：`plan_mode_reasoning_effort`（`codex-rs/core/src/config/mod.rs:873`）と実行フェーズは独立したフィールドで、Plan フェーズには単独でより高い段階を設定できる。

### CC はどうするか

**段階はモデルによって異なる**（model-config ページ）：

- Opus 4.8 / Opus 4.7：`low` / `medium` / `high` / `xhigh` / `max`
- Opus 4.6 / Sonnet 4.6：`low` / `medium` / `high` / `max`
- デフォルト段階：Opus 4.8・Opus 4.6・Sonnet 4.6 はいずれも `high`；Opus 4.7 は `xhigh`
- 現在のモデルがサポートしていない段階を設定すると、その段階を超えない最高サポート段階にフォールバックする（例：Opus 4.6 で `xhigh` を設定すると実際は `high` で実行）

**永続性の違い**：`low`/`medium`/`high`/`xhigh` はセッションをまたいで永続する；`max` は現在のセッションのみ有効（`CLAUDE_CODE_EFFORT_LEVEL` 環境変数経由でない限り）。`max` は最も深い推理・トークン上限なしの段階で、公式ドキュメントの原文は「prone to overthinking, test before adopting broadly」。

**さらに二つのものがあり、これはモデルの段階ではなく CC 独自のものだ**：

`ultracode` は単独で詳しく説明する価値がある。よく「Opus 4.8 の追加推理段階」と誤解されるが、そうではない。これは CC のセッションレベル設定だ：有効にするとモデルに `xhigh` を送り、同時に Claude が各実質的なタスクに対して自動的に **dynamic workflow** を編成するよう指示する——Claude がその場で書き、バックグラウンドで実行される JavaScript 編成スクリプトで、作業を数十から百以上の subagent に分割して並行実行する（dynamic workflow は v2.1.154 で導入）。

通常の subagent 編成との重要な違い：スクリプト自身がループ・分岐・中間結果を持ち、Claude のコンテキストには最終的な答えだけが残る。だから「一つの会話でコーディネートしきれない」大規模タスクに適している——全リポジトリのバグスキャン・数百ファイルの移行・複数ソースのクロス検証調査など；スクリプトで品質の定型手順を固定することもでき、例えば複数の agent が互いに対立的に他の結論をレビューしてから報告させることもできる。

トリガーは二つのルートがある：prompt で直接要求する（自分の言葉で、またはキーワード `ultracode` を書く）、または `/effort ultracode` で Claude がセッション全体を通じて各実質的なタスクに対して自動的にプランを立てるようにする。`xhigh` をサポートするモデルでのみ `/effort` メニューに表示される（Opus 4.8 / 4.7）、それ以外のモデルは提供されない。現在のセッションのみ有効で、`effortLevel` 設定・`--effort` フラグ・`CLAUDE_CODE_EFFORT_LEVEL` には属さない。

`ultrathink` キーワード：prompt の任意の位置に `ultrathink` と書くと、そのターンのより深い推理を要求する。セッションの effort は変わらず、API に送られる effort 値も変わらない。コンテキストに in-context の指示が追加されるだけだ。「think」/「think hard」などはキーワードとして認識されない。

**設定エントリポイント**：`/effort`（引数なしでスライダーを開く / `/effort <段階名>` で直接設定 / `/effort auto` でモデルデフォルトに戻す）；`/model` で左右キーでスライダーを調整；`--effort` フラグ；`CLAUDE_CODE_EFFORT_LEVEL` 環境変数；`effortLevel` 設定（low〜xhigh のみ受け付け、`max` と `ultracode` は受け付けない）；skill / subagent frontmatter の `effort` フィールド。優先度：環境変数 > 設定 > モデルデフォルト。

### 比較

| 次元 | Codex | CC |
|------|-------|----|
| 段階数 | 6 段階（None/Minimal/Low/Medium/High/XHigh） | モデルにより、最大 5 段階（low/medium/high/xhigh/max） |
| デフォルト段階 | `Medium` | 多くのモデルは `high`（Opus 4.7 は `xhigh`） |
| クライアントで処理するか | 処理しない（純粋な透過転送、API の一つのフィールドだけ変更） | 追加のラッピングがある（ultracode は後台ワークフロー編成をトリガー） |
| 単一ターンの一時的な深化 | 専用キーワードなし | `ultrathink` キーワード（セッション段階は変わらない） |
| max の永続性 | N/A | 現在のセッションのみ（環境変数での設定でない場合） |
| モデル切り替え時の段階マッピング | `nearest_effort`：絶対値最近段階（昇格する場合もある） | 対象段階を超えない最高サポート段階（降格のみ） |
| Plan フェーズ独立 effort | あり（`plan_mode_reasoning_effort` 独立フィールド） | 専用フィールドなし |

二者は low/medium/high/xhigh という中間の段階名を共有し、モデルのデフォルト段階とフォールバック機構もともにある。差異は：Codex は effort を「ローカルでは処理せず API に転送するだけのフィールド」として扱い、ソースコードで検証できる；CC は同じ段階のほかに独自のものをいくつか追加した——上には `max`（上限なし）・`ultracode`（effort に加えてバックグラウンドワークフロー編成を連動）・`ultrathink`（単一ターンの一時的な深化、セッション段階は変えない）。

フォールバックアルゴリズムも異なる：Codex の `nearest_effort` は rank 差の絶対値が最小になるようにマッピングし、対象段階より高くなる場合もある；CC は「対象段階を超えない最高サポート段階」を取り、降格のみで昇格しない。

### 実践的なアドバイス

**Codex**：デフォルトの `Medium` がほとんどのシナリオに適している；深い推理が必要なら `High` / `XHigh` に調整する；Plan モードには単独でより高い段階を設定する（計画は深く考える価値があるが、実行フェーズは同じ段階は必要ない）；段階を変えても、サーバーに送る文字列を一つ変えるだけでローカルで「より頑張る」わけではないことを覚えておく。

**CC**：日常は `high`（Opus 4.8 デフォルト）で十分；難問は一時的に深化させるには prompt に `ultrathink` と書くだけでセッション設定を変えなくてよい；Claude にタスクを自動分割して並行編成させたいなら `ultracode` を使う；`max` は慎重に使う、overthinking になる可能性があり、まず小規模に試してから使う。

**共通**：effort はモデルごとに調整されており、同じ段階名でも異なるモデルでは同じ算力を意味しない。モデルを切り替えたあとは実際に有効になっている段階を確認すること。

---

## まとめ

二節通じて、二者のコントロールサーフェスの方向をそれぞれ一文で言い表せる：

**Codex**：権限は「ポリシーマトリクス」（二つの確定的次元の直交組み合わせ、オープンソースで検証可能）、effort は「そのまま転送する段階」（どの段階に設定してもそのまま送信し、ローカルでは処理しない）。予測可能性を優先し、動作が検査可能。

**CC**：権限は「モデル審査＋ポリシーのフォールバック」（auto モードの分類器がセマンティック審査し、deny ルールがハードフォールバック）、effort は「段階＋いくつかの追加要素」（max は上限なし、ultracode はバックグラウンド編成を連動、ultrathink は単一ターンの一時的な深化）。柔軟性を優先し、コントロールサーフェスを厚くする。

二つの解法に優劣はなく、異なるシナリオに適している。動作の強い予測可能性と機械で検証可能なセキュリティ境界が必要なら、Codex の組み合わせマトリクスの方が直接的だ；人手による介入を減らし、セマンティック判断で複雑な境界を処理したいなら、CC の分類器ルートの方が楽だ。

---

## Harness Engineering シリーズ

coding agent の「プラットフォーム層（harness）と業務工学」について、一記事ずつ解説する：

1. [harness とは何か](./01-what-is-harness.md) —— プラットフォーム層と業務工学の境界
2. [複雑なタスクの spec の書き方](./02-how-to-write-specs.md) —— マルチ Agent・オーケストレーターエントリポイント・rules / docs / skills の整理
3. [harness の拡張方法](./03-extending-the-harness.md) —— skill・設定ディレクトリと hook：CC と Codex の二つの機構
4. [harness で agent を制御する](./04-permissions-and-effort.md) —— 権限と effort：CC と Codex のコントロールサーフェス
5. harness で長いタスクを乗り切る —— compact・memory・goal（執筆中）

> 他の言語：[English](../en/04-permissions-and-effort.md) · [한국어](../ko/04-permissions-and-effort.md) · [日本語](../ja/04-permissions-and-effort.md) · [中文](../zh/04-permissions-and-effort.md)
