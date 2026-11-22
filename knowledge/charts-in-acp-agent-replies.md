# T3 Code で ACP エージェントの返信にグラフを出すなら、テーマ変数だけで描いた HTML を html_render に渡す

## TL;DR

- T3 Code は、ACP エージェント（自作の Cantle）に自分の MCP サーバーを渡していた。その中に、HTML をスレッドに表示する `t3-code.html_render` があり、Cantle のモデルが呼ぶと成功した。
- 色と文字を T3 Code のテーマ変数（`var(--chart-1)`、`var(--foreground)`、`var(--font-sans)` など）だけで指定した図は、T3 Code のライトとダークの両方で T3 の配色に従った。日本語はゴシック体で出た。
- 先に試した「SVG を手元に書き、返信に `![](パス)` で載せる」形も表示はできた。ただし、PNG では日本語が「□」になり、SVG ではフォントの指定を書き換える必要があった。さらに、SVG の中の `prefers-color-scheme` は OS のテーマに従うと指摘されたので、採用しなかった。
- 見た目と正しさを安定させるため、モデルに HTML を自由に書かせず、用意した部品（棒グラフ、折れ線グラフ、数値カード、表）を選ばせる形を Cantle に実装する。この部品の形は、まだ検証していない。
- 確かめていないことは 3 つある。`/private/tmp` 以外のパスの HTML を `html_render` が表示するか、表示した後にファイルを消すとスレッドの図が消えるか、SVG の `prefers-color-scheme` が本当に OS に従うかである。検証日は 2026-11-22、モデルは `claude-opus-5-5`。

## 結論

