# openclaw の test-audit スキルの考え方と根拠（2026-11）

openclaw の test-audit スキルは、エージェントが書くテストの価値を判定し、価値の低いテストを削るための指示である。その考え方と、スキルが根拠にした Telegram のテスト削減の PR を照合した。自分たちが採用した判断ではない。

## TL;DR

- 1つの契約は、最も強い境界にいる1つのオーナーのテストだけが証明する。
- テストの削除にも証拠を求める。削除候補ごとに、消しても残る証明を名指しする。
- 根拠の PR の数値は、再計算ですべて一致した。行数は、PR 自身の数え方による自己申告である。
- 手元の設定との差分を挙げたが、どの設定にも入れていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | openclaw/openclaw の test-audit スキル（3ファイル）と、その根拠になった PR 3件・Issue 1件 |
| 著者・発行元 | openclaw のメンテナ。CAMPAIGN.md を加えた PR #156316 の作者は obviyus |
| 公開日 | 記載なし（随時更新されるファイル）。初版は `a8714eea9a`（2026-08-10）、CAMPAIGN.md の初出は #156316（2026-09-23 merge）、最終変更は `3195fba906`（2026-10-19） |
| 照合した版 | main の `28d20f9532`（2026-11-13）。スキルのディレクトリは `3195fba906` 以降変わっていないため、この版の本文は最終変更時と同じ |
| 照合日 | 2026-11-13 |
| 照合した範囲 | スキルの本文、スキルがリンクするファイルの実在、PR とレビューに書かれた件数と数値の再計算 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 種別 |
| --- | --- | --- |
| SKILL | [.agents/skills/test-audit/SKILL.md](https://github.com/openclaw/openclaw/blob/28d20f9532f4a5c4f20e5ad85a1ea370444886f6/.agents/skills/test-audit/SKILL.md) | スキル本体。テストを書くときの判定と、削除の基準 |
| audit | [.agents/skills/test-audit/references/audit.md](https://github.com/openclaw/openclaw/blob/28d20f9532f4a5c4f20e5ad85a1ea370444886f6/.agents/skills/test-audit/references/audit.md) | 削除候補の証拠と検証の手順 |
| CAMPAIGN | [.agents/skills/test-audit/CAMPAIGN.md](https://github.com/openclaw/openclaw/blob/28d20f9532f4a5c4f20e5ad85a1ea370444886f6/.agents/skills/test-audit/CAMPAIGN.md) | サブシステム全体のテストを1つの PR で削る手順 |
| #155040 | [refactor(telegram): remove redundant test coverage](https://github.com/openclaw/openclaw/pull/155040) | Telegram のテスト削減の PR。2026-09-23 merge |
| レビュー | [#155040 の保存レビュー](https://github.com/openclaw/openclaw/pull/155040#pullrequestreview-5278755685) | 削除で失われた契約を探したレビュー。投稿者は roboclaw-bot |
| #156316 | [docs(skills): prevent junk tests and add subsystem test-pruning campaigns](https://github.com/openclaw/openclaw/pull/156316) | #155040 の結果からジャンクパターンと CAMPAIGN.md を加えた PR |
| #168083 | [docs: prefer meaningful integration proof over low-value tests](https://github.com/openclaw/openclaw/pull/168083) | SKILL がリンクする writing-tests.md の節を加えた PR |
| #155003 | [Prune low-value Telegram tests while preserving behavioral contracts](https://github.com/openclaw/openclaw/issues/155003) | #155040 の依頼。neocybereth の prune-low-value-tests スキルを使うよう指示している |

## 中心の主張

**1つの契約は、最も強い境界にいる1つのオーナーのテストだけが証明する。** 同じ契約を別のレイヤーで重ねてテストするのは、そのレイヤーに固有のリスクがあるときに限る。

#156316 は、この規則を足した理由を2つ挙げる。エージェントが実装を言い直すテストを追加し続けること。同じ契約を複数のレイヤーで繰り返すテストを追加し続けること。

もう1つの柱は、削除にも証拠を求めることである。削除候補ごとに、それを消しても残る証明を名指しする。証拠が揃わなければ消さない。#155040 のレビューは、「ほかで覆われている」という判断の粒度が粗いと、契約が唯一の証明を失うことを9件の実例で示した。

## 考え方と根拠

| 考え方 | 資料と根拠 | 照合の結果 |
| --- | --- | --- |
| テストを足す前に4つの問いに答える。守る振る舞いは何か。どんな現実的な回帰で落ちるか。既存の証明がなぜそれを拾えないか。本番の呼び出し元がない export・flag・注入口が要るか | SKILL の Authoring gate | 原文に記述がある。#156316 の試行では、`expected = matchRule(input)` と書くテスト案が却下された。期待値を被テストの helper から取り、テスト専用の export も要るためである |
| 1つの契約は最も強い境界の1オーナーが証明する。バグ修正の回帰テストはオーナーに1本だけ足し、通過する各レイヤーには足さない | SKILL。#156316 | 原文に記述がある |
| ジャンクパターンを拒否する。例：自己比較、在庫や export 一覧のコピー、ソースの grep、呼び出し形のアサート、契約の重複、テスト専用の seam、被テストコードが作った期待値、アサートする挙動を実装している mock | SKILL の Junk patterns。audit の Junk patterns | 原文に記述がある。レビューは、期待値を被テストの helper で生成していた spool テストが、独立した literal の ID に直された例を挙げる |
| ジャンクパターンのうち次の6種は、#156316 が足した。被テストコードで計算した期待値、アサートする挙動を実装する mock、オーナーが作るべき順序を捏造する fixture、宣言した capability flag を言い直すテスト、無関係な理由で通る否定テスト、検証しない挙動を約束するテスト名 | #156316 は、この6種を #155040 の結果から取り出したと書く | 原文に記述がある。レビューは「無関係なガードがすべてを拒否するために通る拒否テストがある。最も近い有効なケースと組にする」と書く |
| 独立した契約は残す。例：API、SDK、プロトコル、設定、移行、ストレージ、セキュリティ、既定値。静的なテストや遅いテストであることは削除の理由にならない | SKILL と audit の Retention bar | 原文に記述がある。レビューは、`disableAudioPreflight` のスキーマのテストの削除を「設定の形は公開契約で、自動的に literal slop になるわけではない」として差し戻した |
| ソースを検査するテストは、識別子だけの改名で壊れず、ユーザーに見えるキーを守る場合に限って残す | SKILL。audit | 原文に記述がある。#156316 の試行では、定数名を grep する案が「識別子の改名で壊れ、ユーザー向けのキーの改名では通る」として却下された |
| ベースラインで落ちるテストは製品のバグであり得る。テストを消さず、オーナー側を直す | SKILL。CAMPAIGN は「Telegram では3件とも実際の配信バグだった」と書く | 一部一致。#155040 は3件を消さずに残した。ただし PR が欠陥と書くのは2件である。食い違う点の1に書く |
| 削除の前に、候補ごとに次の5つを記録する。名前と検出できる失敗、本番側の呼び出し元、残るより強い証明、存在する理由、検証コマンド。証拠が欠ければ消さない | audit の Candidate evidence。レビューは「意味のある削除ごとに、残すテストの名前と失敗のしかたを対応させる」と書く | 原文に記述がある。レビューは、「ほかで覆われている」の粒度が粗いことを繰り返し起きた問題として挙げる。例：共有 helper のテストは、呼び出し元が正しい metadata を渡すことを証明しない |
| テスト専用の本番 seam は、テストと一緒に消す | SKILL。audit。#155040 は、`message-cache.ts` のテスト用の store・bucket 引数、`update-offset-persistence.ts` の検査用 getter、`resetMissingDefaultWarnFlag` の export、`action-runtime.ts` の間接層を消した | PR 本文に記述がある。差分そのものは照合していない |
| サブシステム全体を削るときは8段階で進める。ベースライン、オーナー境界でのレーン分け、レーンごとの読み取り専用の台帳、重複レイヤーの計画、切り替え、保存レビュー、製品欠陥の修正、main との調整 | CAMPAIGN。#156316 は、この手順を #155040 から取り出したと書く | 原文に記述がある。#156316 では、読み取り専用のエージェントが `extensions/policy` で手順2〜4を試した。329宣言を R 296・C 26・D 7 と判定したと報告する |
| 台帳では各テスト宣言に4つの印のどれかを付ける。R は残す、F は契約を残してアサーションを直す、C は別のオーナーへまとめる、D は消す。D には残る証明を書く | CAMPAIGN | 原文に記述がある。#156316 は、試行で見つかった空振りの否定テスト5件から F を足したと書く。どれも `arrayContaining` で、1項目だけ漏れても通るものだった |
| テストは名前ではなくアサーションで判断する | CAMPAIGN。進捗ウィンドウを退役させる名前のテストが、ウィンドウが消えないことをアサートしていた例を挙げる | 例は未照合。#155040 の本文、レビュー、コメント、コミットに該当する記述が見つからなかった |
| 削除の後、独立したレビュアーが、削除した範囲と残すテストを比べる。唯一の証明を失った契約を探す | CAMPAIGN は「Telegram のレビューは実際の欠落9件と、到達しないアサーション1件を見つけた」と書く | 欠落9件は一致した。レビューの主要5件と補足4件で9件で、PR の修正一覧も9項目である。「到達しないアサーション1件」は、#155040 の本文、レビュー、コメント、コミットに該当する記述が見つからず未照合 |
| 復元した契約ごとに本番コードを1か所わざと壊し、テストが赤になるのを確かめる。その後、ソースをバイト単位で元に戻す | CAMPAIGN。レビューは「緑であることだけでなく感度を確かめる」と提案した | #155040 は、10個の mutation がすべて検出されたと書く。件数は PR の自己申告で、再現していない。レビューの「Original review」の節は、テストも mutation も実行していないと書く。同じ本文の追記の節は、4件のテストを回し、`setThinking()` を消すとテストが落ちることを確かめたと書く |
| 削除した行数を目標にしない | SKILL は「never chase deletion counts」と書く。レビューは「削除した LOC は結果として扱い、目標にしない」と書く | 原文に記述がある。#156316 は、試行の判定が D 7件（329宣言中）だったことを、削除を押し付けない証拠に挙げる |
| 振る舞いは所有する境界で証明する。外部 I/O は固定の fixture にし、期待値は独立に書く | SKILL がリンクする writing-tests.md の「Prove behavior at the owning boundary」 | 節は照合した版に実在する。#168083 は、ユニットテストが見逃した compaction のミス5件を挙げる。5件は PR の自己申告で、照合していない |
| 削除のバッチを入れる前に、`deadcode:dependencies`、`deadcode:unused-files`、`deadcode:exports` を回す。消したテストが孤立させた support ファイルや export は、通常の PR CI では拾われない | audit の Validation and handoff。`fecdb0fb6a` が `deadcode:dependencies` を求め、`3195fba906` が3つすべてに広げた | 3つのスクリプトは、照合した版の `package.json` に実在する。PR CI が拾わなかった実例は照合していない |

## 資料の間で食い違う点

1. **ベースラインで落ちたテストのうち、製品のバグだった件数。** CAMPAIGN は「all three were real delivery bugs」と書く。
   - #155040 の「Earlier pruning-head evidence」は、落ちた3件を挙げる。進捗の tool-result の受理がタイムアウトする、quote-off で2通送られる、message-tool の進捗が余分なメッセージを残す、の3件で、消さずに残したと書く。
   - 同じ PR の User Impact が欠陥と書くのは2件である。message-tool の final の後に古い回答ブロックが送られる欠陥と、quote-off の欠陥。
   - 「Landing preparation」は、3件のオーナーが通るようになった手段として、2つの runtime の修正と、fixture・テスト側の修正を挙げる。3件目のタイムアウトが製品の欠陥だったかは、本文から特定できない。
2. **どの境界で証明するか。** SKILL は、1つの契約を「最も強い境界」で証明するよう書く。CAMPAIGN も、mock した協調オブジェクトより、偽のネットワークをつないだ本物の輸送層を優先する。
   - レビューは「最も高い境界ではなく、足りる中で最も安い境界を選ぶ」と書く。小さな parser・設定・helper のテストも公開契約を守れる、HTTP のシミュレーションは Telegram の規則をすべて独立に確かめるわけではない、という理由である。
   - SKILL の Retention bar は、設定や既定値などの契約を独立に残すため、この差の一部を吸収している。

## 手元の設定との差分（未採用）

この節は、照合者が `~/.claude` の設定と突き合わせて組み立てた案である。どの設定にも入れていない。`~/.claude` の行番号は 2026-11-13 時点のものである。

次の項目は、手元の設定に既にある。

| test-audit の項目 | 手元の設定 |
| --- | --- |
| tautology、mock した被テストコード、重複アサーション、実装への結合 | `~/.claude/agents/reviewers/reviewer-coverage.md:56-61` の Anti-patterns |
| 被テストのモジュールを mock しない。テスト専用の本番メソッドを作らない | `~/.claude/skills/use-workflow-tdd-cycle/references/writing-tests.md:31`、`:35-36` |
| 唯一の証明と回帰テストは残す。ほかで覆われているときだけ消す | `~/.claude/agents/enhancers/enhancer-code.md:55-56`、`:65` |
| 契約を固定するテストは消さず、具体的な literal に書き直す | `~/.claude/agents/_lib/simplification-rules.md:43` |

次の4つは、手元の設定に無い。

| # | test-audit の項目 | 手元の設定で欠けている点 | 入れ先の候補 |
| --- | --- | --- | --- |
| 1 | 1つの契約は1つのオーナーが証明し、回帰テストは各レイヤーに足さない | 重複の判定は同じレイヤーの中だけを見る（`simplification-rules.md:13-14`）。同じ契約を複数のレイヤーで繰り返すテストを止める規則が無い | `writing-tests.md`、`reviewer-coverage.md` |
| 2 | #156316 が足した6種のジャンクパターン | `reviewer-coverage.md:56-61` にも `simplification-rules.md:13` にも無い。テスト名は曖昧さ（`simplification-rules.md:36`）だけを見て、名前とアサーションのずれは見ない | `reviewer-coverage.md` の Anti-patterns |
| 3 | 削除候補ごとの証拠の記録と、証拠が欠けたら消さない規則。ベースラインで落ちるテストは削除候補でなくバグ候補として扱う | `enhancer-code.md:65` の確認は、ugrep でほかのテストを探すことだけ | `enhancer-code.md` の Step 5 |
| 4 | 本番コードを1か所壊して、残すテストが赤になるのを確かめる | 「実装が壊れても通るテスト」（`reviewer-coverage.md:3`）の判定は静的な推論だけ | assert ワークフローの動的な証拠、または critic-evidence |

CAMPAIGN の8段階はこの案に含めない。CAMPAIGN は、サブシステム全体のテストを1つの PR で削る作業の手順である。照合者は、手元でその作業が予定されていることを把握していない。openclaw に固有の手順も入れない。例：Vitest の checkout の保護、`scripts/check-changed.mjs`、`pnpm deadcode:*`、maintainer の landing 手順。

### 採用前に測るもの

| 指標 | 判断すること | 測り方 |
| --- | --- | --- |
| 手元のリポジトリの既存テストのうち、差分2のパターンに当たる宣言の数 | `reviewer-coverage.md` に行を足すか | リポジトリを1つ選び、テストの宣言ごとに audit の Junk patterns と照らす |
| 同じ契約を2つ以上のレイヤーで証明している箇所の数 | 差分1の規則を足すか | 同じリポジトリで、契約ごとにそれを検証するテストの層を数える |
| enhancer-code が消したテストのうち、唯一の証明だったものの数 | 差分3と4を足すか | enhancer-code の削除差分ごとに、消したテストが守っていた本番コードを1か所壊し、残るテストが赤になるかを見る |

## 照合の結果

- **スキルの版：** 照合した main の `28d20f9532` で、スキルのディレクトリの最終コミットは `3195fba906`（2026-10-19）だった。
- **リンク先の実在：** 照合した版に次のものが実在する。`docs/help/testing/writing-tests.md` とその「Prove behavior at the owning boundary」節。`.agents/skills/openclaw-testing/SKILL.md` とその「Proof policy」節。`scripts/check-changed.mjs`。`package.json` の `deadcode:dependencies`、`deadcode:unused-files`、`deadcode:exports`。
- **再計算した数値：** すべて一致した。
  - テストコードの削減率：46,471 / 111,122 = 41.8%
  - テストと support を合わせた削減率：48,278 / (111,122 + 7,493) = 40.7%
  - #156316 の試行の台帳：296 + 26 + 7 = 329
  - レビューの欠落：主要5件 + 補足4件 = 9件
- **数値の性質：** #155040 の行数は、PR 自身が「campaign measurements, not GitHub's PR diff statistics」と書く自己申告である。GitHub の差分の統計は +15,424 / −64,067、280ファイルで、行数の数え方が違う。
- **自己申告で再現していないもの：** #155040 の mutation 10件の検出、#168083 の compaction のミス5件、#156316 の試行の判定結果。
- **見つからなかったもの：** CAMPAIGN が挙げる「到達しないアサーション1件」と「進捗ウィンドウ」のテスト名の例。#155040 の本文、レビュー、コメント、コミットを検索して該当する記述が無かった。
- **コード検索で出なかったもの：** #156316 の試行が挙げる例のうち、「reports policy findings」という名前のテストは GitHub のコード検索でヒットしなかった。改名や索引の漏れがあり得るため、存在しないとは判断せず未照合とした。
- **出自：** #155003 は、#155040 の作業に neocybereth の prune-low-value-tests スキルを使うよう指示している。CAMPAIGN はその作業の後に取り出されたものである。prune-low-value-tests の本文は未照合。
- **レビューの立場：** 保存レビューの投稿者は roboclaw-bot である（GitHub のアカウント種別は User）。欠落9件を挙げた「Original review」の節は、テストも mutation も実行せず、ソースを読んで判定したと自ら書く。同じ本文の先頭にある追記の節は、4件のテストを回し、否定の対照を1件確かめたと書く。

## 照合の手順

1. スキルのディレクトリのコミット履歴を読む：`gh api 'repos/openclaw/openclaw/commits?path=.agents/skills/test-audit&per_page=50' --jq '.[] | [.sha[0:10], .commit.committer.date, (.commit.message|split("\n")[0])] | @tsv'`
2. スキルの3ファイルを照合した版で読む：`gh api 'repos/openclaw/openclaw/contents/.agents/skills/test-audit/SKILL.md?ref=28d20f9532f4a5c4f20e5ad85a1ea370444886f6' -H 'Accept: application/vnd.github.raw'`。`references/audit.md` と `CAMPAIGN.md` も同じ方法で読む。
3. リンク先を同じ方法で読む。`writing-tests.md` と `openclaw-testing/SKILL.md` の節の見出しを探す。
4. deadcode のスクリプトを探す：`gh api 'repos/openclaw/openclaw/contents/package.json?ref=28d20f9532f4a5c4f20e5ad85a1ea370444886f6' -H 'Accept: application/vnd.github.raw' | grep -o '"deadcode:[a-z-]*"' | sort -u`
5. PR の本文を読む：`gh pr view 155040 -R openclaw/openclaw --json body --jq .body`。User Impact、Landing preparation、Review follow-up、Earlier pruning-head evidence の節を見る。#156316 と #168083 も同じ方法で読む。
6. 保存レビューを読み、欠落を数える：`gh api repos/openclaw/openclaw/pulls/155040/reviews/5278755685 --jq .body`。「Important gaps」の見出し5つと、「Additional narrower preservation gaps」の箇条4つを数える。
7. CAMPAIGN の例を探す。`gh api --paginate 'repos/openclaw/openclaw/pulls/155040/reviews?per_page=100' --jq '.[].body' | grep -iE 'unreachable|not cleared|progress window'` を実行する。`issues/155040/comments` と `pulls/155040/comments` にも同じ検索をかける。コミットは `gh pr view 155040 -R openclaw/openclaw --json commits --jq '.commits[].messageBody'` で読む。
8. 数値を再計算する：`python3 -c 'print(46471/111122, 48278/(111122+7493), 296+26+7)'`
9. 試行の例を探す：`gh search code 'reports policy findings' --repo openclaw/openclaw`

## 見直す条件

- test-audit のディレクトリに `3195fba906` より後のコミットが入ったとき。
- openclaw が Telegram 以外のサブシステムでキャンペーンを行い、結果を公開したとき。#156316 は `extensions/discord` を例に挙げている。
- 手元の `reviewer-coverage.md`、`enhancer-code.md`、`writing-tests.md`、`simplification-rules.md` を変えたとき。「手元の設定との差分」の行番号と差分が古くなる。
