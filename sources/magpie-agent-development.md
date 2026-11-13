# Magpie の agent 主導の開発のアプローチ（2026-11）

Magpie は、コードの実装、レビュー、マージ、リリースを agent が担う OSS である。品質を保つために使っているアプローチを、repo の文書と照合した。自分たちが採用した判断ではない。どのプロジェクトにも合うものではなく、合う条件をあわせて書く。

| 項目 | 内容 |
| --- | --- |
| 出典 | [yetone/magpie](https://github.com/yetone/magpie) の agent 向け文書と、それを紹介した [@xulijian1994 の投稿](https://x.com/xulijian1994/status/2119956996279669088) |
| 著者 | yetone（repo）。投稿は第三者による紹介 |
| 公開日 | repo の作成は 2026-09-23。投稿は 2026-11-10 |
| 照合した版 | `main` の `0051781`（2026-11-13） |
| 照合日 | 2026-11-13 |
| 照合した範囲 | `AGENTS.md`、`LESSONS.md`、`docs/code-standards.md`、`docs/contributors.md`、投稿の本文 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 種別 |
| --- | --- | --- |
| 投稿 | [@xulijian1994 の投稿](https://x.com/xulijian1994/status/2119956996279669088)（`xr tweet` で取得） | 3本の `/loop` による自動開発の紹介 |
| AGENTS | [AGENTS.md](https://github.com/yetone/magpie/blob/00517810444931e38c4c06e9dd69f8c435ff0d14/AGENTS.md) | agent が作業前に読む文書の案内 |
| LESSONS | [LESSONS.md](https://github.com/yetone/magpie/blob/00517810444931e38c4c06e9dd69f8c435ff0d14/LESSONS.md) | マージ済みの作業から得た教訓 |
| standards | [docs/code-standards.md](https://github.com/yetone/magpie/blob/00517810444931e38c4c06e9dd69f8c435ff0d14/docs/code-standards.md) | 受入基準とレビューの手順 |
| contributors | [docs/contributors.md](https://github.com/yetone/magpie/blob/00517810444931e38c4c06e9dd69f8c435ff0d14/docs/contributors.md) | 外部からの PR の扱い |

## 中心の主張

**agent が書き、レビューし、マージする開発でも、3つの仕組みで品質を保つ。**

1. 受入基準：何を実行し、何が出力されたかを証拠として示さない変更はマージしない。
2. 教訓の昇格：毎晩その日の commit を振り返り、繰り返し見つかった失敗を規則に格上げする。
3. 害の先行確認：外部からの入力は、中身を評価する前に害が無いかを確かめる。

投稿によれば、作業は Claude Code と Opus 5.5 で動く3本の `/loop` が回す。

- 1本目：GitHub と Discord を見張り、要求への対応からリリースまでを行う
- 2本目：X の製品へのフィードバックを安いモデルでふるい、Opus 5.5 に渡す
- 3本目：定期的にコードを整理し、教訓を `LESSONS.md` に書く

## アプローチと根拠

| アプローチ | 仕組み | 照合の結果 |
| --- | --- | --- |
| 実行した証拠でマージを決める | standards の Acceptance criteria。何を実行し何が出力されたかを書かせ、実行していない確認は「実行していない」と書かせる。省略は許さない | 原文に記述がある |
| 変更が無いと失敗するテストを示す | standards の受入基準1。古いコードに戻して新しいテストを実行し、振る舞いの名前が付いた本物の `FAIL` を確かめる。コンパイルエラーやスキップは数えない | 原文に記述がある |
| テストを隔離した HOME で動かす | 受入基準2。テストが `~/.codex` や `~/.claude` などの実際の設定を読み書きしないようにする | 原文に記述がある |
| 選んだパッケージではなく全体を検査する | 受入基準3は `./...` を求める。変更が触れないパッケージにも、全体を守るテストがあるため | 原文に記述がある。10-13 に、選んだパッケージだけを実行した4つの commit が main を赤にした記録がある |
| レビューでは、わざと壊して確かめる | standards の Reviewing a PR。修正を戻すか、守っているコードを何通りかに壊し、新しいテストがそれぞれを拾うかを見る | 原文に記述がある。#901 で `backup.go` を4通りに壊した例を挙げる |
| 失敗を、見つかった日数で規則に格上げする | LESSONS の冒頭。毎晩その日の commit を振り返り、教訓を「規則、理由、証拠」で書く。`Seen` に見つかった日数を数え、別々の3日で見つかった教訓は standards に移す | 原文に記述がある。10-07 に4つ、10-08 に2つを移した記録がある。移した後もレビューで各規則を確かめると書く |
| 外部の入力は害を先に確かめる | contributors の Look for harm first。次の変更を含む PR は、メンテナが見るまでレビューを止める：`.github/` やリリース用のスクリプト、新しい通信先、認証情報、依存の追加、バイナリ | 原文に記述がある。メンテナの承認前に、実アカウントのあるマシンで PR のコードを動かすことも禁じる。PR を開けるのは collaborator だけで、外部からの修正は Issue で受け取る |
| 正本を明記する | AGENTS。例：`internal/provider/migrate*.go` の `movers` が正本で、`TestMovedBuiltinsSayTheirPlugin` がずれを検出する | 原文に記述がある |

## 合うプロジェクトの条件

ここに挙げる条件は、照合者が LESSONS の記録から読み取ったものである。Magpie 自身は書いていない。

| 条件 | 理由 |
| --- | --- |
| 変更の数が多い | 教訓の昇格は、同じ失敗が別々の日に3回見つかって初めて働く。LESSONS は、10-05 に220 commit と約170回のリリースがあったと書く |
| 対象の環境が多い | 受入基準3と4は、3つの OS と2つのブラウザエンジンでの検査を求める。こうした組み合わせは、人が手で確かめきれない |
| 外部からの要求が多い | 害の先行確認と、Issue 経由での受け付けが働く。2026-11-13 時点で open の Issue は4094件 |
| 速度と引き換えの手戻りを受け入れられる | LESSONS によれば、10-05 のリリースのうち約40件は、同じ日か前日のリリースを直すものだった |

変更が少ないプロジェクトでは、同じ失敗が3日見つかるまでに時間がかかり、教訓の昇格が働きにくい。失敗のリリースが利用者に直接届くプロジェクトでは、この手戻りの量は許容しにくい。

## 照合できなかったこと

- **3本の `/loop` のプロンプト。** repo では見つからなかった。yetone 本人の投稿は、`xr user-posts yetone` が API の応答の形を解釈できずに読めなかった。
- **X のフィードバックの経路の防御。** contributors は PR を対象にしており、X の投稿を入力にする経路への防御は、照合した文書に書かれていなかった。
- **投稿の内容。** 第三者による紹介である。3本の `/loop` と、3つの OS の実機の検証環境は、repo の文書では確かめていない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた案である。どの設定にも入れていない。

| Magpie のアプローチ | 手元の資料 |
| --- | --- |
| 教訓を見つかった日数で格上げする | `scribe` skill は、過去の PR と Issue から繰り返し現れるパターンを wiki に抽出する。ただし、いつ規則に格上げするかの閾値を持たない。`Seen` の日数は、その閾値の候補になる |
| 変更が無いと失敗するテストを示す | `code` workflow は、Red を確かめられなかった unit を異常として記録する。Magpie はさらに、それを確かめた記録をマージの条件にしている |
| わざと壊して確かめる | [hadome.md](hadome.md) の `hadome mutate` と、[test-audit-openclaw.md](test-audit-openclaw.md) の、本番コードを1か所壊す確認 |
| `/loop` を回し続ける | [stop-hook-done-command.md](../knowledge/stop-hook-done-command.md)。止めどきを agent に任せず、完了判定のコマンドで決める |

## 照合の手順

1. 投稿を読む：`xr tweet https://x.com/xulijian1994/status/2119956996279669088`
2. repo を探す：`gh search repos magpie --owner yetone`
3. 版を固定する：`gh api repos/yetone/magpie/commits/main --jq .sha`
4. 構成を読む：`scout repo-tree yetone/magpie`。`AGENTS.md`、`LESSONS.md`、`.github/workflows/` を探す。
5. 文書を読む：`scout repo-read yetone/magpie AGENTS.md`。`LESSONS.md`、`docs/code-standards.md`、`docs/contributors.md` も同じ方法で読む。
6. LESSONS の昇格の記録を探す：冒頭の段落と「Moved to the code standards」の節を読む。

## 見直す条件

- yetone が `/loop` のプロンプトを公開したとき。
- `LESSONS.md` の昇格の規則（別々の3日）が変わったとき。
- `docs/contributors.md` が、X や Discord の入力への防御を書き足したとき。
- 手元で `scribe` に格上げの閾値を入れたとき。
