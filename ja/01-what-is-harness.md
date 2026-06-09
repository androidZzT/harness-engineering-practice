# harness とは何か

## 一、harness の定義

### 業界における主要な定義

「Harness Engineering」という言葉が2026年に広まったが、「harness」自体はますます曖昧に使われるようになっている。まず一次資料をいくつか示し、そのうえで私の理解を述べる。

**OpenAI** は [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)（2026年2月、Ryan Lopopolo）で harness をこう定義している：

> "the full environment of scaffolding, constraints, and feedback loops that surrounds the agent."

OpenAI のこの定義では、Codex harness とは `codex-core` という Rust 共有ライブラリそのもの——agent loop、thread lifecycle、config / auth、sandboxed tool execution をすべて含む——を指す。UI シェル（VS Code プラグイン、CLI）は harness には含まれない。

**Anthropic** は [Building agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk) でこう述べている：

> "the agent harness that powers Claude Code (the Claude Code SDK) can power many other types of agents, too."

2025年11月の [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) では、harness を独立した工学的対象として論じており、context compaction・structured handoff・tool sandboxing を harness の重要設計要素として挙げている。

**Simon Willison** は [How coding agents work](https://simonwillison.net/guides/agentic-engineering-patterns/how-coding-agents-work/) で最も広い使い方をしている：

> "A coding agent is a piece of software that acts as a harness for an LLM, extending that LLM with additional capabilities that are powered by invisible prompts and implemented as callable tools."

彼は Claude Code・Cursor・Codex CLI を UI シェルも含めてすべて harness と捉えている。

**LangChain** の [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness) はこう定式化している：

> "Agent = Model + Harness."

モデルはエンジンであり、harness はそのエンジンを継続的に動かす乗り物に仕立てるものだ。

これらの定義の**共通点**は明確だ：harness とはモデル外側の実行時機構——scaffolding・制約・フィードバックループ・ツール呼び出し・コンテキストとメモリの管理・agent loop——である。**違い**は範囲にある：OpenAI が最も狭く（SDK / core のみ）、Simon が最も広く（UI surface を含む）、Anthropic はその中間に位置する。

### 私の理解

本稿では OpenAI 寄りの狭義の定義を採用し、次の一文で定義する：

> **harness は coding agent のプラットフォーム層——コンテキスト管理・メモリ・subagent オーケストレーション・skill 機構・ツール呼び出し・hook・実行ループ——である。業務工学はその上に構築するものであり、harness 自体を変更すべきではない。**

なぜ狭義を採るのか。この記事で議論したい「何が harness で何が業務工学か」という境界の問題は、狭義でなければ明確に語れないからだ。CLI 全体を harness とすれば、業務工学の境界が CLI の外側に出てしまい議論の余地がなくなる。逆に harness を LLM 呼び出しインターフェースだけに絞ると、context / memory / tool がすべて業務工学側に落ち、工学規律を業務側に全部押し付けることになりこれも現実的ではない。**coding agent の SDK / core 層**が折り合いのつく中間点だ。

### SDD ≠ Harness Engineering、Spec ≠ Harness

この二組の概念はよく混同されるが、位置付けはまったく異なる。

**SDD（Spec-Driven Development）** は OpenAI の Sean Grove が [The New Code](https://www.darekm101.com/articles/the-new-code-sean-grove-openai) などで広めたもので、**業務側の工学パラダイム**に焦点を当てる：直接コードを書くのではなく spec を人と AI の協働の主成果物とする——要件 spec・設計 spec・受け入れ spec を一連の流れとし、コードは spec の派生物とみなす。SDD が問うのは **what to build** である。

**Harness Engineering** は OpenAI の Ryan Lopopolo が 2026年2月のブログ記事で正式に命名したもので、**agent プラットフォーム層の工学パラダイム**に焦点を当てる：context compaction をどう設計するか、agent loop をどう管理するか、sandbox をどう構成するか、長いセッションでどう安定を保つか。Harness Engineering が問うのは **how the agent runs** である。

両者の関係：**SDD は上層パラダイム、Harness Engineering は下層パラダイム**だ。SDD は「業務成果物としてどんな spec を書くか」を問い、Harness Engineering は「coding agent がそれらの spec を安定して実行するにはどうあるべきか」を問う。harness がなければ spec は動かず、spec がなければ harness がどれほど安定していても空回りするだけだ。

**Spec と harness も別物である**。Spec は業務工学の産物——ワークフロー主幹・フェーズ契約・原子ツール・ドメイン知識——これらが spec の異なる形態だ。harness は spec を動かすためのプラットフォームである。両者の関係は「業務の spec が harness の能力を呼び出す」であり、**「spec は harness の一部」でも「harness は spec の一種」でもない**。

この二層を混同すると、「何が業務の仕事で何がプラットフォームの仕事か」が永遠に語れなくなる。

## 二、harness は coding agent のプラットフォーム層

私が現在使っているのは Claude Code と Codex だ。工学的観点では両者の違いは大きくない：ローカル CLI エントリポイント + リモートモデル + モデルが制御可能にタスクを完了するための実行時機構。この実行時機構が harness だ。

以下の 7 項目は Claude Code / Claude Agent SDK を主要参照系として整理したものだ。「skill」と「hook」は Claude エコシステムの命名であり、Codex 側の対応概念は「permissions / sandbox / agent loop hooks」と呼ばれるが本質は同じで、名称が異なるだけだ。この 7 項目は公式の分類ではなく、実際のオーケストレーション経験から整理したものであり、遭遇する agent の挙動の大半を説明できる。

**1. コンテキストウィンドウ管理。** どの情報を現在の会話ウィンドウに入れるか、いつ圧縮するか、いつ切り捨てるか、いつ過去のコンテキストを復元するかを決める。あるランが複数フェーズにまたがる場合、harness は古くなった詳細を退け、現フェーズに必要な事実を残す。モデル自身は現在のウィンドウしか見えず、背後のトレードオフは見えない。

**2. メモリ管理。** ランをまたぐ永続情報：プロジェクトルール・ユーザー設定・再現可能な証跡。`CLAUDE.md`・`AGENTS.md`・`.inbox`・`run state` はすべてメモリインフラに属する。harness がそれらをいつウィンドウに注入するかを決め、業務工学はそれらに何を書くかを決める。

**3. subagent のスポーン。** エントリ agent は具体的な作業を自分でやらず、タスクを独立したコンテキストを持つ subagent に分割し、自身はオーケストレーション・証跡の集約・クローズだけを行う。重要な点として：subagent は harness が提供するプリミティブであり、業務工学は「派遣する」ことだけができ、派遣機構を再発明すべきではない。

**4. Skill 機構。** Claude Code の skill と Codex の plugin / preset は本質的に同じもの——トリガー条件・入力制約・出力契約・リスクマークを持つロード可能な振る舞いユニット——だ。harness は skill がどのように発見・ロード・組み合わされるかを決め、業務工学は各 skill に何を書くかを決める。

**5. ツール呼び出し。** `git`・`worktree`・シェル・ビルドコマンド・テストコマンド・MCP サーバー・サードパーティ SaaS——すべての副作用はツール呼び出しを通じて発生する。harness はプロトコル層（登録方法・スケジューリング・タイムアウト処理・出力の切り捨て）を担当し、業務工学は「どのツールを登録するか」を担当する。

**6. Hook 機構。** `SessionStart`・`Stop`・`PreCompact`・`PostCompact` というフックは、harness が業務側に「ライフサイクルの重要なタイミングに割り込む」口を提供するものだ。`.inbox` への注入、終了前のワークフロークローズチェック、圧縮前の重要証跡の保全——これらはすべて hook に頼る。

**7. 実行ループ。** 以上の 6 項目をつなぐメタ能力：状態読み取り → フェーズ判断 → subagent 派遣 → 証跡集約 → 状態同期 → 継続か停止かの判断。成熟した harness はこのループが常に収束することを保証し、「半分やって黙って立ち去る」ことを防ぐ。

7 項目に共通するのは：**これらはすべて coding agent チーム（Anthropic / OpenAI）が継続的に磨いていることであり、業務を開発する者が再実装すべきことではない**、という点だ。プロジェクトはこれらの能力をどう「使うか」を決めることはできるが、それらを迂回して自前の実装を持つべきではない。

## 三、業務工学とは何か：harness の上に構築された一組の spec

第一節の議論に立ち返ると：業務工学の産物は本質的に一組の spec——実行可能で、agent に派遣できる工学契約——だ。spec を動かすのは harness であり、harness が何を動かすかは spec によって定まる。

この spec が仕様書リポジトリの中でどう階層化され、どう整理され、ファイルをどう配置するかは別稿で詳しく論じる価値がある。本節では抽象的に一点のみ明確にしたい：完全な業務 spec には通常どんな種類の内容が必要か、言い換えれば次のような問いに答えなければならないか。

**今回の成果物はどこから始まり、どのフェーズを経て、いつ終わるのか？** これがワークフロー主幹だ。それ自体は「考える」のではなく、宣言するだけだ：現在どのフェーズにあるか、次に誰を派遣すべきか、どの条件で停止するか。プロジェクト全体で主幹 spec は通常一つあれば十分だ。それはエントリオーケストレーターだが、agent そのものではない——実際に「考える」のは harness が提供する subagent だ。

**各フェーズは何をし、何を出力し、いつ通過するのか？** これがフェーズ契約だ。各フェーズには明示的な入力・出力・通過条件・ブロック規則がある。フェーズ間は宣言的な接続であり、互いに先走ることはない——これが SDD と従来の agent loop の最大の工学的規律の違いだ。何フェーズに切るか、どこで切るかはプロジェクトによって異なる。「要件分析 / 設計 / 実装 / レビュー / テスト / リリース」は一般的な骨格だが、唯一の組み合わせではない。

**フェーズが作業する際にどんな原子能力を使うか？** これがツールボックスだ：リポジトリスキャン・証跡抽出・ビルドコマンド・E2E テスト実行・ビジュアル diff・タスクシステム連携……各原子能力は自身の入出力にだけ責任を持ち、**どのフェーズから呼ばれているかを知るべきではない**——そうであってはじめて複数フェーズから再利用でき、特定フェーズに縛られない。

**spec が判断を下す際に何を背景として頼るのか？** これがドメイン知識だ：業務アーキテクチャ・踏み抜き禁止事項・参考実装。この種の内容は直接 invoke されないが、他のすべての spec が「これは良い設計か」「これは落とし穴か」「参照実装はどんな形か」を判断する際に暗黙的に依存している。

これら 4 種類の内容が合わさって「業務工学はプロジェクトの中で何を書くべきか」を定義する。共通の特徴がある：

- **すべて業務自身が書くもの**——プロジェクトリポジトリの一部であり、プロジェクトの進化とともに変わる。
- **すべて harness が公開するプリミティブを通じて動く**——業務側は agent loop・コンテキスト管理・subagent スケジューリングを自分で実装せず、これらはすべて harness を呼び出す。
- **互いに参照関係で組織される**——主幹がフェーズを参照し、フェーズがツールを参照し、すべてがドメイン知識を参照する。

ここまで書けば明確に見えてくる：**業務工学 = この spec をうまく書くこと；harness = この spec を動かす実行時環境。** 両者の関係は「呼び出すと呼び出される」であり、「変更すると変更される」ではない。

## 四、なぜ harness を変更してはいけないのか

いくつかのプロジェクトで、問題に直面した際の最初の反応が「Claude Code の ×× 機構を少し変えられないか」というものを見てきた：自分でコンテキスト圧縮を引き受ける、独自の subagent スケジューリングを作る、hook を迂回した外部インジェクターを書く。これらの発想はすべて素朴な工学的直感から来ている：より具体的なビジネス要件があり、プラットフォームのデフォルト動作が不十分だ、と。

しかし harness を迂回することはほぼ常に割に合わない。理由は三つある。

**第一に、harness 自体は常に進化している。** Claude Code がこの一年でどれほど変わったか自分でわかるだろう：コンテキスト圧縮戦略・skill graph・subagent アイソレーション・hook の呼び出しタイミング・ツール結果のスリム化……毎回のアップグレードは無料で享受できる、ただしパッチを当てていない限りで。一度パッチを当てると、毎回のアップグレードで互換性を再評価しなければならず、3 秒で済んだアップグレードが 3 時間のデバッグになる。

**第二に、harness の複雑さは直感をはるかに超えている。** コンテキスト管理は一見単純に見えるが、実際にはウィンドウバジェット・キャッシュヒット・圧縮アルゴリズム・情報忠実度・ランをまたぐ一貫性が絡み合っている。「自分で簡略版を書く」と 200 行で済むように聞こえても、動かしてみればエッジケースが次々と現れる。これはプラットフォーム層とアプリケーション層の永遠の非対称——見えているのはプラットフォームのデフォルト動作だが、見えていないのはそれが回避してきた何百ものエッジケースだ。

今日まさにこんな例を見た。あるプロジェクトで独自の「メモリ機構」を実装していた：リポジトリに `memory/` ディレクトリを維持し、agent セッションが起動するたびに hook で `memory/` 内のすべてのファイルを全量コンテキストに注入する。表面上は「自前のメモリシステム」だが、何回か動かすとコンテキストウィンドウが溢れ、途中で切り捨てられ、重要な情報が押し出された。

これが典型的な「harness への不正介入」だ。

メモリ管理はそもそも harness の 7 つの能力のひとつだ。Anthropic の [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) は論文全体をかけてこう述べている：進捗ファイルの構造化引き継ぎ・compaction 戦略・必要情報の保全と冗長の除去、核心目標はウィンドウを溢れさせないことだ。Claude Code も `CLAUDE.md`・`.inbox`・run state というプリミティブを提供しており、注入には取捨選択がある：何をウィンドウに入れ、何を圧縮し、何を切り捨て、何を外部ストレージに遅延ロードするか、すべてにルールがある。

`memory/` の全量注入はこれらすべての取捨選択を飛ばしており、プラットフォームチームが繰り返し磨いてきた能力を、「ウィンドウの大きさに関係なく全部突っ込む」という一行の hook に簡略化したのに等しい。その結果、harness がすでに処理してくれていたウィンドウバジェット問題を自分に引き戻し、しかもプラットフォーム層の圧縮・復元機構という安全網もない。

同じことを harness のプリミティブでやれば（`CLAUDE.md` にプロジェクトルールを書く・hook で適切なタイミングに `.inbox` を注入する・run state でフェーズをまたいで必要な事実を同期する）、コスト数行の設定で済む。自分で再実装するとコンテキストウィンドウ溢れ・プラットフォームアップグレードの恩恵を失う・保守負担の増大というコストになる。専門的なことは専門家に任せる。業務は harness の機構を変えるべきではない——これは説教ではなく、工学経済学だ。

**第三に、harness を迂回すると SDD の軌道から外れてしまう。** SDD の工学規律は harness が公開するいくつかのハード制約から来ている：subagent はコンテキストをまたいで情報を盗み見できない、フェーズは gate を跨いで先走れない、ツール呼び出しは証跡を残す必要がある。ある業務要件のために harness を迂回した瞬間、これらの制約はすべて崩れる。短期的には一つの制限を回避できるが、長期的には工学規律全体の土台を壊すことになる。

正しいアプローチは：**プラットフォーム層が不十分な箇所に直面したとき、まず業務 spec 側で吸収できないか考えること。**

もうひとつ、ポジティブな例を挙げる。自分で harness の実践を組み上げていたとき、こんなシナリオに直面した：モデルがフェーズをまたぐ際にときどき「一歩余分に歩きたがる」——要件の証跡だけを出すべき場面でモデルがついでに設計に入ろうとする。最初の発想は「subagent スケジューリングにパッチを当てて現フェーズだけやらせられないか」だった。しかし後で気づいた：これは業務 spec 側でも解決できる。フェーズ契約に現フェーズの境界をきちんと書き込み、gate チェックを追加し、subagent が収束すべきタイミングで hook で強制収束させる。harness に触れずに、振る舞いを制約できた。

「業務工学でプラットフォームの限界を吸収する」能力は、SDD エンジニアが最も鍛えるべき筋肉だ。

## 五、一文を覚えておく

> **業務工学の仕事 = この spec をうまく書き、harness が公開するプリミティブを組み合わせること；harness 自体は変更しない。**

この一文には二つの意味がある。

上を向いて：harness が与えるプリミティブを使い切る。コンテキスト管理・subagent・skill・hook・ツール呼び出し、これらは無料の「工学能力の外付け」であり、プラットフォームチームはあなたよりそれらをどう実装すべきか知っている。

下を向いて：業務成果物の差異化は、spec の書き方・組み合わせ方に現れるものであり、harness をどれだけ深く変えたかではない。同じ種類のプロジェクトに取り組む 2 チームがあっても、spec の形は異なる——差異はそれぞれのフェーズの切り分け・原子ツールの選択・ドメイン知識の蓄積にあり、どちらかが Claude Code のカーネルを独自改造した点にはない。

これが私が Harness Engineering に取り組んできて最も強く感じる直感だ：**克制は工学規律の核心である。** 変えるべきでないものを変えない。浮いた力はすべきことに全力投球する：業務側の spec をうまく・正確に書く。

次稿では 2 つの関連する問いを展開する：**spec には具体的に何を書くべきか？skill はどう階層化すべきか？** ワークフロー主幹・フェーズ契約・原子ツール・ドメイン知識それぞれをどこに置き、互いにどう参照するかを、具体的なプロジェクトに落とし込んだ実行可能な構造として示す。

---

## Harness Engineering シリーズ

coding agent の「プラットフォーム層（harness）と業務工学」について、一記事ずつ解説する：

1. [harness とは何か](./01-what-is-harness.md) —— プラットフォーム層と業務工学の境界
2. [複雑なタスクの spec の書き方](./02-how-to-write-specs.md) —— マルチ Agent・オーケストレーターエントリポイント・rules / docs / skills の整理
3. [harness の拡張方法](./03-extending-the-harness.md) —— skill・設定ディレクトリと hook：CC と Codex の二つの機構
4. [harness で agent を制御する](./04-permissions-and-effort.md) —— 権限と effort：CC と Codex のコントロールサーフェス
5. harness で長いタスクを乗り切る —— compact・memory・goal（執筆中）

> 他の言語：[English](../en/01-what-is-harness.md) · [한국어](../ko/01-what-is-harness.md) · [日本語](../ja/01-what-is-harness.md) · [中文](../zh/01-what-is-harness.md)
