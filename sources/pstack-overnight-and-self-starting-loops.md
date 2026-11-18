# pstack の手引きが書く、agent を一晩回す条件と、自分で始まる自動化（2026-11）

pstack の手引きの第7章は、agent に仕事を任せて席を離れる方法を4つの段で書く。1件の作業を一晩任せる段、複数の PR の列を任せる段、複数日の計画を任せる段、人が始めなくても起動する段である。第7章と、最後の段の実例である `automations/benny` の説明を、原文で照合した。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- ループを自律させる前に、4つの条件を満たす。手で一度やってある、agent が人と同じ道具を持つ、各段が成果を証明して止められる、繰り返す失敗を検査にしてある、の4つである。
- 任せる指示は、目標、終わりの条件、許可、止まる条件を持つ。時間の長さは終わりの条件にならない。
- 自分で始まる自動化は、各段が列を止められることと、各段が証拠を渡すことで信頼を保つ。実例の benny は draft PR までで、merge も deploy もしない。
- 効果の件数は書かれていない。playbook と skill の本文は読んでいない。
- 手元の [thkt/feedback-loop#187](https://github.com/thkt/feedback-loop/issues/187) は、同じ形を、テストと lint の改善に限って求める。

| 項目 | 内容 |
| --- | --- |
| 出典 | [cursor/plugins の pstack の手引き 第7章](https://github.com/cursor/plugins/blob/main/pstack/docs/guide/07-overnight.md)と、[automations/benny](https://github.com/cursor/plugins/tree/main/pstack/automations/benny) |
| 著者・発行元 | pstack は Cursor のプラグインのリポジトリにある。作者は poteto（Lauren Tan） |
| 公開日 | 第7章を最後に変更した commit は 2026-10-27（`c742dd6`）。benny を最後に変更した commit は 2026-11-15（`bf0c7cd`） |
| 照合した版 | `cursor/plugins` の main `bf0c7cd` |
| 照合日 | 2026-11-16 |
| 照合した範囲 | `pstack/docs/guide/07-overnight.md`（122行）、`pstack/automations/benny/README.md`（23行）、`pstack/automations/benny/FOR_AGENTS.md`（89行）の全文 |
| この資料を根拠にした Issue | [thkt/feedback-loop#187](https://github.com/thkt/feedback-loop/issues/187) は、この章を読んだ後に起票した。Issue の本文は、この資料を引いていない |

## 中心の主張

**自分の仕事を検証できる agent だけを、1人にしておける。** 章の冒頭は、任せることを安全にするものを3つ挙げる。確かめられる終わりの条件、隔離した worktree か cloud の agent、朝に監査する判断の記録である。信頼していないループは、確かめていない成果物を速く作るだけで、繰り返すたびに散らかりが増えると書く。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 自律させる前に、4つの条件を満たす | その作業を手で一度やったか、agent がやるのを見て、良い結果の形を知っている。agent が、人が使うのと同じ道具と信号（検証の skill、profiler、ログ）を持つ。各段が成果を証明し、基準に届かないときに列を止められる。記録をいくつか読み、繰り返す失敗を道具、skill、検査に変えてある。4つが揃うまでは、見ている間だけ回す | 第7章に記述がある。4つを満たしたかを判定する基準と、測定は書かれていない |
| 任せる指示は、目標、終わりの条件、許可、止まる条件を持つ | 例の指示は4行である。寝ることを告げると、agent は質問をやめて進める。commit の許可を先に答えておくと、agent は許可待ちで止まらない。数時間行き詰まったら止まって理由を書く、という逃げ道を置く | 第7章に記述がある。逃げ道が無いと、agent が8時間かけて目標を読み替えると書く。その件数は示されない |
| 終わりの条件は、合否が出る述語にする | 例は「古い呼び出しが0件、fixture がすべて通る、古い API を削除済み」である。「4時間作業する」は、agent が確かめるものを持たず、朝に残るのは結果ではなく4時間分の動きである | 第7章に記述がある。章の末尾は、これを落とし穴として挙げる |
| 1回の繰り返しは、変更1つ、確認1つ、記録1行にする | 終わりの条件を確かめる。根拠のある最小の変更をする。実物に対して検証する。進んだら commit し、進まなければ捨てる。判断を1行記録し、最初に戻る | 第7章の図と本文に記述がある |
| 行き詰まっても、終わりの条件を緩めない | 進みが止まったら、止めるのではなく方向を変える。終わりの条件を黙って緩めて、完了を宣言しない | 第7章に記述がある |
| 朝は、成果物ではなく判断を監査する | `/show-me-your-work` は、判断ごとに、時刻、段階、判断、理由、証拠の場所、結果を TSV に1行書く。要約を返す前に、別のモデル系列のレビュアーが記録を読む。返答の末尾は、人が見るべき点の一覧である | 第7章に記述がある。skill の本文は読んでいない |
| PR を書いた agent の判定で、merge しない | Autopilot-full は、独立した PR の列を merge まで運ぶ。PR ごとに持ち主の agent が1つ付く。別に起動した検証の agent 群が、持ち主のコードを確かめる。merge を許すのは、merge する patch に対する問題なしの判定だけである | 第7章に記述がある。playbook の本文は読んでいない |
| merge を人に残す形を選べる | Autopilot-stack は、Autopilot-full から merge を除いた形である。朝に、判定の付いた検証済みの PR が残り、人が読んで merge する | 第7章に記述がある |
| 複数日の計画は、コードを書かない調整役が持つ | Orchestrate は、1つの agent の寿命を超える計画に使う。調整役は、どの agent が何をするかを決め、成果を確かめ、自分ではコードを書かない。1つの agent が1回で終えられる作業は、一晩の指示に戻す | 第7章に記述がある。Cursor の Project が、調整役に続く thread と cloud の subagent を与えると書く |
| 人が始めない自動化は、各段が列を止め、証拠を渡す | 予定か出来事で起動する自動化は、人が始める手順を無くす。保守は、triage、再現、修正、検証の段に分かれる。triage は仕様どおりと判定でき、再現は再現できずに終われ、修正は危険と判断して止まれる。再現は壊れた状態の画面と動画を付け、修正は前後の証明を付ける | 第7章に記述がある。止まった割合と、届いた修正の件数は示されない |
| benny は、triage と、再現と修正の2つの自動化である | triage は、Slack の報告を読み、種類を分け、tracker で重複を探し、元の thread に判定を1回だけ返信する。再現と修正は、triage の印を待ち、誰かが修正を持っていれば止まり、実際の UI で症状を2回再現し、画面、動画、状態の読み取りを残す。その後に、範囲を限った根本原因の修正を1回試せる | benny の README と `FOR_AGENTS.md` に記述がある。3つの `SKILL.md` は読んでいない |
| benny は、足りないものがあれば止まり、draft PR で止まる | channel の座標、tracker への接続、操作用の adapter、feature map のどれかが無いか不確かなら、何もしない。draft PR は、前後の証明が通ったときだけ開く。merge と deploy はしない。subagent は Slack に投稿できない | `FOR_AGENTS.md` の shared rules に記述がある |

## 限界

- **効果の測定は無い。** 一晩の実行が終わりの条件に届いた割合、止まった割合、benny が開いた PR の件数は書かれていない。
- **Cursor の機能に依存する。** `/loop` は Cursor の組み込みの起動の仕組みで、pstack の skill ではないと章が書く。自分で始まる段は Cursor の Automations を、複数日の段は Cursor の Project と cloud の subagent を使う。Claude Code 向けの文書（`docs/harness/claude-code.md`）は読んでいない。
- **作者は当事者である。** pstack は Cursor のプラグインのリポジトリにあり、章は Cursor の機能の使い方として書かれている。
- **benny は、導入先ごとの構成を要する。** 入力は Slack の報告である。tracker、UI を操作する adapter、機能と画面の対応表を、導入先が用意する。
- **Autopilot-full は、自動で merge する。** merge を許す条件の細部は playbook にあり、読んでいない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた整理である。どの設定にも入れていない。

ループを3つに分けて読んだ。内側のループは、agent が編集し、検査を受け、直す繰り返しである。外側のループは、誰が作業を始め、誰が確かめるかである。改善のループは、訂正を検査に変える繰り返しである。内側と外側の呼び名は [all-code-is-slop-poteto-talk.md](all-code-is-slop-poteto-talk.md) の講演による。改善のループは照合者の呼び名である。第7章が扱うのは外側のループで、4つの条件の4つ目が改善のループにあたる。

| pstack の考え方 | 手元の資料 |
| --- | --- |
| 繰り返す失敗を検査に変えてから、自律させる | [all-code-is-slop-poteto-talk.md](all-code-is-slop-poteto-talk.md) の講演は、訂正のたびに lint か設計で止めると述べる。[magpie-agent-development.md](magpie-agent-development.md) の Magpie は、同じ失敗が別々の3日に見つかると規則に格上げする処理を、毎晩回す。feedback-loop#187 は、この変換を人が始めずに回す状態を求める |
| 終わりの条件は、合否が出る述語にする | [stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) は、完了の判定を文章からコマンドに移した手元の例である |
| 人が始めない自動化を、draft PR で止める | feedback-loop#187 は、変更する対象をテストと検査の設定に限り、draft PR で止める。benny は、製品コードの修正まで行う。未 merge の改善の PR がある間は次を作らない、という条件は #187 の側にあり、この資料には無い |
| 各段が証拠を渡す | feedback-loop#187 は、すり抜けた壊し方か、元になった訂正を PR に載せることを条件にする |
| PR を書いた agent の判定で、merge しない | [code-comments-for-agents.md](../knowledge/code-comments-for-agents.md) は、agent が書いた14件のテストのうち1件が、後でバグとして直された挙動を固定していたと記録する。テストを足す処理は自動にできても、採否は人が読む必要がある |
| 手で一度やってから、自律させる | [measure-tests-before-adding-practices.md](../knowledge/measure-tests-before-adding-practices.md) は、壊し方45種でテストを測り、すり抜けた12種を agent が埋めるところまでを、手で1回行った。訂正を lint の規則にする側は、手元で1回も行っていない。4つの条件の1つ目に照らすと、#187 の lint の側は、設計の前に手で1回行う段階にある |

## 照合の結果

- **照合した主張：** 「考え方と根拠」と「限界」の各行は、右の列に挙げた原文に記述がある。
- **動かしていないもの：** pstack、`/loop`、Cursor の Automations、benny。
- **読んでいないもの：** 章がリンクする playbook 5つ（Autonomous run、Pause safely、Autopilot-full、Autopilot-stack、Orchestrate）。`show-me-your-work`、`ketchup`、`figure-it-out` の `SKILL.md`。benny の3つの `SKILL.md`、`references/`、`templates/`。Cursor の Projects の紹介記事。手引きのほかの章。

## 照合の手順

1. 版を確かめる：`gh api 'repos/cursor/plugins/commits?path=pstack&per_page=1' --jq '.[0].sha'`
2. ファイルの一覧を取得する：`scout repo-tree cursor/plugins -p pstack`
3. 第7章を読む：`scout repo-read cursor/plugins pstack/docs/guide/07-overnight.md`
4. benny の説明を読む：`scout repo-read cursor/plugins pstack/automations/benny/README.md` と `scout repo-read cursor/plugins pstack/automations/benny/FOR_AGENTS.md`
5. ファイルごとの最後の変更日を確かめる：`gh api 'repos/cursor/plugins/commits?path=pstack/docs/guide/07-overnight.md&per_page=1' --jq '.[0].commit.committer.date'`

## 見直す条件

- 第7章か benny が変更されたとき。照合した版は `bf0c7cd` である。
- feedback-loop#187 の設計で、agent を動かす場所と起動のきっかけが決まったとき。
- 手元で、人が始めない自動化を1つ動かしたとき。
