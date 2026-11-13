# hadome の考え方と根拠（2026-11）

hadome は、変更の前と後のコードを比べ、その変更が増やした違反と弱めたテストだけを報告する静的チェッカーである。agent が書いたテストを人が1本ずつ見ずに信じるための仕組みとして、その主張を README と設計文書で照合した。自分たちが採用した判断ではなく、手元では動かしていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [rerorets/hadome](https://github.com/rerorets/hadome) |
| 著者 | rerorets |
| 公開日 | リポジトリ作成は 2026-11-03。最新 release は v0.5.0（2026-11-11） |
| 照合した版 | `main` の `7804dd5`（2026-11-13） |
| 照合日 | 2026-11-13 |
| 照合した範囲 | `README.ja.md`、`docs/model.ja.md`。設計文書（`docs/design/`）は一覧だけを見た |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 種別 |
| --- | --- | --- |
| README | [README.ja.md](https://github.com/rerorets/hadome/blob/7804dd5d4fd3a71a17959e68eb92dc45e72010f9/README.ja.md) | 仕組み、ルール、終了コード、agent 向けの指示、他の道具との比較 |
| model | [docs/model.ja.md](https://github.com/rerorets/hadome/blob/7804dd5d4fd3a71a17959e68eb92dc45e72010f9/docs/model.ja.md) | テストを信じるための3つの段 |

## 中心の主張

**テストは本数ではなく、テストが呼ぶ関数で比べる。** hadome は、テストが呼ぶ関数の集合を「守っている範囲」と呼ぶ。変更の後にこの範囲が狭まったら、後退として止める。

| 変更 | テストの数 | 守っている範囲 | 判定 |
| --- | --- | --- | --- |
| 重複した3本のテストを1本にまとめた | 減る | 変わらない | 整理 |
| コードと一緒にそのテストを消した | 減る | 対象と一緒に消える | 整理 |
| テストだけを消した、または弱めた | 減るか同じ | 狭まる | 後退 |

model は、テストを信じるための段を3つに分ける。hadome はその3段目にあたる。

| 段 | 中身 |
| --- | --- |
| 1. 不変条件 | 壊れてはいけないことの短い一覧（10個以内）。守るテストを紐付ける |
| 2. 記載ルール | 信じていいテストの書き方。例：期待値を実装からコピーしない、時刻を読まない、sleep で待たない |
| 3. チェック | 1と2が守られているかを、変更ごとに検査する |

model は「人が見るのは、1つずつのテストケースではなく、1と2の中身」と書く。

## 考え方と根拠

| 考え方 | 資料と根拠 | 照合の結果 |
| --- | --- | --- |
| main にすでにある違反は報告しない | README の概要。毎回 base を検査し直し、結果を保存しない | 原文に記述がある。README の比較表は、結果を記録ファイルに保存する Betterer と比べ、記録の更新が要らないことを違いに挙げる |
| テストを消したことで、触っていない関数が無防備になった場合も拾う | README の比較表。変更した行だけを見る linter（golangci-lint の `--new-from-rev`、reviewdog）では見えない例として挙げる | 原文に記述がある。手元では動かしていない |
| テストを実行せずに、テストが呼ぶ関数を読む | README の比較表の、カバレッジの差分との行 | 原文に記述がある。実行するまで呼び先が決まらない呼び出しは読めず、人の確認（終了コード `3`）に回すと書く |
| LLM を使わず、通信もしない | README の比較表の、AI のコードレビューとの行。使う場面と終了コードの節 | 原文に記述がある。同じ変更には毎回同じ結果を返すと書く |
| 終了コードで agent の次の動きを決める | README。`0` 後退なし、`1` 後退あり、`2` 設定の誤り、`3` 人の確認が要る | 原文に記述がある。agent 向けのブロックは「`3` なら止まって聞け。通すためにコードを変えるな」と書く |
| 例外は理由を付けたときだけ効く | README の例外の書き方。`gate:allow <rule> reason: ...` | 原文に記述がある。agent 向けのブロックは、通すためだけに `gate:allow` を足すことを禁じる |
| ルールを緩める設定の変更は人が確認する | README。`config/loosened` は終了コード `3` になる | 原文に記述がある |
| 呼ぶだけで確かめないテストは別のルールと mutation で拾う | README の検査するものの表。`no-assertion`、`always-true`、`guards-nothing`。`hadome mutate` は変更した関数を1か所ずつ壊す | 原文に記述がある。mutation は壊し方ごとにテストを1回流すため、`check` よりずっと時間がかかると書く |
| 8言語を1つのバイナリで読む | README。TypeScript、Python、Java、Kotlin、Go、Rust、Terraform、Ansible | 原文に記述がある。Go から入れて C コンパイラが無い場合は、Go、Terraform、Ansible だけになる |
| agent が読むファイルに使い方を書き込む | README。`hadome agents --write CLAUDE.md` は `<!-- hadome:begin -->` と `<!-- hadome:end -->` の間を書き換える | 原文に記述がある |

## 限界

- **呼んでいることと、確かめていることは別である。** 関数を呼んだまま assert だけを弱めても、守っている範囲は狭まらない。`no-assertion` と `always-true` が一部を拾い、残りは `hadome mutate` に頼る。
- **呼び先が実行するまで決まらない呼び出しは読めない。** 動的ディスパッチの多いコードでは、終了コード `3` が増えると見込む。どの程度増えるかは測っていない。
- **プロジェクトが若い。** 作成から10日で、release は v0.5.0 である。npm への公開は、パッケージ名の審査を待っている（README と Issue #134）。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた案である。どの設定にも入れておらず、実行もしていない。

| 手元の資料 | hadome との関係 |
| --- | --- |
| [hill-climb.md](hill-climb.md) | Hill-Climb はロック対象を sha256 で固め、テストのファイルが変わると採点を拒否する。hadome は、テストを書き換えても守っている範囲が縮まなければ通す。テストのリファクタリングを許したまま、弱体化だけを止められる |
| [test-audit-openclaw.md](test-audit-openclaw.md) | test-audit は、テストを消す前に「残る、より強い証明」を候補ごとに記録させる。hadome の `scope-shrunk` は、そのうち「その関数を呼ぶテストが残っているか」を機械的に検査する。証明の強さまでは見ない |
| [stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) | `hadome check` を完了判定のコマンドにできる。ただし、今の hook は exit 0 以外をすべて差し止める。このままでは exit `3` のときも agent に作業を続けさせ、hadome の「止まって聞け」と食い違う |

Stop hook と組み合わせるときは、`until` に次のような変換を書く案がある。

```sh
hadome check --format json; c=$?
[ "$c" -eq 1 ] && exit 1   # 後退：agent に直させる
exit 0                     # 0 は完了。2 と 3 は人に回すため、止めるのを許す
```

この変換では、止めた理由が人に届かない。hook の側で exit `3` を「止めるのを許し、`systemMessage` で理由を出す」に分けると、人が気づける。どちらも未検証である。

## 照合の結果

- **版：** `main` の HEAD は `7804dd5`（2026-11-13）。最新 release は v0.5.0（2026-11-11）。
- **照合した主張：** 「考え方と根拠」の各行は、README と model に原文がある。
- **動かしていないもの：** `hadome check`、`hadome mutate`、`hadome agents --write`。README の出力例と demo の結果は再現していない。
- **読んでいないもの：** `docs/design/` の各ルールの仕様、`docs/for-agents.ja.md`、`docs/ci.ja.md`。

## 照合の手順

1. 版を固定する：`gh api repos/rerorets/hadome/commits/main --jq .sha`
2. release を読む：`gh api repos/rerorets/hadome/releases --jq '.[0:3][]|.tag_name+" "+.published_at'`
3. README と構成を読む：`scout repo-overview rerorets/hadome` と `scout repo-tree rerorets/hadome`
4. 日本語の README と model を読む：`scout repo-read rerorets/hadome README.ja.md` と `scout repo-read rerorets/hadome docs/model.ja.md`
5. open の Issue を読む：`gh api 'repos/rerorets/hadome/issues?state=open' --jq '.[]|"#\(.number) \(.title)"'`

## 見直す条件

- 手元の repo で `hadome check --base main` を実行したとき。「動かしていないもの」が変わる。
- hadome が v1.0 を出したとき。または終了コードの意味を変えたとき。
- Stop hook の `until` に hadome を使ったとき。「手元の資料との関係」の変換を検証し、[stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) に結果を足す。
