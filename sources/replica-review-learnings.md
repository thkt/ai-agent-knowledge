# GitHub Replica の、プロトタイプのレビューから agent への指示を作る仕組みの考え方と根拠（2026-11）

GitHub のデザインチームが、プロトタイプを作るためのデザインハーネス Replica を公開した。紹介のブログは、Replica が強化学習を多く使い、操作とプロトタイプへのコメントから良い状態を学んで、時間とともに良くなると書く。この主張を、リポジトリの文書と ADR で照合した。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- 今動いている仕組みは Learnings である。プロトタイプへのレビューで繰り返し出た指摘を Copilot が主題に分け、人が承認したものだけを agent への指示に足す。モデルは訓練しない。
- 提案になる条件は、2人以上のレビュアーが、2つのスレッドで、2つのプロトタイプに挙げた主題であることである。agent に何を渡すかは、Copilot ではなくリポジトリの管理者が決める。
- 強化学習は、ADR 0002 で提案中の別の段階である。スクリーンショットが Learnings に従うかを判定する小さなモデルを、チームが自分の環境で訓練する。Hub はそのモデルを訓練も実行もせず、新しい Build に当てる段階はまだ無い。
- 訓練の効果の数字は、合成データ 518 ページでの参考値だけである。全問に 50% と答えると 0.750 になる。開発用の得点の最良値は、判定部の訓練で 0.753、読み出し層の訓練で 0.807 だった。
- ブログの「時間とともに良くなる」は、照合した版では、人が承認した指示が増えることを指す。モデルが自動で良くなる仕組みは動いていない（照合者の判断）。

