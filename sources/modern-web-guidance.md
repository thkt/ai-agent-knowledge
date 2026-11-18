# Modern Web Guidance が agent に Web の新しい書き方を引かせる作りと、効果の測り方（2026-11）

Modern Web Guidance は、Chrome のチームが公開した agent 向けの skill 集である。Web の新しい機能の使い方を、agent が CLI で検索して context に入れる。紹介のページ、skill の本文、公開の eval の説明、同じチームのブログ記事を照合した。自分たちが採用した判断ではなく、手元では入れていない。

## TL;DR

- 主張は、指導なしの agent は古い書き方に寄る、というものである。ブログは、検査の合格率が約50%から約90%に上がると書く。
- skill は知識を抱えない。agent は CLI の `search` で候補を得て、`retrieve` で必要な guide だけを読む。
- eval は、同じ課題を skill なしと skill ありで走らせ、Playwright がブラウザで採点する。採点の要件は、guide が定める。
- 合格率の内訳（モデル、課題の数、試行の数）は取得できなかった。数字は、提供元の自己申告である。
- skill の説明は、該当する作業で最初に必ず実行するよう求める。コマンドは、実行のたびに最新のパッケージを取得する。

| 項目 | 内容 |
| --- | --- |
| 出典 | 次の表の5件 |
| 著者・発行元 | Google Chrome のチーム。README は、Chrome、Edge、Firefox、MDN のチームと Web 開発のコミュニティが運営すると書く |
| 照合日 | 2026-11-17 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 版と公開日 | 読んだ範囲 |
| --- | --- | --- | --- |
| 紹介 | [Modern Web Guidance](https://developer.chrome.com/docs/modern-web-guidance?hl=ja) | 日付の記載なし | 全文。日本語の版 |
| skill | [GoogleChrome/modern-web-guidance](https://github.com/GoogleChrome/modern-web-guidance)（Apache-2.0） | main `64a6d70`、2026-11-16。作成は 2026-03-23 | `README.md` の冒頭119行と、`skills/modern-web-guidance/SKILL.md` の冒頭69行。ファイルの一覧 |
| 元の repo | [GoogleChrome/modern-web-guidance-src](https://github.com/GoogleChrome/modern-web-guidance-src)（Apache-2.0） | main `6971ebb`、2026-11-16 | `docs/EVALS.md` の Overview、Claude Code の節、effort の記録の節。`docs/eval-results.md` の全文 |
| ダッシュボード | [Modern Web Guidance Evals](https://googlechrome.github.io/modern-web-guidance-src/) | 毎晩更新される | 「How we test & measure」の説明。数値は読み込まれず、取得できなかった |
| ブログ | [Top 10 things AI gets wrong in web development](https://developer.chrome.com/blog/top-10-things-ai-gets-wrong-in-web-dev)（Philip Walton） | 2026-11-11 | 全文 |

## 中心の主張

**新しいモデルでも、指導なしでは Web の新しい書き方に従わない。** ブログは、よく受ける問いとして「新しいモデルは、すでに良い書き方に従うのではないか」を挙げ、測った結果として否と答える。理由は README が書く。学習に使われたコードに古い書き方が多く、モデルは、ブラウザに組み込みの機能で済む作業に、量の多い JavaScript を書く。API があると知っていても、実際の使い方の例が足りない。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| skill に知識を抱えず、CLI で引かせる | skill の本文は9.6KB である。agent は `search` で guide の ID と説明を得て、`retrieve` で必要な guide だけを読む。guide は210ファイルあり、README は262の機能を覆うと書く | `SKILL.md` とファイルの一覧で確かめた。README の262は、機能の数で、ファイルの数ではない |
| 検索の書き方を指示する | 検索は、2語から5語で、1つの機能だけを書く。framework の名前とアプリの名詞を入れない。利用者の言い方（確認してから消す）を、部品の名前（確認のダイアログ）に言い換える。結果が弱ければ、1回だけ言い換える | `SKILL.md` に記述がある |
| 当たる guide が無ければ、無理に使わせない | 言い換えても説明が合わなければ、関係の無い guide を読まずに進める | `SKILL.md` に記述がある |
| 実装を、読んだ guide に照らして確かめさせる | 納品の前に、guide の書き方と fallback が当たっているかを確かめる。頼まれていない機能は足さない | `SKILL.md` に記述がある |
| 使ったことを開示させる | 納品の最後に、当てた guide の ID と、その guide の規則の短い引用を書く。context が圧縮された後は、記憶から ID や引用を書かず、検索し直す。検索して何も当てなかった場合は、調べた話題と、当たらなかった理由を1文で書く | `SKILL.md` に記述がある |
| 古い書き方を、決定論的な規則で探す | `legacy-scan` は、決定論的な規則を持つ guide だけを調べる。全体の約20%である。見つかったものは、残す条件と捨てる条件に照らし、誤検出を落とす | `SKILL.md` に記述がある。規則の中身は読んでいない |
| guide から、モデルが知っている内容を削る | eval を走らせ、どのモデルも知っている内容を削る。token を節約するためである | README に記述がある。削った量は示されない |
| guide ごとに、満たすべき要件を定める | guide は、書き方に従ったなら真になる要件を定める。eval は、同じ依頼を skill ありと skill なしの agent に渡し、両方の出力を要件に照らす | ブログに記述がある |
| ブラウザで採点する | 課題は、雛形のアプリと機能の依頼の組である。Playwright が、できたアプリをブラウザで開き、DOM、スタイル、ソースを要件ごとに確かめる。合格率は、通った検査の数を、走らせた検査の数で割る | ダッシュボードの説明に記述がある。課題ごとの合否ではなく、検査ごとの割合である |
| 対の試行を、そろえて外す | 基盤の障害や時間切れで採点に届かなかった試行は、対になる試行と一緒に外す | ダッシュボードの説明に記述がある |
| 依頼は、skill の名前を出さない | agent が、guide が当たると気づき、細部まで従う必要がある。skill ありで100%にならない理由として、手順の飛ばし、guide の引き損ない、採点の厳しい端の場合を挙げる | ダッシュボードの説明に記述がある。引き損ないは、検索の質のページで別に追うと書く。そのページは読んでいない |
| 妥当な別の実装を落とさない | 採点は、guide に固有の変数名や構造ではなく、描画された振る舞いと、仕様に沿ったアクセシビリティの意味を見る。別の実装を落とす採点が見つかれば、検査を広げて採点し直す | ダッシュボードの説明に記述がある。見直した件数は示されない |
| 所要時間と token も報告する | 所要時間は、採点を除いた中央値を agent ごとに出す。token は、キャッシュを除いた量を出す。会話の履歴を毎回送り直す分で、合計が膨らむためである | ダッシュボードの説明に記述がある |
| 推論の強さを、試行ごとに記録する | 試行と job は、モデルと並べて、推論の強さと、その出どころ（観測、設定、モデルの ID、agent の既定）を記録する。設定とモデルの ID が食い違う指定は、起動時に失敗させる | 元の repo の `docs/EVALS.md` に記述がある |

## 数字

合格率は、資料ごとに書き方が少し違う。

| 資料 | 指導なし | 指導あり | 時点 |
| --- | --- | --- | --- |
| 紹介 | 約51% | 91% | 記載なし |
| ブログの本文 | 約50% | 約90% | 記載なし |
| ブログの図の説明 | 52% | 91% | 2026-11-09 |

ブログは、ほかに2点を書く。eval の全課題を通じて、依頼に書かれない限り、どのモデルも CSP と cross-origin isolation のヘッダーを足さなかった。依頼がセキュリティを直接求めても、指導なしのモデルは、セキュリティの検査の半分より多くで落ちた。

## 指導なしの agent が間違える10の領域

ブログは、領域ごとに4件から5件の誤りを挙げる。代表を1件ずつ引く。

| 分野 | 領域 | 代表の誤り |
| --- | --- | --- |
| アクセシビリティ | フォーカスと `inert` | 自作の modal に、JavaScript の tab の閉じ込めと `aria-modal` を使う。modal の `<dialog>` か、背景の `inert` を使わない |
| アクセシビリティ | 動きを減らす設定 | スクロール連動のアニメーションや view transition で、`prefers-reduced-motion` を確かめない |
| アクセシビリティ | フォームの検証と自動入力 | CSS の `:user-invalid` を、`blur` の JavaScript で作り直す。触れていない必須の欄を、tab で通っただけで誤りと表示する |
| 利用体験 | スワイプとタッチ | 消す、開くの操作を `pointermove` で作る。CSS の Scroll Snap を使わない |
| 利用体験 | dialog、popover、最前面の層 | `position:fixed` の重ねと、外側のクリックの処理を自作する。`popover` を使わない |
| 利用体験 | ダークモード | `<meta name="color-scheme">` を入れず、読み込みの間に白い画面が出る |
| 性能 | 操作への応答 | 優先度の低い作業に、自作の `setTimeout` の列を使う。`scheduler.postTask()` と `scheduler.yield()` を使わない |
| 性能 | レイアウトのずれ | `<video>` に `width` と `height` を付けない。画面の高さに `vh` を使い、モバイルのツールバーで中身が跳ねる |
| セキュリティ | DOM XSS と HTML の無害化 | `DOMParser` の許可リストを自作する。入力をそのまま返す Trusted Types の policy を登録する |
| セキュリティ | passkey | Base64URL の変換を `atob()` と `btoa()` で自作する。組み込みの直列化の関数を使わない |

## 限界

- **合格率の内訳を取得できなかった。** ダッシュボードの値は読み込まれず、対象のモデル、課題の数、試行の数、区間は分からない。3つの資料の数字が1点ずつずれる理由も、確かめていない。
- **採点の要件を、guide を書いた側が定める。** eval が測るのは、guide が教える書き方に従ったかである。ダッシュボードの説明は、妥当な別の実装を落とす採点がありうることを認め、見つけたら直すと書く。
- **合格率は、検査の単位である。** 1つの課題が複数の検査を持つので、課題が最後まで動いた割合とは違う。
- **比べたのは、skill の有無だけである。** 指導なしの agent に、同じ長さの別の文書を渡した条件は、読んだ範囲に無い。
- **skill の説明が強い。** HTML、CSS、クライアント側の JavaScript の作業すべてで、最初に必ず実行し、飛ばさないよう書く。入れると、該当する作業のたびに検索が入る。
- **コマンドが、毎回最新のパッケージを取得する。** skill は `npx -y modern-web-guidance@latest` を実行させる。実行には npm の registry への通信が要り、取得する中身は、公開された時点の最新になる。`search` には skill の版を渡す引数があるが、パッケージの版は固定されない。
- **事例は読んでいない。** 紹介のページは、4社の事例があると書く。本文は読んでいない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の設定と資料を読んで突き合わせた結果である。どの設定も変えていない。

| Modern Web Guidance の考え方 | 手元の資料 |
| --- | --- |
| skill に知識を抱えず、CLI で引かせる | [astryx-agent-evals.md](astryx-agent-evals.md) の Astryx も、agent にパッケージのファイルを読ませず、コマンドで文書を引かせる。2つの資料が、同じ入口の形を選んでいる |
| ブラウザで採点し、skill の有無で比べる | Astryx の eval と同じ形である。違いは3つある。Astryx は、4つの検査がすべて通った課題を合格とし、合格率に区間を付け、採点器を使う前に試す。Modern Web Guidance は、検査ごとの割合を出す。区間と、採点器の事前の検査は、読んだ範囲に記述が無い |
| ネイティブの機能を先に使う | `~/.claude` の Reuse Ordering は、新しいコードを書く前に、プラットフォームの機能を使うと書く。例は、date の入力と、JavaScript より CSS である。`reviewer-progressive` は、JavaScript のレイアウトやアニメーションのうち、ブラウザの CSS で置き換えられる箇所を探す。手元に無いのは、機能ごとの具体的な書き方と、互換性のデータである |
| 使ったことを開示させる | [skill-evals-makarevych.md](skill-evals-makarevych.md) は、skill ごとに、指示どおり動いたかと、成果が良くなったかの2つを測ると書く。開示の行は、前者を記録から数えられるようにする。手元の skill に、この形は無い |
| guide から、モデルが知っている内容を削る | [always-on-agents.md](always-on-agents.md) の Managed Agents の記事は、ハーネスの仮定がモデルの改善で古くなると書く。`~/.claude` の `ablate` の skill は、要素を外して結果が動くかを確かめる。同じ問いへの手段である |
| コマンドが、毎回最新のパッケージを取得する | feedback-loop の OUTCOME は、導入先の日常の実行が、毎回取得する最新版に依存しないことを制約に挙げる。逆の設計である |
| skill の効果を、公開の数字で示す | skill-evals のページは、手元の31個の skill のうち3個が、90日のあいだ一度も読み込まれていなかったと記録する。効果を測った skill は、手元に無い |

## 照合の結果

- **照合した主張：** 各行は、右の列に挙げた資料に記述がある。
- **数えたもの：** `skills/modern-web-guidance/guides/` の下の Markdown は210ファイルだった。`SKILL.md` は9.6KB だった。
- **取得できなかったもの：** ダッシュボードの数値。ページの script が読む `suites.gen.json` は、取得すると空の配列だった。
- **動かしていないもの：** CLI、`legacy-scan`、eval。
- **読んでいないもの：** guide の本文。`legacy-scan` の規則。README の120行目以降と、`SKILL.md` の70行目以降。1.0 の発表の記事、4社の事例、Codelab、検索の質のページ。元の repo のコード、`docs/CONTEXT.md`、`src/harness/README.md`。

## 照合の手順

1. 紹介のページを取得する：`scout fetch "https://developer.chrome.com/docs/modern-web-guidance?hl=ja"`
2. 版を確かめる：`gh api 'repos/GoogleChrome/modern-web-guidance/commits?per_page=1' --jq '.[0].sha'`。元の repo も同じ方法で確かめる。
3. skill を読む：`scout repo-read GoogleChrome/modern-web-guidance skills/modern-web-guidance/SKILL.md`
4. guide を数える：`scout repo-tree GoogleChrome/modern-web-guidance` の出力を、`skills/modern-web-guidance/guides/` で絞り、`.md` の行を数える。
5. ダッシュボードの説明を取得する：`scout fetch https://googlechrome.github.io/modern-web-guidance-src/`
6. ブログを取得する：`scout fetch https://developer.chrome.com/blog/top-10-things-ai-gets-wrong-in-web-dev`

## 見直す条件

- ダッシュボードの数値を、ブラウザで読めたとき。モデル、課題の数、試行の数を足す。
- 手元の Web の UI の案件で、この skill を入れて試したとき。結果は `knowledge/` に書く。
- skill が、パッケージの版を固定する形に変わったとき。照合した版は `64a6d70` である。
- 第三者が、同じ課題で合格率を測ったとき。