T3 Code で ACP エージェントの返信にグラフを出すときは、色と文字を T3 Code のテーマ変数だけで指定した HTML 文書を作り、`html_render` で表示する。Cantle では、部品（`bar_chart`、`line_chart`、`metric_cards`、`table`）を HTML に描くツールを足し、モデルがそのファイルのパスを `html_render` に渡す形にする（[thkt/cantle#286](https://github.com/thkt/cantle/issues/286)）。

## 採用した理由

| できるようになったこと | 仕組み |
| --- | --- |
| ACP エージェントの側を変えずに HTML を表示できる | T3 Code は、ACP の `session/new` で自分の MCP サーバーを渡す。Cantle の履歴の `tool_scope` では、送ったツール 95 個の中に `t3-code.html_render` と `t3-code.html_preview` があった |
| T3 Code のテーマに従う | `html_render` は、T3 Code のテーマ変数を `:root` に置いた枠で HTML を表示する。色をテーマ変数で指定すれば、アプリのテーマを切り替えたときに図の色も変わる |
| 日本語を出す | 文字の形は、`--font-sans` が指すアプリのフォントで描かれる。描画する側にフォントを持たせる必要がない |
| 見た目と正しさを安定させる | Generative UI の分類でいう Static（用意した部品からモデルが選ぶ）にする。モデルが選ぶのは部品と値だけで、目盛りの刻みや配色は部品の側で決める。この判断は記事の整理（[Qiita](https://qiita.com/Takenoko4594/items/4387782ff8310427048e)、[Zenn](https://zenn.dev/peishim/articles/37ff4d894f8fbc)）に基づき、測定はしていない |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| mpljs で SVG を書き、返信に `![](パス)` で載せる | T3 Code は表示した。ただし、PNG では日本語が「□」になった（`Glyph 25505 (採) missing from font(s) Inter`）。SVG は文字を `<text font-family="Inter">` で書くので、Inter が無い環境ではセリフ体になり、フォントの指定を書き換える必要があった。ライトとダークの切替は SVG の中の `prefers-color-scheme` に頼るしかなく、これは T3 Code ではなく OS のテーマに従うと指摘された。ファイルの置き場所と消すタイミングの管理も要る |
| mpljs の色を目印の色で塗り、後でテーマ変数に置き換えて `html_render` に渡す | 動いた。目印の色（`#010101` など）は SVG の中ですべて置き換わり、固定の色は残らなかった。ただし mpljs は npm に無く、git 依存で入れても `dist/` が作られないので、`node_modules/mpl/src/index.ts` をパスで import する必要がある。依存を増やさずに描ける部品の数に絞ったので、使わなかった |
| mpljs の公開デモ（`mpl.ivar.workers.dev`）の URL を書く | URL の `/v1/` の後ろは、定義の JSON を raw deflate で圧縮して base64url にした文字列である。2,502 文字の JSON が 1,188 文字になっていた。モデルには圧縮を計算できず、データも外部の Worker へ送られる |
| モデルに HTML を自由に書かせて `html_render` に渡す（Open-Ended） | ACP エージェントを変えずに動く。子スレッドで、手で書いた棒グラフの HTML を Cantle に渡させると表示された。ただし、モデルが毎回書く HTML では、見た目と目盛りの正しさがそろう保証がない。記事は Open-Ended の短所として、デザインのそろえにくさ、出力トークンの多さ、安全性を挙げる |
| json-render の画像の部品（`@json-render/image`、Satori） | 部品は Frame、Box、Row、Column、Heading、Text、Image、Divider、Spacer だけで、グラフが無い。Satori は文字をパスに変えるので、日本語フォントのファイルを渡す必要がある |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-22 |
| モデル | `claude-opus-5-5`（HTML と SVG の作成、`html_preview`）。`html_render` を呼んだのは T3 Code に登録した Cantle で、モデルは記録していない |
| ツール | T3 Code（版は記録していない）、Cantle（commit は記録していない）、Bun 1.4.5、mpljs（`ivarvong/mpl` の commit `9d6e19e`）、Darwin 27.2.0（Apple silicon） |

### 再現手順

1. T3 Code に ACP エージェントを登録し、そのエージェントが受け取った MCP のツールの一覧に `html_render` があるかを確かめる。Cantle では、`~/.config/cantle/run-history/` の最新の履歴の `tool_scope` に一覧がある。
2. 色と文字をテーマ変数だけで指定した SVG を、`<!doctype html>` で始まる HTML 文書に入れて保存する。`html` と `body` には背景色を付けない。
3. 自分のスレッドで `html_preview` に、その文書の絶対パスと `appearance: "dark"`、`appearance: "light"` を順に渡し、2 枚の画像を比べる。
4. 同じ文書から `<!doctype html>` を外した断片を `html_preview` の `path` に渡し、拒否されることを確かめる。
5. T3 Code の子スレッドで ACP エージェントに、その HTML を `html_render` で表示させる。
6. 比べるため、mpljs で日本語のタイトルを含む図を PNG と SVG で書き出し、SVG の絶対パスを `![説明](パス)` の1行で ACP エージェントに返させる。

### 結果

| 確かめたこと | 結果 |
| --- | --- |
| ACP エージェントへの `html_render` の受け渡し | Cantle が送ったツール 95 個の中に `t3-code.html_render` と `t3-code.html_preview` があった |
| ACP エージェントからの `html_render` | 呼び出しは成功し、Cantle は「表示しました」と返した。T3 Code はこの呼び出しを種類 read の操作として表示した |
| テーマ変数で描いた図 | ダークとライトのどちらでも、線、目盛り、文字の色が T3 Code の配色になった。日本語のタイトルと軸ラベルはゴシック体で出た |
| `path` で渡す HTML の形 | 断片は「`<!doctype html>` か `<html>` で始まる文書を渡すこと」として拒否された |
| Markdown の画像（比較） | `![採点の平均](/private/tmp/.../japanese.svg)` は画像として表示された。PNG の日本語は「□」、SVG の日本語は最初セリフ体だった |

## 見直す条件

- T3 Code が、ACP エージェントに MCP サーバーを渡さなくなったとき、または `html_render` の名前や引数が変わったとき。
- 部品の形を Cantle に実装し、T3 Code で表示を確かめたとき。この知見に結果を足す。
- ACP の仕様に、エージェントがクライアントに HTML や部品を渡す方法が入ったとき。
