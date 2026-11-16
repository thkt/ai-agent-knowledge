# poteto の講演「All Code Is Slop (Except Mine)」の考え方と根拠（2026-11）

Lauren Tan（poteto）が AI Engineer のチャンネルで公開した17分の講演を、文字起こしと動画の説明欄で照合した。agent に書かせるコードの品質を、コードベースと検査の側で保つ方法を述べる。自分たちが採用した判断ではなく、講演が挙げる数字は講演者の自己申告である。

| 項目 | 内容 |
| --- | --- |
| 出典 | [All Code Is Slop (Except Mine)](https://www.youtube.com/watch?v=O_zcc6Xl12g)（YouTube） |
| 著者・発行元 | Lauren Tan（poteto）。チャンネルは AI Engineer |
| 公開日 | 2026-11-14。講演が行われた日は確かめていない |
| 長さ | 17分31秒 |
| 照合日 | 2026-11-16 |
| 照合した範囲 | 講演の文字起こしの全文（00:12 から 16:54）と、動画の題名、説明欄、章の一覧 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**agent を信頼できるのは、agent が働く環境に投資したときだけである。** 講演者は、ソフトウェアエンジニアの仕事を「生産量を大きく増やしながら、コードの品質を保つか上げる方法を見つけること」と定義する（[00:42](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=42s)）。数千件の PR を出せるようになるまでに6か月かかったと述べる（[01:22](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=82s)）。

投資の配分は、逆ピラミッドの4層で示す（[02:02](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=122s)）。時間をかける量が多い順に並び、上の2層は決定論的である。

| 層 | 中身 |
| --- | --- |
| 1. コードそのもの | API の設計、アーキテクチャ、リファクタリング |
| 2. 静的な検査 | lint、型、コンパイラ、各種のテスト |
| 3. 検証 | 実行時の検証と形式検証。不変条件を確かめ、agent が意図を理解したかも確かめる |
| 4. 文章 | rules、skills、AGENTS.md。判断が要るものだけを少量 |

## 考え方と根拠

時刻は文字起こしの段落の開始位置で、リンクは動画のその位置を開く。右の列の「記述がある」は、文字起こしにその発言があることを指す。

| 考え方 | 講演の記述 | 講演が挙げる根拠と照合の結果 |
| --- | --- | --- |
| 訂正は、rules の行ではなく決定論的な検査にする | agent を訂正するたびに、AGENTS.md に1行足す代わりに、検査にするか、API の設計とアーキテクチャでその問題を種類ごと無くす（[02:58](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=178s)、[09:10](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=550s)） | 記述がある。根拠は「コードにあるほうが再現できる」という説明で、測定は無い。繰り返すとレビューで見る項目が減り、agent が自分の PR の大半を merge できるようになると述べる |
| コードベースは agent への最大の入力である | 回避策、アンチパターン、重複、技術的負債は、確立したパターンに見えるので、次の agent が写す（[04:24](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=264s)） | 記述がある。コードベースを庭にたとえる。2024年は同僚の PR を3件レビューし、今は300件ほどだと述べる（[03:26](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=206s)） |
| 使い分けに細かい判断が要る API は禁止する | React の `useEffect` を製品コードで禁止し、自作のフレームワーク Dune の API の内側だけで許可した（[04:57](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=297s)） | 記述がある。この問題はレビューで拾う必要が無くなったと述べる。Dune は Electron アプリ向けで、公開されているかは確かめていない |
| 制約をディレクトリ構造で伝える | 最上位に `electron-renderer` と `electron-main` を置き、相互の import を依存グラフの解析と lint で禁止した（[06:10](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=370s)） | 記述がある。きっかけは、Cursor で agent が main thread のコードを renderer に import し、long task と描画の引っかかりを起こしたことだと述べる |
| agent は文章を、正しいことをしない理由の置き場にする | 文書、コメント、rules、skills に、正しい対応をしない理由を書く。小さな訂正を、二度と破れない規則として扱いすぎることもある（[06:55](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=415s)） | 記述がある。例は Cursor での1件である。agent が `setTimeout`、`requestAnimationFrame`、`requestIdleCallback` を応急処置に使い、正当化するコメントを付け、タイマーが広がって性能が落ちた（[07:22](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=442s)）。原因は VS Code の命令的な書き方と React の宣言的な書き方の衝突だと説明する |
| コメントの99%を禁止し、lint のエラーメッセージに置き換える | コメントは古くなり、人は読まず、強制できない。代わりに、根本原因を直すよう agent に伝えるエラーメッセージを持つ lint ルールを置いた（[08:19](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=499s)） | 記述がある。根拠は上の行のタイマーの経験である。禁止の前後を比べた数字は無い。残す1%が何かは述べていない |
| コードレビューは、検証の成果物を見る作業になる | レビューの対象は、動画、スクリーンショット、図、形式検証の結果に縮んだ（[09:39](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=579s)） | 記述がある |
| 実行時の検証に、機能の一覧と専用の CLI を持たせる | agent が自分でコードを動かし、Chrome DevTools Protocol などの道具を使う。skill に「feature map」（利用者から見た機能の一覧）を置き、指示の語彙を揃える。専用の CLI を使い回してトークンを減らす（[09:39](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=579s)） | 記述がある。feature map と CLI の実物は示されない |
| 形式検証は、層の1つとして実験中である | TLA+ で、複数の端末にまたがる通知などの主要なフローをモデル検査する。Lean で、自作の仮想化ライブラリにちらつきが無いことを確かめる（[10:49](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=649s)） | 記述がある。バグを「かなりの数」見つけたと述べるが、件数は無い。銀の弾丸ではないとも述べる。説明欄も TLA+ と Lean を挙げる |
| UI にも property-based testing を使う | Bombadil というライブラリで、UI のバグを数件見つけた（[11:34](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=694s)） | 記述がある。Bombadil は確かめていない |
| トークンの大半を検証に使う | 講演者のトークンの大半は検証に使われる。他者の投稿の「1日のトークンの70%が検証」を引く。検証の手法は新しくなく、agent が書く費用と保守の費用を下げたと述べる（[11:51](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=711s)） | 記述がある。引かれた投稿は読んでいない。講演者自身の割合は数字で示されない |
| 状態機械の図をレビューの材料にする | pstack の `/statechart` で、状態機械を利用者の視点で図にし、動画やスクリーンショットと並べて読む（[12:54](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=774s)） | 記述がある。skill の本文は [pstack-formal-verification.md](pstack-formal-verification.md) で照合した |
| 自動化の前に、コードベースを直す | 訂正のたびに、まず lint ルールで新しい発生を止め、古いパターンを片付け、検証を揃える。内側のループが slop を出すなら、自動化は slop を増やし、トークンの消費も増える（[15:10](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=910s)） | 記述がある。説明欄の章の題も「Foundations before software factories」である |
| 土台の上に、外側のループを作る | リリースのたびに、アプリ全体を利用者のように操作して fuzz するプロジェクトが起動する。見つけたバグごとに修正の PR を開き、自動で merge する（[16:06](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=966s)） | 記述がある。自動で merge された PR の件数と、その revert の割合は示されない |

## 講演が挙げる数字

数字はどれも講演者の自己申告で、測り方は示されない。

| 項目 | 講演の数字 | 動画の説明欄 |
| --- | --- | --- |
| 講演者が1か月に本番へ出した PR | 3,000件（[00:42](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=42s)） | 記載なし |
| チームが1か月に本番へ出した PR | 11,000件（[13:22](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=802s)） | 同じ数字がある |
| stable 版のリリース | 約1日に1回 | 記載なし |
| 10月に1件以上の PR を出した人 | 約227人 | 記載なし |
| revert された PR の割合 | 約0.5%で安定。割合は以前より下がった（[14:14](https://www.youtube.com/watch?v=O_zcc6Xl12g&t=854s)） | 約0.5%とある |
| 性能 | 起動、入力、新しいチャットを開く操作、チャットの切り替えが、公開時より速くなった | 記載なし |

## 限界

- **品質の指標は2つだけである。** 講演が品質の裏付けに挙げるのは、revert の割合と性能である。revert されずに残った欠陥は、どちらにも現れない。
- **測り方が分からない。** PR の粒度、revert の定義、性能の測定方法と値は示されない。「1か月」がどの月かも、講演の日付が分からないので確定できない。
- **講演者は当事者である。** 講演者は、例に挙げる製品と pstack の作者またはチームの一員である。結びは自社製品の紹介である。
- **例は1種類のコードベースに限られる。** 例はすべて、Electron と React で作るデスクトップアプリである。
- **コメントの禁止は、経験1件に基づく。** 根拠はタイマーの広がりの1件で、禁止の効果を測った結果は無い。
- **所属と製品名は、文字起こしと説明欄の表記による。** 文字起こしは「Grok Bot at SpaceX xAI」、題名と説明欄は「SpaceXSI」と書く。pstack の README（`bf0c7cd`）は Cursor での仕事として書かれている。表記の違いの理由は確かめていない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせた結果である。どの設定にも入れていない。

| 講演の考え方 | 手元の資料 |
| --- | --- |
| 訂正は、rules の行ではなく決定論的な検査にする | [sensible-defaults.md](sensible-defaults.md) に引いた Osmani の記事は、守られない CLAUDE.md の行を hook に移すと書く。講演は、hook より手前の lint と設計を先に挙げる。[stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) は、完了の判定を文章からコマンドに移した手元の例である |
| 自動化の前に、コードベースを直す | [measure-tests-before-adding-practices.md](../knowledge/measure-tests-before-adding-practices.md) の順番は「まず測る」で、講演の順番は「訂正のたびに lint を足す」である。どちらも、観察した失敗から手法を入れる。講演は、今ある検査がどこまで拾うかを測る手順を示さない |
| agent が自分でコードを動かして確かめる | 同じページは、確認の段が実行と読解のどちらで行われているかを点検すると書く。講演の「実行時の検証」は、実行の側にあたる |
| コードベースは agent への最大の入力である | [codebase-sweep-prompts.md](codebase-sweep-prompts.md) は、コードベースを総点検するプロンプトを集めた。講演の「古いパターンを片付ける」作業にあたる |
| 形式検証は、層の1つとして実験中である | [pstack-formal-verification.md](pstack-formal-verification.md) は skill の本文を照合した。用途に Lean があることと、実験中という位置づけは、skill の本文には無く、この講演で分かった。[state-machine-check-in-app-language.md](../knowledge/state-machine-check-in-app-language.md) は、TLA+ を入れずに状態を全列挙する形を題材で確かめた |
| コメントの99%を禁止する | `~/.claude` の rules は、コードに持たせられない why と why-not に限ってコメントを残す。講演の方針と食い違う。[code-comments-for-agents.md](../knowledge/code-comments-for-agents.md) は、コメントを消した条件で agent が規則を消すかを測り、消さなかった。その結果から、強さを選べるコメントの検査を feedback-loop の Issue にした。手元で、コメントが応急処置の正当化に使われた例は集めていない |

## 照合の結果

- **照合した主張：** 「考え方と根拠」の各行は、文字起こしに発言がある。チームの PR の件数、revert の割合、TLA+ と Lean の利用は、動画の説明欄にも記載がある。
- **文字起こしと動画の対応：** 説明欄の章の開始時刻8つを、文字起こしの段落の時刻と比べた。7つは差が3秒以内で、「11,000 PRs」の章だけ10秒ずれる（説明欄 13:32、文字起こし 13:22）。
- **照合していないもの：** 文字起こしと音声の一致。文字起こしの作成方法は分からない。スライドの内容も見ていない。
- **読んでいないもの：** 講演が引く「トークンの70%が検証」の投稿。Dune、Bombadil、feature map の実物。

## 照合の手順

1. 動画の題名とチャンネルを取得する：`https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=O_zcc6Xl12g&format=json` を開く。`title` と `author_name` を読む。
2. 動画のページの HTML を取得する。`scout fetch` は本文を抽出できず、題名だけを返す。HTML を直接取得し、`"publishDate"`、`"lengthSeconds"`、`"shortDescription"` の値を読む。
3. 説明欄の章の開始時刻を、文字起こしの段落の時刻と比べる。
4. 文字起こしを通読し、考え方ごとに発言の時刻を控える。

## 見直す条件

- 講演者かチームが、PR の件数、revert の割合、性能の測り方を公開したとき。
- Dune、feature map を持つ検証 skill、コメントを禁止する lint ルールのいずれかが公開されたとき。
- 手元で、コメントや rules が応急処置の正当化に使われた例を集めたとき。コメントを残す範囲を見直す材料になる。
- 動画の音声と文字起こしを突き合わせたとき。
