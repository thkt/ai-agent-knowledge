# OpenClaw が CI の計算資源を減らした記事の考え方と根拠（2026-11）

Vincent Koc の記事は、agent がコードとテストを増やして詰まった OpenClaw の CI を、どう軽くしたかを書く。記事の全文と付録を読み、記事が版を固定した openclaw/openclaw の `a7d05c7` で、テスト行数の上限、選んだ理由の記録、全件に戻すスイッチ、定期実行の設定を照合した。記事の算術も再計算した。手元の feedback-loop（`d9898c8`）とも突き合わせた。自分たちが採用した判断ではない。

## TL;DR

- 題名は「速く、95% 安く」だが、速さの根拠はジョブ単位の短縮に限られる。PR の CI 時間の中央値は 6分41秒（6月）、12分10秒（9月）、9分18秒（11月4日）と動き、費用の減少と並んで一様に速くなったわけではない。PR を作ってから merge するまでの時間が縮んだかは「まだ示せていない」と記事自身が書く。
- 減ったのは計算資源である。runner の稼働時間は 86.5%、公開価格での見積もり額は 95.5% 減った。同じ期間に作業量も変わったので、施策ごとの効果には割り振れないと記事自身が書く。算術は再計算で一致した。
- 順番は、要らない作業を消し、残りを変更に合わせて選び、準備を使い回し、ジョブを詰めて均す、である。速い runner を買うのは最後である。
- テストの選び方は決定論的なコードである。関係が分からなければ範囲を広げ、選んだ理由を記録する。学習モデルは、全件を走らせたまま予測を記録する段に留め、テストを飛ばす権限を与えていない。
- feedback-loop は、選び方と、分からないときに全件へ広げる点で同じ形を持つ。選んだ理由の記録、全件に戻すスイッチ、見落としを測り続ける仕組みは持たない。テスト行数の上限は、入れないと判断した。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Making CI faster and 95% cheaper](https://vincent.build/openclaw-ci-cost-reduction/)（Vincent Koc）と、記事が版を固定した [openclaw/openclaw](https://github.com/openclaw/openclaw) の `a7d05c7` |
| 著者・発行元 | Vincent Koc。記事は OpenClaw のチームを「私たち」と書く。runner は Blacksmith と boat.dev がスポンサーとして提供し、GitHub も runner を寄付している。金額は Blacksmith の公開価格での見積もりで、支払った額ではない。記事の末尾は、本文を Fable で書き、Codex で事実を確かめたと書く |
| 公開日 | 2026-11-16 |
| 照合した版 | openclaw/openclaw は `a7d05c7`（記事が引く版）。feedback-loop は main の `d9898c8` |
| 照合日 | 2026-11-18 |
| 照合した範囲 | 記事の全文と付録。openclaw は、`docs/ci/local-proof.md` のテスト行数の節、`docs/ci/adaptive-testing.md` の全件に戻す変数、`scripts/test-projects.test-support.mts` と `scripts/lib/ci-changed-node-test-plan.mts` の選んだ理由と fallback、`.github/workflows/ci.yml` の定期実行と main への push の条件。記事が引く PR は開いていない。記事は、private のログと provider の書き出しを公開していないと書く |
| この資料を根拠にした Issue | [thkt/feedback-loop#203](https://github.com/thkt/feedback-loop/issues/203)（選んだ理由の記録）、[#204](https://github.com/thkt/feedback-loop/issues/204)（全件に戻す環境変数）、[#205](https://github.com/thkt/feedback-loop/issues/205)（見落としを数え続ける方法の調査）。[#188](https://github.com/thkt/feedback-loop/issues/188) の本文に、選ばれたテストが落ちることの確認と、3/N の読み方を足した（2026-11-18） |

## 中心の主張

**速い機械を買う前に、走らせる意味のない作業を消す。** 記事の制約は、token でもメモリでもなくテストだった。agent がコードを速く書くほど、テスト、ジョブ、再試行が増え、変更が安全かの答えを待つ時間が積み上がった。

記事は、目標を「役に立つ作業を早く終え、途中で無駄にする計算資源を減らす」とする。安い機械で、後に続く agent の待ちが延びるなら勝ちではない。速い機械で、大半が遊んでいるなら勝ちではない。始めなくてよかったジョブは、費用もかからず、誰も待たせない、と書く。

## 考え方と根拠

### 作業を減らす順番

番号は記事の節の番号である。

| 考え方 | 記事の記述 | 照合の結果 |
| --- | --- | --- |
| 1. 弱いテストを先に消す | agent が育てたテストの大半は、独立した契約を示せるまで雑音と見なす。モデルは足すのが得意で、消すには明示の指示と完了の定義での位置づけが要る。テスト行数の上限を、プロンプトではなく検査で置く。1つの変更で正味 400 行まで記録なしで足せ、超える分は記録が要る。リポジトリ全体でテストを削る作業を、`/goal` で続けさせるプロンプトも載せる | 上限は `docs/ci/local-proof.md` の `test-line-budget` の節（L334-339）にある。超えた分の記録は `config/test-lines-allocations/<change>.txt` に置く。10月24日の当初の検査（削った分だけ足せる）は PR #182230 で、開いていない |
| 2. 検査を段階ごとに分ける | PR には merge に必要な答えだけを出す。main への push では小さな検査だけを走らせる。全体の検証は1時間ごとの定期実行に移す。CodeQL も1時間ごとにし、変わっていない区分の成功は24時間まで使い回す | `.github/workflows/ci.yml` に、毎時の cron（`23 * * * *`、L4-5）と、main への push で全体の経路を走らせるのは `vars.OPENCLAW_CI_ON_PUSH == 'true'` のときだけという条件（L125）がある。変数の実際の値は、記事も確かめていないと書く |
| 3. 同じレビューを2回払わない | レビュー bot の ClawSweeper は、PR ごとに自動のレビューを1回だけ行う。2回目からは依頼したときだけ走る。手元でレビューする CLI もある | 未照合 |
| 5. 安い失敗は手元で拾い、速い道具に替える | 書式や lint の失敗は、remote の CI に行く前に拾う。型検査を tsgo から tsrs に替え、手元の比較で 447秒が64秒になった | 算術は一致した（85.7%）。PR #175711 は開いていない |
| 6. 同じ環境を作り直さない | blobless checkout を使う。キャッシュが本当に当たっているかを確かめる。55.1 MB の archive を、30.0 MB と 0.48 MB の2つに分けた。一部の Vitest のレーンで worker を使い回し、174.2秒が97.1秒になった | 算術は一致した（44.3%、37.6%）。PR は開いていない |
| 7. ジョブを詰めて、終わる時刻をそろえる | shard は batch の分け方で、並列はいくつ同時に走らせるかである。shard を 95 から 63 にまとめた。逆に、ジョブを 18 から 20 に増やして最も遅いジョブを 403秒から300秒にした。小さい runner は 41.7% 安くなり、16.5% 遅くなった | 算術は一致した（25.6%、41.7%、16.5%）。小さい runner の比較は1組の実行で、2つとも同じ既知の失敗を含む、と記事が書く |

### テストの選び方

| 考え方 | 記事の記述 | 照合の結果 |
| --- | --- | --- |
| 決定論的なコードで選ぶ | 同じ版、差分、規則なら、同じテストを選ぶ。モデルの確信度は、依存関係の代わりにならない | 記事に記述がある |
| 担当するテストと、import している側のテストを辿る | 変更したテスト自身、明示した担当のテスト、変更したコードを import する側のテストを選ぶ。import に見えない fixture の読み込みは、Vite の `addWatchFile` で宣言し、fixture を壊して選ばれ落ちることを確かめた | `scripts/test-projects.test-support.mts` が、選んだテストに `explicit-owner`、`conventional-owner`、`import-consumer` の規則名を付けて記録する（L4906-4922） |
| 分からなければ広げる | 依存の記録が無いことは、何も壊れない証拠ではなく、多く走らせる理由である。担当の情報が欠けた絞り込みは、空の成功ではなく却下し、広い実行に戻す | `scripts/lib/ci-changed-node-test-plan.mts` は、標準の shard の情報が無い対象を fallback に回す（L1293）。`policy-watch` と `fixed-smoke` の規則名で、差分に関係なく残す検査も記録する（L489-493） |
| 全件に戻すスイッチを持つ | リポジトリの変数 `OPENCLAW_CI_NODE_SELECTION=full` で、PR での絞り込みを止める | `docs/ci/adaptive-testing.md` の L254 に記述がある |
| 選択器を過去の不具合で確かめる | 修正の前の版に戻し、その変更を選択器に通し、不具合を見せるテストが選ばれて落ちるかを確かめる。削除、共有の helper、fixture、名前の変更も含める | 記事に記述がある。すべての過去の不具合で再生したわけではない、と記事が書く |
| 学習モデルは、全件を走らせたまま測る | 学習モデルが選ぶはずの組を記録し、全件を走らせ、組の外で落ちたテストを見落としと数える（shadow mode）。モデルは小さな決定木で、テストを飛ばす権限も並べ替える権限も無い。見落とし0件でも、独立した変更 N 件なら見落とし率の上限は約 3/N である | ranking AUC の 0.98276 は、342/348 で再計算が一致した。64行のうち正例は6件で、植え込んだ変異であり、実際の不具合ではないと記事が書く。3/N は「3の規則」と呼ばれる統計の目安である |

### 測り方

記事は、時計を2つ並べる。CI の結果が出るまでの時間と、PR を作ってから merge するまでの時間である。CI だけを速くしても、レビューと直しの待ちが残ると書く。11月4日までの30日で、著者の PR 1,714件の中央値は、draft の時間を含めて37.8分、除いて25.3分だった。どちらも絞り込みを入れる前の値である。

## 記事が挙げる数字

| 数字 | 記事の値 | 照合の結果 |
| --- | --- | --- |
| 30日あたりの見積もり額 | ピークの週から $1,268,958、最近の週から $57,324。95.5% 減 | 付録の式（週の額 ÷ 7 × 30）で再計算し、一致した |
| runner の稼働時間 | 158,546 時間から 21,395 時間。86.5% 減 | 付録の分の値 ÷ 60 で再計算し、一致した |
| ジョブの数 | 1,482,257 から 301,413。79.7% 減 | 一致した |
| 11月5日の1日の額 | 9月のピークの1日の平均より 81.7% 低い | 一致した |
| テストの行数 | 9月28日から11月13日で、テストのファイルが正味 417,906 行減った。補助を含めると 280,540 行減った | 一致した。補助のファイルは 137,366 行増えた |
| 走らせたテストの例 | 4,018 が 15、4,480 が 16、6,895 が 97 | 一致した。著者が選んだ3件の例で、分布ではない |
| PR の CI 時間の中央値 | 6分41秒、12分10秒、9分18秒 | 未照合。生のデータを開いていない |
| PR から merge までの中央値 | 37.8分、draft を除いて25.3分 | 未照合 |

## 限界

- 作業量が変わった。比べた2つの週で、メインのリポジトリの commit は約44% 減った（3,850 から 2,149）。記事は、同じ作業を2つの CI で再生した比較ではないと書く。
- 11月6日から7日の大きな下がりは、絞り込みを入れた時期と重なるが、それ以前の改善も積み上がっていた。記事は、日付の前後は因果の分離ではないと書く。
- PR から merge までの時間は、絞り込みを入れた後の値を示していない。
- 金額は公開価格での見積もりで、runner はスポンサーの提供である。

## 資料の間で食い違う点

1. **テストの選び方に、履歴を使うか。** [claude-code-team-verification.md](claude-code-team-verification.md) が読んだ Anthropic の test impact analysis の記事は、選び方を、テスト結果の履歴と package の関係による、と1文で書く。OpenClaw は、履歴と学習モデルを shadow に留め、テストを飛ばす権限を与えない。どちらも、選び漏れの数は示さない。
2. **選び方の軸。** [e2e-thickness-and-test-selection.md](e2e-thickness-and-test-selection.md) の PIVOT の記事は、ケースを消さずに、回す分をリスクで選ぶ。OpenClaw は、変更からの依存で選ぶ。過去の不具合に当てて選び方を確かめる点は同じである。
3. **テストを消す規則。** [test-audit-openclaw.md](test-audit-openclaw.md) は、同じプロジェクトの test-audit スキルを照合した。そのページはテスト行数の上限に触れていない。この記事は、スキルの規則（守る振る舞いを名指しし、壊れたら落ちる不具合を示し、契約ごとに強いテストを1つ残す）に、行数の上限の検査を足した形である。

## feedback-loop との関係（未採用）

この節は、照合者が feedback-loop の `d9898c8` を読んで突き合わせた結果である。どの設定も変えていない。feedback-loop は、導入先の repo に静的検査と関連テストの反復コマンド `check:affected` を入れる。OUTCOME の Behavior は、反復コマンドが変更の影響を受ける quality テストを実行し、影響が不明なら全 quality を実行することを挙げる。

### OUTCOME の Behavior に直接かかわるもの

| 記事の考え方 | feedback-loop の今 | 参考になる形 |
| --- | --- | --- |
| 選んだ理由を記録する | 全件へ広げたときは理由を出す（`src/bun.ts` L162-163、`src/selection.ts` L110-117）。絞った選択は件数だけを出す。`Selection` の `some` は理由を持たない（`src/selection.ts` L35） | 絞った選択にも、変更したテスト自身、`dependencies`、import のつながり、Bun の記録、記録が無いため常に選ぶ、のどれで選んだかを付ける。#188 が見落としを理由で分けるときの材料にもなる |
| 全件に戻すスイッチ | `affected` は `--base` 以外の引数を読まない（`src/runtime.ts` L113-118）。hook が足すファイルのパスを受け取る導入先があるため、とコメントが書く。全件を走らせるには `globalChanges` か全体 check を使う | 実行時に全 quality へ広げる指定。環境変数なら、引数を読まない理由に当たらない |
| 過去の不具合で選択器を確かめる | #188（open）は、導入先1件の CI で落ちたテストを `check:affected` が選んでいたかを数える。Vitest に限り、Bun は過去の commit で選択を再現できないと書く | 記事は、選ばれたうえで落ちることまで確かめる。#188 が見落とし0件で終わっても、件数が少なければ見落とし率の上限は約 3/N にとどまる |
| 見落としを測り続ける | 無い。#188 は1回の計測である | 全体 check か CI で全 quality を走らせるときに、`check:affected` が選んだはずの組を記録し、落ちたテストと比べる。feedback-loop は全体 check と CI を自分でつながない（`references/contract.md` L97）。入れるなら、feedback-loop が記録のコマンドを配り、導入担当がつなぐ形になる |

### 制約にかかわるもの

- 理由や選んだ組をファイルに残すなら、Constraints の「Git のディレクトリへ書かない」に従う。
- CI で選んだはずの組を記録し続けることは、Non-goals の「常駐監視」に当たらないと、feedback-loop の利用者が判断した（2026-11-18）。

### 今の範囲の外にあるもの

- 検査を段階ごとに分けること。feedback-loop は CI を入れない。全体 check と CI は導入担当が既存の構成へつなぐ（`references/contract.md` L97）。入れられるのは、導入する agent への案内だけである。
- テスト行数の上限。OUTCOME の Behavior にテストの量は無い。feedback-loop の利用者は、入れないと判断した（2026-11-18）。理由は、大事なのは行数ではなく、テストが必要なものになっているかだからである。記事も、上限は有用なテストを消す理由ではないと書き、必要かの判定は test-audit スキルの規則（守る振る舞いを名指しし、壊れたら落ちる不具合を示す）に任せる。記事は上限を、増え方を見えるようにし、例外に理由を書かせる検査として置く。#187（closed）と #190（open）は、既存のテストがすり抜ける壊し方を一覧にする Issue で、テストを足す向きである。

### すでに同じ向きのもの

- 分からなければ広げる。lockfile、`package.json`、`tsconfig.json`、設定の変更と、`globalChanges` に当たる変更は全 quality を実行する（`src/config.ts` L157-163、L171-173）。Bun で記録の無い quality は常に選ぶ（`src/bun.ts` L153-154）。一方、`ignoredChanges` と assurance だけの変更は、テストを走らせずに成功で終わる（`src/runtime.ts` L318-324）。宣言した変更に限った例外である。
- import に見えない入力は宣言で補う。`dependencies` が担当の対応表である（`src/config.ts` L184-188）。#162 は、ファイルを読む呼び出しから登録漏れを検出する方法を試し、誤検出は導入先で 9件中4件、この repo で 19件中18件だった。#162 は対応しないと決めた。理由は、漏れても全体 check と CI で見つかること、選ばれないまま成功するには入力が別の quality から import されている必要があり導入先の未登録4件は当たらなかったこと、誤検出の多さである。
- 速い道具。#199（open）は、型認識の lint の backend を tsgolint から tsrs に替える。

## 照合の結果

- **算術：** 「記事が挙げる数字」の表で「一致した」とした値は、記事の付録の式と元の値から再計算した。
- **openclaw の版：** 「考え方と根拠」で行番号を挙げた記述は、`a7d05c7` で確かめた。
- **feedback-loop：** 行番号を挙げた記述は、`d9898c8` で確かめた。Issue は #162、#188 の本文と、#162 の最後のコメントを読んだ。#187、#190、#199 は題名と状態だけを見た。
- **読んでいないもの：** 記事が引く PR の本文。記事の数字の元になった行ごとのデータ（公開されていない）。ClawSweeper と AutoReview の実物。
- **照合していないもの：** CI 時間と PR から merge までの中央値。Rust のジョブの81秒。SimSlim の効果（記事も数字を付けていない）。

## 照合の手順

1. 記事を取得する：`scout fetch https://vincent.build/openclaw-ci-cost-reduction/`
2. テスト行数の上限を読む：`scout repo-read openclaw/openclaw docs/ci/local-proof.md --ref a7d05c7af04aa17ac57c994da238d0c60f058ca6 -l 330-345`
3. 選んだ理由の規則名を探す：`scout repo-read openclaw/openclaw scripts/test-projects.test-support.mts --ref a7d05c7af04aa17ac57c994da238d0c60f058ca6 | ugrep -n 'rule: '`
4. fallback と残す検査を探す：`scout repo-read openclaw/openclaw scripts/lib/ci-changed-node-test-plan.mts --ref a7d05c7af04aa17ac57c994da238d0c60f058ca6 | ugrep -n -e 'policy-watch' -e 'fixed-smoke' -e 'shard metadata'`
5. 全件に戻す変数を探す：`scout repo-read openclaw/openclaw docs/ci/adaptive-testing.md --ref a7d05c7af04aa17ac57c994da238d0c60f058ca6 | ugrep -n NODE_SELECTION`
6. 定期実行と push の条件を探す：`scout repo-read openclaw/openclaw .github/workflows/ci.yml --ref a7d05c7af04aa17ac57c994da238d0c60f058ca6 | ugrep -n -e cron -e OPENCLAW_CI_ON_PUSH`
7. 算術を再計算する。例：`python3 -c "p=296090.104/7*30; l=13375.542/7*30; print(p, l, 1-l/p)"`
8. feedback-loop の版を読む：`git -C <feedback-loop の checkout> show d9898c8:src/selection.ts | sed -n '30,40p;74,118p'` と、同じ形で `src/runtime.ts` の L110-120 と L295-326 を読む。
9. Issue を読む：`gh issue view 162 --repo thkt/feedback-loop --json comments --jq '.comments[-1].body'` と `gh issue view 188 --repo thkt/feedback-loop`

## 見直す条件

- OpenClaw が、学習モデルか履歴にテストを飛ばす権限を与えたとき。見落としの予算と測った見落とし率を公開したとき。
- 著者が、絞り込みを入れた後の PR から merge までの時間を示したとき。
- feedback-loop で、選んだ理由の記録、全件に戻す指定、見落としの継続的な計測のどれかを Issue にしたとき。
- feedback-loop#188 の計測の結果が出たとき。
