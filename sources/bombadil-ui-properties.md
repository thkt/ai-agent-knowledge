# Bombadil の、操作の順番を決めずに UI を探索し、条件を破った操作列を短くする考え方と根拠（2026-11）

Antithesis の Bombadil は、Web とターミナルの UI を自動で操作し、書いた条件を破る操作列を探す道具である。README とマニュアル v0.8.0 の記述を照合した。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- 中心の主張は、テストに操作の順番を書かず、状態の列が満たすべき条件だけを書けば、道具が操作を生成して条件を破る列を探す、というものである。マニュアルの例は「画面に出る通知は常に 5 件以下」を TypeScript で書く。
- 条件を破った操作列は、元と同じ設定で再生を試せる。実行が元の列から逸れると、再生は失敗する。`shrink` は、同じ条件を破る短い列を探す。マニュアルは `shrink` を実験的と書く。
- 見つかった列は不具合の証拠になる。探索が通っても、試していない列については何も言えない（照合者の判断）。
- 照合したのは文書だけである。探索、再生、短縮は手元で実行していない。照合日は 2026-11-23。
- 自作のエージェントハーネス（Cantle）への当てはめは、[notes/test-combination-for-cantle-cancel.md](../notes/test-combination-for-cantle-cancel.md) に分けた。

| 項目 | 内容 |
| --- | --- |
| 出典 | [antithesishq/bombadil](https://github.com/antithesishq/bombadil/tree/v0.8.0) の README と、[The Bombadil Manual](https://antithesishq.github.io/bombadil/) |
| 著者・発行元 | Antithesis |
| 公開日 | v0.8.0 のリリースは 2026-11-19。マニュアルのページに公開日の記載は無い |
| 照合した版 | tag `v0.8.0`。マニュアルの表紙も v0.8.0 と表示していた |
| 照合日 | 2026-11-23 |
| 照合した範囲 | README の全文。マニュアルのうち、既定の条件と操作の生成、`extract`、時間の演算子、通知数の例、条件を破った列の再生と短縮の節 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**テストに書くのは操作の手順ではなく、状態の列が満たすべき条件である。** 操作は道具が生成する。そのため、書き手は「条件を破る操作」を前もって知らなくてよい。README は Bombadil を、Web とターミナルの UI のための property-based testing と説明する。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 手順ではなく条件を書く | 通知数の例は、`extract` で DOM の `.notification` の数を読み、`always` で「5 以下」を条件にする | 記述がある（[03-specification-language.md L717-L731](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/03-specification-language.md?plain=1#L717-L731)） |
| 条件は、各時点のものにも、先の状態との関係にも書ける | 時間の演算子に `always`（今とこの先のすべての状態）、`next`（次の状態）、`eventually`（今かこの先のどこか）がある | 記述がある（[L271-L276](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/03-specification-language.md?plain=1#L271-L276)） |
| 条件が読むのは、ブラウザの状態である | `extract` に渡す関数は、ブラウザの状態を受け取る。主なものは `document` と `window` である。ターミナルでは、描画された画面の内容やカーソルの位置を読む | 記述がある（[L195-L201](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/03-specification-language.md?plain=1#L195-L201)） |
| 操作の生成は既定のものに任せられる | たいていの Web アプリで使える既定の条件と操作の生成器を同梱し、仕様のファイルから再 export して使う。ターミナル向けの既定もある | 記述がある（[L80-L98](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/03-specification-language.md?plain=1#L80-L98)） |
| 条件を破った列を短くする | `shrink` は、条件を破った列を受け取り、同じ条件を破る短い列を探す。最初に、再現が安定するかを調べる段階がある。元の `test` や `fuzz` と同じ設定で実行する。再生は成功が保証されず、実行が逸れると失敗する | 記述がある（再生は [04-workflows.md L28-L31](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/04-workflows.md?plain=1#L28-L31)、短縮は [L33-L65](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/04-workflows.md?plain=1#L33-L65)）。実験的で、オプションと出力の形式は変わりうると書く（[L41](https://github.com/antithesishq/bombadil/blob/v0.8.0/docs/manual/src/04-workflows.md?plain=1#L41)） |
| 手元でも CI でも動く | 手元の開発環境、CI、Antithesis の中で動く | 記述がある（[README](https://github.com/antithesishq/bombadil/blob/v0.8.0/README.md)） |

## 限界

- **文書の照合だけである。** 探索が実際に不具合を見つけるか、再生と短縮が安定するかは、手元で確かめていない。
- **条件に書いていない不具合は拾えない。** 通知数の条件だけでは、必要な通知が出ない不具合も、裏の処理の正しさも判定できない。何を違反とするかは、書き手が仕様から決める（照合者の判断）。
- **`shrink` は実験的である。**

## 照合の結果

- **照合した主張：** 「考え方と根拠」の各行は、引いた行に記述がある。
- **読んでいないもの：** マニュアルのほかの節（導入、インストール、TypeScript と CLI のリファレンス）、`examples/` の仕様、Rust のソース。
- **範囲外：** Antithesis の中で動かす使い方。

## 照合の手順

1. README を読む：`scout repo-read antithesishq/bombadil README.md --ref v0.8.0`
2. 通知数の例と、既定の条件と操作の生成の節を読む：`scout repo-read antithesishq/bombadil docs/manual/src/03-specification-language.md --ref v0.8.0`
3. 再生と短縮の節を読む：`scout repo-read antithesishq/bombadil docs/manual/src/04-workflows.md --ref v0.8.0`
4. 公開されているマニュアルの版を確かめる：`scout fetch https://antithesishq.github.io/bombadil/`。表紙に版が出る。
5. リリース日を確かめる：`gh release view v0.8.0 -R antithesishq/bombadil --json publishedAt`

## 見直す条件

- 手元で Bombadil を動かしたとき。結果は `knowledge/` に書く。
- `shrink` が実験的でなくなったとき。