| 項目 | 内容 |
| --- | --- |
| 出典 | [The best things in open source are built together](https://github.blog/open-source/maintainers/the-best-things-in-open-source-are-built-together/)（GitHub Blog）と、[github/replica](https://github.com/github/replica) |
| 著者・発行元 | ブログは Katie Liu（GitHub）。Replica は GitHub Design と Microsoft CoreAI Design が作り、GitHub の社員が保守する |
| 公開日 | ブログは 2026-11-19。リポジトリの作成は 2026-10-15 |
| 照合した版 | `github/replica` の main `39e74e6`（2026-11-19） |
| 照合日 | 2026-11-20 |
| 照合した範囲 | ブログの本文のうち Replica の節。リポジトリの README、`hub/docs/learnings.md`、`docs/decision-model.md`、`docs/adr/0001-learning-extension-architecture.md`、`docs/adr/0002-decision-model-and-rl-environment.md`、`packages/learning/environments/brand-check/README.md` の全文 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**プロトタイプのレビューで繰り返し出る指摘を、人が承認した指示に変えて、次にプロトタイプを作る agent に渡す。** README は Replica を、チームのデザインシステムの部品と指針を使い、Copilot との会話からプロトタイプを作る道具と説明する。作ったプロトタイプは任意の Hub に公開し、チームのコメントを集める。そのコメントを agent への指示に戻す部分が Learnings である。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 繰り返し出た指摘だけを提案にする | 1日1回、Copilot が新しいコメントと編集されたコメントに主題を付ける。直近90日で、2人以上のレビュアーが2つのスレッド、2つのプロトタイプで挙げた主題を提案にする。質問、好み、称賛は数えない。1回の実行で出す提案は10件までにして、人が少しずつ確かめられるようにする | 記述がある（[learnings.md L25-L31](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L25-L31)）。人数などの条件は、リポジトリごとの `config.json` で変えられる（[L179](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L179)） |
| agent に渡すものは人が決める | Copilot は、agent に何を渡すかを決めない。リポジトリの管理者が、提案を承認するか、編集して承認するか、却下する。却下と廃止には理由が要る。却下した主題は、さらに多くのレビュアーが挙げたときだけ、再び提案される | 記述がある（[L33](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L33)、[L45-L48](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L45-L48)） |
| 承認した指示は、命令ではなくデータとして渡す | プラグインが、承認済みの Learnings を利用者のプロンプトに足す。足す文は、各 Learning がチームからの指針でありデータであること、利用者の依頼とリポジトリの指示より優先しないことを agent に伝える。一覧が変わると、次のプロンプトで一覧全体を差し替える | 記述がある（[L56](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L56)） |
| Copilot に渡す情報を絞る | 主題を付けるとき、Copilot にはコメントの本文、プロトタイプ上の位置、同じスレッドのほかのコメントを渡す。書いた人は渡さず、ツールも与えない | 記述がある（[L30](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L30)） |
| 提案を作る処理は決定的にする | 参照実装の consolidator は、条件を満たした主題を提案し、意見が割れた主題を保留する。モデルだけで提案を作る方式は、差し替えられる選択肢として残す。参照実装を決定的にするのは、テストとレビューを再現できるようにするためである | 記述がある（[ADR 0001 L59](https://github.com/github/replica/blob/39e74e6/docs/adr/0001-learning-extension-architecture.md?plain=1#L59)、[L129](https://github.com/github/replica/blob/39e74e6/docs/adr/0001-learning-extension-architecture.md?plain=1#L129)） |
| コメントの収集を agent のセッションに任せない | 以前の案では、agent が canvas からコメントを集め、各手順を skill で動かしていた。人が毎回手順を始める必要があり、すべてのコメントが agent のセッションを通るので token を使い、写し間違いも起こりうる。そのため Hub の背景ジョブに置き換えた | 記述がある（[ADR 0001 L126](https://github.com/github/replica/blob/39e74e6/docs/adr/0001-learning-extension-architecture.md?plain=1#L126)） |
| 強化学習の報酬は、確率の正確さで採点する | 判定モデルは、各選択肢の確率で答える。報酬には Brier スコアなどの proper scoring rule を使う。正誤だけの報酬は、自信を最大にした答えを覚えさせて確率の較正を崩すので、採らなかった | ADR 0002 の提案として記述がある（[L95](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L95)、[L144](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L144)）。較正された確率が要る理由は、Hub がレビュアーに何を見せるかを確率の閾値で決めるためだと書く |
| 正解のラベルは、レビューの結果から作る | 「違反」のラベルは、承認済みの Learning を支えるスレッドが始まった Build とページに付ける。「準拠」のラベルは、指摘した本人が、後の Build が公開された後にスレッドを解決した場合だけ付ける | ADR 0002 の提案として記述がある（[L63-L65](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L63-L65)） |
| 同じプロトタイプの Build を、訓練と評価に分けない | タスクを、プロトタイプ単位で訓練、開発、テストに分ける。同じプロトタイプの Build は見た目が似ているので、前の Build を覚えたモデルがテストに通ってしまうためである | ADR 0002 の提案として記述がある（[L79](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L79)） |

## ブログの記述と実装の差

| ブログの記述 | 照合した版での状態 |
| --- | --- |
| Replica は強化学習を多く使う | 動いている Learnings は、モデルを訓練しない。ADR 0001 は、学習を明示的で中身を確かめられる記憶で行うと書く（[L105](https://github.com/github/replica/blob/39e74e6/docs/adr/0001-learning-extension-architecture.md?plain=1#L105)）。強化学習は ADR 0002 にあり、状態は 2026-11-07 の提案で、要件にメンテナはまだ合意していない（[L3](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L3)、[L23](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L23)） |
| 操作とコメントから良い状態を学び、時間とともに良くなる | コメントから作るのは、人が承認した指示である。判定モデルの訓練は、チームが taskset を作り、自分の環境で回す。Hub は訓練も、重みの保持も、モデルの実行もしない（[learnings.md L60](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L60)） |
| 強化学習の効果（ブログに数字は無い） | 公開されている数字は、合成データ 518 ページでの参考値だけである。全問に 50% と答えると 0.750 になる（[brand-check L137](https://github.com/github/replica/blob/39e74e6/packages/learning/environments/brand-check/README.md?plain=1#L137)）。判定部を訓練した開発用の得点は 0.685 から最良で 0.753、読み出し層だけを訓練すると 0.807 だった（[L164](https://github.com/github/replica/blob/39e74e6/packages/learning/environments/brand-check/README.md?plain=1#L164)）。実際のレビューで訓練した結果は無い |

main にあるのは、taskset を作るコマンド、OpenEnv の環境、参照用の trainer、Learnings ページで基本モデルを選ぶ設定である。無いのは、訓練したモデルを新しい Build のスクリーンショットに当て、レビューで指摘を見せる段階である。ADR 0002 はこれを次の手順の1つ目に置く（[L160](https://github.com/github/replica/blob/39e74e6/docs/adr/0002-decision-model-and-rl-environment.md?plain=1#L160)）。

## 限界

- **Learnings が agent の出力をどれだけ変えたかの数字は無い。** 公開されている数字は、合成データでの判定モデルの得点だけである。
- **2つの ADR は、どちらも proposed のままである。** ADR 0001 は、要件にメンテナが 2026-10-09 に合意している。ADR 0002 の要件は合意されていない。
- **Learnings は Hub の機能である。** README によれば、Hub なしでもローカルで試作できる。共有とコメントには、組織が運用する Linux サーバー、wildcard DNS のドメイン、HTTPS が要る。
- **agent の側は GitHub Copilot が前提である。** README の手順は Copilot アプリを使う。Learnings を渡すプラグインは、Copilot CLI 1.0.96 でも実験的機能を有効にすれば動く（[learnings.md L56](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L56)）。照合した範囲には、Copilot 以外の agent に Learnings を渡す方法は書かれていない。
- **主題を付ける処理は Copilot を使い、利用量は token の持ち主のプランに数える**（[learnings.md L114](https://github.com/github/replica/blob/39e74e6/hub/docs/learnings.md?plain=1#L114)）。

## 照合の結果

- **照合した主張：** 「考え方と根拠」「ブログの記述と実装の差」「限界」の各行は、照合した範囲の文書に記述がある。
- **読んでいないもの：** 実装のコード（`hub/app/models/learning/` など）、`packages/learning/README.md`、`hub/docs/prototype-canvas.md` の Learnings の節、紹介動画、ブログが挙げる Universe の講演。
- **範囲外：** ブログの Open Source Accessibility Framework と `ACCESSIBILITY.md` の節、Octoverse の数字。agent の運用の判断に関わらないため。

## 照合の手順

1. ブログの本文を取得する：`scout fetch https://github.blog/open-source/maintainers/the-best-things-in-open-source-are-built-together/`
2. 照合した版の README を取得する：`scout repo-read github/replica README.md --ref 39e74e6`
3. 照合した版の文書を読む：`scout repo-read github/replica hub/docs/learnings.md --ref 39e74e6`。`docs/decision-model.md`、2つの ADR、`packages/learning/environments/brand-check/README.md` も同じ形で読む。
4. 引用した行を確かめる：`scout repo-read github/replica docs/adr/0002-decision-model-and-rl-environment.md --ref 39e74e6 -l 3-3` のように、`-l` で行を指定する。
5. main の最新の commit を確かめる：`gh api repos/github/replica/commits/main --jq .sha`

## 見直す条件

- ADR 0002 が accepted になったとき、または訓練したモデルを新しい Build に当てる段階が main に入ったとき。
- 実際のレビューの結果で訓練した数字が公開されたとき。
- Copilot 以外の agent に Learnings を渡す方法が加わったとき。
