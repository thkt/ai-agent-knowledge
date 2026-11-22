# ACP エージェントの返信にグラフを載せるなら、mpljs で手元に2テーマの SVG を書き、返信に Markdown の画像として書く

## TL;DR

- 自作の ACP エージェント（Cantle）の返信に `![説明](/絶対パス/chart.svg)` を書くと、T3 Code はその SVG を画像として表示した。グラフは mpljs で手元に描き、定義もデータも外へ送らない。
- mpljs の PNG では、日本語の文字が「□」になった。同梱のフォントが DejaVu と Inter の一部の文字だけのためである。SVG は文字を `<text>` で書くので、表示する側のフォントで日本語が出た。
- ライトとダークの両方で読めるよう、同じ図を `pro-slate` と `pro-dark` で描いて1つの SVG にまとめ、`prefers-color-scheme` で切り替えた。背景は透明にし、`font-family` に日本語のゴシック体を足した。
- 確かめていないことは2つある。1つは、`/private/tmp` 以外の場所（`~/.config` など）の SVG を T3 Code が表示するか。もう1つは、T3 Code のテーマを OS と別に切り替えたときに、SVG の中の `prefers-color-scheme` がどちらに従うか。検証日は 2026-11-22、モデルは `claude-opus-5-5`。

## 結論

ACP エージェントの返信にグラフを載せるときは、mpljs で SVG を手元に書き出し、その絶対パスを Markdown の画像として返信に書く。SVG は、2テーマを切り替える形、透明な背景、日本語のゴシック体を含む `font-family` にする。

## 採用した理由

| できるようになったこと | 仕組み |
| --- | --- |
| モデルがグラフの定義を書ける | mpljs の定義は、matplotlib の呼び出しをそのまま並べた JSON である（`[["bar", [["a", "b"], [1, 2]], {}], ...]`）。モデルが書き慣れた matplotlib の API で書ける |
| 定義もデータも外へ送らない | mpljs は依存のない TypeScript で、Bun の中で描ける。描画サービスに URL で定義を渡す必要がない |
| 日本語のラベルを出す | SVG は既定で文字を `<text>` として書く（rc の `svg.fonttype` が `none`）。文字の形は表示する側のフォントで描かれる |
| ライトとダークの両方で読める | 2テーマで描いた図を `<g class="theme-light">` と `<g class="theme-dark">` に入れ、CSS の `@media (prefers-color-scheme: dark)` で片方だけを表示する。背景を透明にしたので、どちらのテーマでも白い四角が浮かない |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| mpljs の公開デモ（`mpl.ivar.workers.dev`）の URL を返信に書く | URL の `/v1/` の後ろは、定義の JSON を raw deflate で圧縮して base64url にした文字列である。2,502 文字の JSON が 1,188 文字になっていた。モデルには圧縮を計算できないので、結局は変換するツールが要る。データも外部の Worker へ送られる |
| PNG で書き出す | 日本語の文字が「□」（NO GLYPH）になり、`Glyph 25505 (採) missing from font(s) Inter` の警告が出た |
| 日本語フォントを mpljs に足す | mpljs はフォントを固定の一覧で持つ（`src/core/fontcodec.ts:19`）。足すには、Python の `gen_fonts.py` でフォントを変換し、一覧を書き換える fork が要る。数百字分の Inter だけで 122 KB あり、数千字が要る日本語フォントはさらに大きくなる |
| 1テーマで、背景を白のまま書き出す | T3 Code のダークモードで、白い背景が浮いて見えた。文字は Inter が無い環境でセリフ体になった |
| `import ... from 'mpl'` で読み込む | mpljs は npm に無い。git 依存で入れても、`dist/` を作る `prepare` は実行されなかった。`package.json` の `exports` は `dist/` だけを指すので、パッケージ名と `mpl/src/index.ts` の import は解決できなかった |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-22 |
| モデル | `claude-opus-5-5`（描画と SVG の作成）。返信に画像を書いたのは T3 Code に登録した Cantle で、モデルは記録していない |
| ツール | Bun 1.4.5、mpljs（`ivarvong/mpl` の commit `9d6e19e`）、T3 Code（版は記録していない）、Darwin 27.2.0（Apple silicon） |

### 再現手順

1. `git clone https://github.com/ivarvong/mpl.git` で mpljs を取得し、commit `9d6e19e` に合わせる。
2. clone の外に置いたスクリプトで、`mpl/tools/replay.ts` の `build` を import する。
3. 日本語のタイトルを含む定義（`style: 'pro-slate'`）から `fig.toPNGSync({ scale: 2 })` で PNG を書き、`fig.toSVG()` で SVG を書く。
4. rc に `{ 'figure.facecolor': 'none', 'axes.facecolor': 'none', 'savefig.facecolor': 'none' }` を渡し、`pro-slate` と `pro-dark` で1回ずつ描く。`toSVG({ idPrefix: 'l' })` と `toSVG({ idPrefix: 'd' })` で、要素の `id` が重ならないようにする。
5. 2つの SVG の `font-family="Inter"` を `font-family="Inter, -apple-system, BlinkMacSystemFont, 'Hiragino Sans', 'Noto Sans JP', 'Segoe UI', sans-serif"` に置き換える。
6. 片方の `<svg>` の開始タグの後ろに、次の `<style>` と2つの `<g>` を入れて、1つの SVG にする。

   ```text
   <style>.theme-dark{display:none}@media (prefers-color-scheme: dark){.theme-light{display:none}.theme-dark{display:inline}}</style>
   <g class="theme-light">（pro-slate の中身）</g><g class="theme-dark">（pro-dark の中身）</g>
   ```

7. T3 Code で ACP エージェントに、その SVG の絶対パスを `![説明](絶対パス)` の1行で返させる。
8. T3 Code をライトとダークに切り替えて、表示を見る。

### 結果

| 確かめたこと | 結果 |
| --- | --- |
| Bun で描けるか | ビルドせずに TypeScript のソースを import して描けた。棒グラフは 8〜13 ms、84 点の折れ線2本と塗りつぶしの図は 64 ms だった |
| 日本語（PNG） | タイトルと軸ラベルの日本語が「□」になった |
| 日本語（SVG） | 文字が `<text font-family="Inter">採点の平均（100点満点）</text>` の形で書かれ、T3 Code では日本語のまま表示された。最初はセリフ体で表示された |
| ACP エージェントの返信での表示 | Cantle が返した `![採点の平均](/private/tmp/.../japanese.svg)` を、T3 Code は画像として表示した |
| 2テーマの SVG | 9.6 KB になった。T3 Code で、ゴシック体の日本語で崩れずに表示された |
| git 依存での導入 | `bun add github:ivarvong/mpl#9d6e19e` は、`--ignore-scripts` の有無と `trustedDependencies` の有無にかかわらず、ソース全体を入れ、`dist/` は作らなかった。`node_modules/mpl/src/index.ts` をパスで指定する import は動き、Cantle の `tsc --noEmit`（strict）も通った |

## 見直す条件

- mpljs が npm に公開されるか、`dist/` を含めて配るようになったとき。パッケージ名で import できるようになる。
- mpljs が日本語のフォントを同梱するようになったとき。PNG も選べるようになる。
- T3 Code が、Markdown の画像に使えるパスを制限するようになったとき。
- ACP の `image` の content block を T3 Code が表示できると分かったとき。ファイルを置かずに画像を渡せる。
