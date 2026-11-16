# Lydia Hallie の講演「Rethinking Claude Code Best Practices」の考え方と根拠（2026-11）

Anthropic の Claude Code チームの Lydia Hallie が、AI Engineer のチャンネルで公開した講演を、文字起こしと Claude Code の公式ドキュメントで照合した。Claude Code の使い方を、人が担っていた作業をどこへ移すかという軸で整理する。自分たちが採用した判断ではなく、講演者は製品の当事者である。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Rethinking Claude Code Best Practices](https://www.youtube.com/watch?v=vlfCjWuv4es)（YouTube） |
| 著者・発行元 | Lydia Hallie（Anthropic）。チャンネルは AI Engineer |
| 公開日 | 取得できなかった。講演は Fable 5.2 の公開を「今週」と述べる |
| 照合日 | 2026-11-16 |
| 照合した範囲 | 講演の文字起こしの全文（00:00 から 17:02）と、動画の題名。公式ドキュメントの hooks、permissions、permission-modes、auto-mode-config、sandboxing、memory、best-practices の該当箇所。Anthropic の記事1本 |
| 照合した版 | ドキュメントは 2026-11-16 に取得した版。手元の Claude Code は 2.1.325 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**人が担っていた作業は無くなったのではなく、モデル、ハーネス、実行環境のどれかに移った。** 講演者は、以前のベストプラクティスを5つの作業に分け、それぞれの今の担い手を示す（[02:00](https://www.youtube.com/watch?v=vlfCjWuv4es&t=120s)、[16:34](https://www.youtube.com/watch?v=vlfCjWuv4es&t=994s)）。人に残るのは、最初のセッションを始めることと、出てきたものをレビューすることだと述べる。

| 以前、人がやっていたこと | 今の担い手 |
| --- | --- |
| 途中で軌道を直す | モデル |
| コマンドを承認する | auto mode |
| できることを制限する | deny rules と実行環境（サンドボックス、クラウドのセッション） |
| 出力を確かめる | テスト、hook、CI |
| セッションを始める | 統括役のセッション |

ループから出ること自体は、既定の設定で済むと述べる。難しいのは、人がいなくても安全で、成果物が合っていると確かめることだとする（[05:13](https://www.youtube.com/watch?v=vlfCjWuv4es&t=313s)）。

## 考え方と根拠

時刻は文字起こしの段落の開始位置で、リンクは動画のその位置を開く。右の列の「記述がある」は、公式ドキュメントにその内容があることを指す。

### 指示と承認

| 考え方 | 講演の記述 | 照合の結果 |
| --- | --- | --- |
| CLAUDE.md には、コードから見えないことだけを書く | 「IMPORTANT」や「never do this!」の強調は、今のモデルでは道を外す原因になり、ツール呼び出しが増える。スタックや探す場所の説明は要らない。コードから直接は参照されていない古い依存先のような事実は残す（[03:12](https://www.youtube.com/watch?v=vlfCjWuv4es&t=192s)） | 記述がある。[best-practices](https://code.claude.com/docs/en/best-practices) は、強調を重ねると Claude が規則を当てはめすぎると書き、曲げられない規則は hook か deny rule にするよう勧める。「never」は名指ししていない |
| 指示の点検にコマンドを使う | `/doctor prompt-audit` が、CLAUDE.md と skills から、今のモデルに要らない指示を探す | 記述がある。[memory](https://code.claude.com/docs/en/memory) によれば、v2.1.283 以降で使える。古いモデル向けの指示のほか、存在しないファイルへの参照と、ファイル同士の矛盾も探す。対象は CLAUDE.md、AGENTS.md、rules、skills、commands、subagents、output styles である。提案だけを出し、頼むまで書き換えない。手元では実行していない |
| 承認は auto mode に任せる | auto mode は、プロジェクト内の読み取りと編集以外に、別の分類器を走らせる。安全と分かるコマンドは通し、判断できないものは止める（[04:11](https://www.youtube.com/watch?v=vlfCjWuv4es&t=251s)） | 記述がある。[permission-modes](https://code.claude.com/docs/en/permission-modes) によれば、v2.1.283 以降、対話型の端末と VS Code のセッションは auto mode で始まる。`claude -p` と Agent SDK は、v2.1.302 以降は Manual で始まる。講演の「8月に既定にした」という時期は、ドキュメントに日付が無い |
| auto mode が信頼する範囲を、平文で設定する | 既定では、このフォルダとこの repo だけを信頼する。信頼する repo、bucket、domain と、常に許すこと、決してしないことを、設定に平文で書く（[04:48](https://www.youtube.com/watch?v=vlfCjWuv4es&t=288s)） | 記述がある。[auto-mode-config](https://code.claude.com/docs/en/auto-mode-config) は、既定で作業ディレクトリと、現在の repo に設定された remote だけを信頼すると書く。設定のキーは `autoMode.environment`、`autoMode.allow`、`autoMode.soft_deny`、`autoMode.hard_deny` である |

### 安全

| 考え方 | 講演の記述 | 照合の結果 |
| --- | --- | --- |
| モデルと分類器は、保証ではない | 危険なコマンドを、人は約14%しか止められず、auto mode は89%を止めた。約1割は通った（[05:35](https://www.youtube.com/watch?v=vlfCjWuv4es&t=335s)） | 未照合。講演が引く調査の一次資料を見つけられなかった |
| 保証が要るものは、deny rule にする | deny rule は、auto mode の分類器より先に評価される。当たった呼び出しは、分類器に届かない（[06:05](https://www.youtube.com/watch?v=vlfCjWuv4es&t=365s)） | 記述がある。[auto-mode-config](https://code.claude.com/docs/en/auto-mode-config) は、deny と明示の ask の規則が分類器より先に評価されると書く |
| 規則にするかは、取り消せるかで決める | 一度間違えたとき、取り消せるならモデルに任せる。間違ったコードは戻せる。未コミットの変更の破棄と、transcript に残る秘密の読み取りは、規則にする（[06:32](https://www.youtube.com/watch?v=vlfCjWuv4es&t=392s)） | 講演の基準である。ドキュメントとは照合していない |
| 権限は、実行中の副作用を縛れない | `npm test` は通すしかない。その中で動く依存のコードは、`.env` や SSH 鍵を読んで外へ送れる。権限が決めるのは、呼び出しが走るかどうかだけである（[07:13](https://www.youtube.com/watch?v=vlfCjWuv4es&t=433s)） | 記述がある。[permissions](https://code.claude.com/docs/en/permissions) は、Read と Edit の deny rule が、ファイルを自分で開く Python や Node のスクリプトには及ばないと書く |
| 副作用は、サンドボックスで縛る | 読めるファイル、書けるフォルダ、届くホストを OS が強制し、子プロセスにも及ぶ。サンドボックスを有効にすると、deny rule が OS に渡る。既定では端末のほとんどを読めるので、SSH 鍵には規則を足す（[08:18](https://www.youtube.com/watch?v=vlfCjWuv4es&t=498s)） | 記述がある。[sandboxing](https://code.claude.com/docs/en/sandboxing) は、OS が強制し、子プロセスも従うと書く。既定で読める範囲に `~/.ssh` と `~/.aws/credentials` を含めている。Read の deny rule がサンドボックスの設定に使われることは、設定元を除外する場合の説明の中で前提として書かれている |
| サンドボックスが覆うのは、シェルのコマンドだけである | Claude 自身のファイル操作、MCP サーバー、hook は、サンドボックスの外で動く（[09:10](https://www.youtube.com/watch?v=vlfCjWuv4es&t=550s)） | 記述がある。[sandboxing](https://code.claude.com/docs/en/sandboxing) の冒頭が同じ内容を書く |
| 長時間の作業は、クラウドのセッションで動かす | repo の新しい複製を持つ専用の環境で動く。`.env`、SSH 鍵、GitHub のトークンは最初から無い。API キーは、モデルに送らずに使える secret として渡す。ネットワークは、パッケージのレジストリと GitHub などに限られる（[09:10](https://www.youtube.com/watch?v=vlfCjWuv4es&t=550s)） | 照合を任せた agent の報告では、cloud-environments のページに記述がある。自分では原文を確かめていない |
| 環境は捨てられるが、外へ出たものは捨てられない | push した commit と、connector 経由で送ったメッセージは残る。クラウドでも、auto mode と、repo に commit した deny rules を使う（[10:16](https://www.youtube.com/watch?v=vlfCjWuv4es&t=616s)） | 講演の考え方である。照合していない |

### 検証と分担

| 考え方 | 講演の記述 | 照合の結果 |
| --- | --- | --- |
| 検証点を途中に置く | 同じ prompt でも、古い API を残す、新しいログインだけ移す、全員をログアウトさせる、のように結果が分かれる。合否で返るもの（テスト、ビルド、スクリーンショット）を途中に置き、人はその検証点をレビューする（[11:12](https://www.youtube.com/watch?v=vlfCjWuv4es&t=672s)） | 講演の考え方である |
| prompt での依頼は、保証ではない | 完了の定義を書き、タスクに分けてテストを持たせ、実際の出力を見せるよう頼む。それでもモデルがやる保証は無い（[12:00](https://www.youtube.com/watch?v=vlfCjWuv4es&t=720s)） | 講演の考え方である |
| 必ず走らせる検査は、hook にする | ファイルを編集するたびに lint を走らせる。タスクを完了にするときに、変えた箇所のテストを走らせる。セッションを終えるときに、全テストを走らせる（[12:24](https://www.youtube.com/watch?v=vlfCjWuv4es&t=744s)） | 記述がある。[hooks](https://code.claude.com/docs/en/hooks) の表は、`TaskCompleted` で exit 2 を返すとタスクが完了にならず、`Stop` で返すと Claude が止まらずに続けると書く |
| hook は exit 2 で終える | テストが失敗すると、多くのツールは exit 1 で終わる。exit 1 では、タスクは完了になる。Claude を直しに戻すのは exit 2 である（[12:24](https://www.youtube.com/watch?v=vlfCjWuv4es&t=744s)） | 記述がある。ただし、講演が述べない例外もある。[hooks](https://code.claude.com/docs/en/hooks) は、`Stop`、`SubagentStop`、`TaskCompleted` で、hook が標準出力に何も出さずに exit 2 で終わり、標準エラーが `No such file or directory` のようにファイルの欠落を示すと、差し止めない失敗として扱うと書く |
| CI は、どのセッションにも同じ検査をかける | セッションは最後に PR を開く。CI の auto-fix を有効にすると、Claude が失敗した検査を見て直しに戻る。一部の PR は、検査が通ったら GitHub に merge させる（[13:53](https://www.youtube.com/watch?v=vlfCjWuv4es&t=833s)） | 照合を任せた agent の報告では、auto-fix は Claude Code on the web のページに記述がある。検査が通ったら merge する設定は、確かめたページには無かった。講演は、merge するのは GitHub だと述べる |
| 人のレビューの量は、リスクで決める | 全 PR を、書いたセッションとは別の Claude が先にレビューし、別の agent の組が指摘を確かめ直す。文書や文言は人なし、通常の変更は1人、リスクの高い変更は2人である（[14:26](https://www.youtube.com/watch?v=vlfCjWuv4es&t=866s)） | 人数の基準は、講演者のチームの運用である。照合を任せた agent の報告では、code-review のページに、複数の agent が読み、報告の前に指摘を確かめる段があると書かれている |
| 統括役のセッションに、開始と確認を任せる | 統括役がタスクを分け、タスクごとにセッションを始め、戻ってきたものを確かめる。統括役には判断の得意なモデルを、作業のセッションには実装の得意なモデルを、別々に指定する。文脈の管理は自動になる（[15:10](https://www.youtube.com/watch?v=vlfCjWuv4es&t=910s)） | 照合を任せた agent の報告では、Projects は公開ベータで、統括役と作業のセッションのモデルを別々に指定できる。自分では原文を確かめていない |
| 試すなら、普段より大きい仕事を1つ渡す | CI が気にする点を拾うことを確かめ、環境のネットワーク、secret、準備のスクリプトを設定し、クラウドで始める。セッションが止まる箇所と、人がまだ入っている箇所を、次に任せる対象にする（[17:02](https://www.youtube.com/watch?v=vlfCjWuv4es&t=1022s)） | 講演の勧めである |

## 講演が挙げる数字

| 項目 | 講演の数字 | 照合の結果 |
| --- | --- | --- |
| Anthropic の1日の変更 | 5月の約2,500件から、9月の3万件超（[01:43](https://www.youtube.com/watch?v=vlfCjWuv4es&t=103s)） | [Redesigning source control at Anthropic](https://www.anthropic.com/engineering/redesigning-source-control)（2026-10-13）に、merge された変更の数として同じ数字がある |
| 危険なコマンドを止めた割合 | 人は約14%、auto mode は89% | 未照合。一次資料を見つけられなかった |

## 限界

- **講演者は当事者である。** 結びは、クラウドのセッションと Projects の勧めである。
- **例は Anthropic の社内の運用である。** 数十のセッションを同時に動かす運用の費用と、失敗した例は示されない。
- **exit 2 の勧めは、例外に触れない。** ドキュメントが書く例外は、失敗したテストの出力で起こりうる。
- **プランと版で、使える機能が違う。** 照合を任せた agent の報告では、Projects は Pro、Max、Ultra のベータで、code review は Team と Enterprise に限られる。
- **照合の一部は、任せた agent の報告による。** クラウドのセッション、auto-fix、code review、Projects の行は、自分では原文を確かめていない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせた結果である。どの設定にも入れていない。

| 講演の考え方 | 手元の資料 |
| --- | --- |
| hook は exit 2 で終える | [stop-hook-return-channels.md](../knowledge/stop-hook-return-channels.md) は、exit 2 の標準エラーに `No such file or directory` があると、Claude Code 2.1.325 が差し止めずに止まることを測った。今回、同じ挙動がドキュメントに書かれていることを確かめた。両方の CLI に登録するスクリプトは exit 0 で `decision: "block"` を返す、という手元の結論は変わらない |
| 必ず走らせる検査は、hook にする | [stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) は、セッションの終了時に完了判定のコマンドを走らせる形を確かめた。講演が挙げる、編集のたびの lint と、タスク完了時のテストは、手元では試していない |
| CLAUDE.md には、コードから見えないことだけを書く | [code-comments-for-agents.md](../knowledge/code-comments-for-agents.md) は、agent がコードから導ける理由をコメント無しで導くことを測った。残す価値があるのは外の事実、という線が、CLAUDE.md とコメントで同じになる |
| 規則にするかは、取り消せるかで決める | [sensible-defaults.md](sensible-defaults.md) に引いた Osmani の記事は、強調しても守られない行を hook に移すと書く。講演は、守られたかではなく、間違えたときに取り消せるかで分ける |
| 副作用は、実行環境で縛る | [agent-in-apple-container.md](../knowledge/agent-in-apple-container.md) は、container の中で agent を動かし、ホストで触れられる範囲をマウントした worktree に限った |
| 検証点をレビューする | [all-code-is-slop-poteto-talk.md](all-code-is-slop-poteto-talk.md) の講演は、コードレビューが検証の成果物を見る作業に縮んだと述べる。この講演は、そこに権限と実行環境の層を足す |

## 照合の結果

- **照合した主張：** 「記述がある」と書いた行は、右の列のページで原文を確かめた。確かめたページは hooks、permissions、permission-modes、auto-mode-config、sandboxing、memory、best-practices と、Anthropic の記事1本である。
- **agent の報告だけによる行：** クラウドのセッション、auto-fix、code review、Projects。
- **未照合：** 危険なコマンドを止めた割合の調査。「取り消せるか」の基準。
- **動かしていないもの：** `/doctor prompt-audit`、`autoMode` の設定、クラウドのセッション、Projects、`TaskCompleted` の hook。
- **照合していないもの：** 文字起こしと音声の一致。文字起こしの作成方法は分からない。動画の公開日と説明欄は取得できなかった。

## 照合の手順

1. 動画の題名とチャンネルを取得する：`https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=vlfCjWuv4es&format=json` を開く。
2. ドキュメントの一覧を取得する：`scout fetch --raw https://code.claude.com/docs/llms.txt`
3. 該当するページを取得する：`scout fetch --raw https://code.claude.com/docs/en/<ページ名>.md`。ページ名は hooks、permissions、permission-modes、auto-mode-config、sandboxing、memory、best-practices である。
4. 取得したページで、講演の主張にあたる文を検索して読む。hooks は「Exit code output」の節を読む。
5. Anthropic の記事を取得する：`scout fetch https://www.anthropic.com/engineering/redesigning-source-control`

## 見直す条件

- Claude Code の hooks、permissions、sandboxing のドキュメントが変わったとき。
- 講演が引く、人と auto mode を比べた調査の一次資料を見つけたとき。
- 手元で `/doctor prompt-audit` を実行したとき、または `TaskCompleted` の hook を試したとき。
- 動画の音声と文字起こしを突き合わせたとき。
