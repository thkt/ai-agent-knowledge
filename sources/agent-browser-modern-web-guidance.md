# agent-browser の modern-web-guidance skill が、ガイドの適用をブラウザで確かめる範囲（2026-11）

Vercel の agent-browser は、v0.43.0 で skill `modern-web-guidance` を加えた。Chrome の Modern Web Guidance でコードの書き方を決め、agent-browser で組み上がったページを確かめる skill である。skill の本文、文書のページ、紹介の投稿を照合し、ブラウザで確かめる範囲を Chrome の guide の数と突き合わせた。Chrome 側の skill は [modern-web-guidance.md](modern-web-guidance.md) で照合した。自分たちが採用した判断ではなく、手元ではこの skill を自分のページに当てていない。

## TL;DR

- skill の主張は、ガイドはコードを書く材料になるがページを動かさないので、適用した規則ごとにブラウザで確かめた証拠を取る、というものである。
- ブラウザの検査が用意されているのは4種類（レイアウトシフト、reduced motion、ダークモード、dialog の focus と inert）だけである。検査の無い規則は「未検証」と報告させる。
- Chrome の公開の guide は210ある。照合者がカテゴリで分けると、画面の見た目と操作で確かめられる見込みのものは141で、4種類の検査が直接覆うのはその一部である。
- Chrome の元の repo には、guide ごとの期待の一覧（`expectations.md`）と Playwright の採点（`grader.ts`）がある。ただし採点は評価用のページを前提にし、公開の skill には含まれない。
- 効果の測定は無い。CLI は実行のたびに最新版を取得する。

| 項目 | 内容 |
| --- | --- |
| 出典 | 次の表の5件 |
| 著者・発行元 | skill は Vercel（vercel-labs/agent-browser、Apache-2.0）。guide は Google Chrome のチーム（Apache-2.0） |
| 照合日 | 2026-11-19 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 版と公開日 | 読んだ範囲 |
| --- | --- | --- | --- |
| 投稿 | [Chris Tate の投稿](https://x.com/ctatedev/status/2122816672625275030) | 2026-11-17 22:18 UTC | 本文。画像は取得していない |
| skill | [skill-data/modern-web-guidance/SKILL.md](https://github.com/vercel-labs/agent-browser/blob/25b5d76/skill-data/modern-web-guidance/SKILL.md) | main `25b5d76`（2026-11-19）。skill を加えた commit は `bc348f0`（2026-11-12）、リリースは v0.43.0（同日） | 全文（165行）。本体の `skills/agent-browser/SKILL.md` の全文（51行） |
| 文書 | 同じ repo の `docs/content/docs/modern-web-guidance.mdx` | 同じ版 | 冒頭から117行目まで |
| Chrome の公開 skill | [GoogleChrome/modern-web-guidance](https://github.com/GoogleChrome/modern-web-guidance) | main `64a6d70`（2026-11-16） | ファイルの一覧。`guides/ui-behaviors/light-dismiss-a-dialog.md`（89行）の見出しと規則の行。同じファイルの全文を、expect、grader、playwright、test、verify の語で検索した |
| Chrome の元の repo | [GoogleChrome/modern-web-guidance-src](https://github.com/GoogleChrome/modern-web-guidance-src) | main `e8feb43`（2026-11-18） | ファイルの一覧。`docs/EVALS.md` の Overview。`guides/ui-components/dialog/expectations.md` の全文。`guides/ui-behaviors/light-dismiss-a-dialog/` の `expectations.md` の全文と `grader.ts` の冒頭80行 |

## 中心の主張

**ガイドはコードを書く材料になるが、ページを動かさないので、効いたかは分からない。** skill はそこで役割を分ける。Modern Web Guidance で何を書くかを決め、agent-browser で組み上がったページを Chrome で確かめる。終えてよいのは、適用した規則のすべてに、ブラウザの検査の証拠が付いたときだけだと書く。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| ガイドを引き、規則を ID つきで書き出す | `search` で候補を得て、`retrieve` で読む。出力が9,500バイトを超えると一時ファイルに保存してパスだけを出すので、そのファイルを全部読む。`legacy-scan` は古い書き方をソースから静的に探す | skill に記述がある。`legacy-scan` が規則を持つのは v1.0.0 で33の guide だけで、指摘が無くても全 guide に従っているとは言えない、と skill 自身が書く |
| 変える前に基準値を取る | 既存の UI を直すときは、編集の前に同じ検査を走らせ、比べる数値かスナップショットを残す | skill に記述がある |
| レイアウトシフトを `vitals` で測る | load の後さらに3秒待ってから、そのページの読込みの指標を出す。`--json` は各シフトの時刻と値を出すので、時刻から原因のコードを探す | skill に記述がある。合格は、CLS が0.1以下で、かつ基準値より低いこと |
| reduced motion とダークモードを、設定を切り替えて読む | `set media reduced-motion` か `set media dark` の下と既定の下で、同じ要素の computed style を読む。見た目は動画（`record`）かスクリーンショットに残し、人が比べる | skill に記述がある。合格は「guide が定める値と一致する」で、何を一致とするかは agent の読みに任される。両方の設定で同じ値なら、設定を無視していると判定する |
| dialog の focus と inert を、accessibility tree と操作で確かめる | 開いている間の `snapshot -i` に背後の要素が出ない。focus が dialog の中にある。背後のクリックが失敗する。`Escape` で閉じると focus が開いたボタンに戻る | skill に記述がある |
| 規則ごとに証拠を報告する | guide、規則、検査、結果、証拠の表を出す。証拠はコマンドの出力の引用で、動画とスクリーンショットはパスで付ける。検査の無い規則は理由つきで未検証とし、コードが guide に従っていることを合格の理由にしない | skill に記述がある |
| 自分のページで動かす | 自分の dev サーバーか preview サーバーを使う。`vitals` はページを読み直すので、開いた dialog などの状態は消える。動画には ffmpeg が要る | skill に記述がある |

## ブラウザで確かめる範囲

この節の分類は照合者のものである。guide の本文は1つずつ読んでおらず、カテゴリの名前と guide の ID から分けた。

| 照合者の分類 | カテゴリと guide の数 | 計 |
| --- | --- | --- |
| 画面の見た目と操作で確かめられる見込み | ui-behaviors 40、ui-components 22、visual-design 22、forms 19、css 18、ui-atoms 13、accessibility 4、user-experience 3 | 141 |
| 一部だけ画面で確かめられる見込み | performance 26（CLS などは `vitals` で測れるが、計測の送信のまとめ方などは画面に出ない）、html 8（Web Components の描画は見えるが、作り方の規則は見えない） | 34 |
| 画面では確かめにくい | security 17、js 6、built-in-ai 5、webmcp 3、pwa 2、wasm 1、privacy 1 | 35 |

skill の4種類の検査が例に挙げる guide は、performance、motion、animate-to-from-top-layer、spinner、dark-mode、color、dialog の7つである。文書のページは、visually-stable-font-fallbacks も挙げる。

Chrome の元の repo は、guide ごとに確かめる材料を持つ。

- **期待の一覧と採点が、guide のほとんどにある。** `guides/` の下に guide のディレクトリが252あり、213に `expectations.md`、209に `grader.ts` がある。公開の guide は210で、数が違う理由は確かめていない。`docs/EVALS.md` は、agent の実装を結果に基づく Playwright の検査で確かめると書く。
- **採点は、評価用のページを前提にする。** 読んだ `light-dismiss-a-dialog/grader.ts` は、環境変数 `TARGET_FILE` の HTML を1つ開き、`.test-dialog-trigger` という評価用の class で要素を探す。`targets/daily-grind/grader.ts` のように、見本のアプリごとの採点もある。利用者のアプリにはそのまま当てられない。
- **期待の一覧は、skill の検査より細かい。** dialog の `expectations.md` は10項目ある。skill の dialog の検査（4条件）に無いものとして、未保存の変更があるときの `closedby="none"`、背後のページがスクロールしないこと、`<dialog>` に `role="dialog"` を付けないこと、がある。
- **公開の skill には、期待の一覧も採点も無い。** 公開の `light-dismiss-a-dialog.md` は、MANDATORY と DO NOT の規則を持つが、期待の一覧と採点は持たない。npm のパッケージに含まれるかは確かめていない。

## 限界

- **効果の測定は無い。** 投稿、skill、文書のどれにも、使う前後を比べた数字は無い。Chrome のブログの「約50%から約90%」は guide 単体の数字で、ブラウザの検査を足した効果ではない（[modern-web-guidance.md](modern-web-guidance.md)）。
- **ブラウザの検査は4種類に限られる。** 画面で確かめられる見込みの guide の多くは、検査が無いので未検証で終わる。
- **CLI は実行のたびに最新版を取得する。** skill の `allowed-tools` は `npx modern-web-guidance@latest` を許可する。同じ repo は 2026-10-29 に、本体の skill から `npx agent-browser:*` の許可を外した（`495a987`、#2249）。その commit は、既定ではパッケージの解決を許可しないためと書く。新しい skill は、この方針と食い違う。
- **合格の条件に、agent の読みに任される部分がある。** 数値で決まるのはレイアウトシフト（CLS 0.1以下）と dialog の4条件で、reduced motion とダークモードは guide との一致を agent が判断する。
- **提供元は当事者である。** skill は、自社の agent-browser の機能（`vitals`、`set media`、`record`）を使う手順として書かれている。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせた結果である。どの設定にも入れていない。

| skill の考え方 | 手元の資料 |
| --- | --- |
| 規則ごとにブラウザの証拠を取る | 手元の `rules/core/OPERATION.md` の Visual Verification は、レイアウトと見た目の変更に前後のスクリーンショットを、操作できる要素にブラウザでの確認を求める（[OPERATION.md:31-36](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/OPERATION.md?plain=1#L31-L36)）。skill は、それを guide の規則ごとの合格の条件まで具体にした形である |
| ガイドを引いた後に、自分のページで確かめる | [modern-web-guidance.md](modern-web-guidance.md) の Chrome の skill は、使った guide の ID と規則の引用を開示するところまでで止まる。この skill は、その後ろに、利用者のページでの実測を足す |
| 検査の無い規則を、未検証として返す | 依頼者は、画面で確かめられる範囲の確認を、skill が全部持つことを求める。今の skill が持つのは4種類で、141の見込みのうちの一部である。Chrome の元の repo には guide ごとの期待の一覧と採点があるが、評価用のページを前提にし、公開の skill に届いていない。その期待の一覧を agent-browser の操作に置き換える手順があれば、この差は縮む。そうした手順は、照合した範囲に無い |

## 照合の結果

- **照合した主張：** 「考え方と根拠」と「限界」の各行は、右の列か本文に挙げた原文に記述がある。
- **動かしていないもの：** skill、`modern-web-guidance` の CLI、agent-browser の検査。手元の agent-browser が 0.43.0 で、`skills list` にこの skill が出ることだけを確かめた。
- **読んでいないもの：** 投稿の画像。文書のページの118行目以降。`light-dismiss-a-dialog/grader.ts` の81行目以降。dialog と light-dismiss-a-dialog 以外の `expectations.md` と `grader.ts`。

## 照合の手順

1. 投稿を取得する：`xr tweet https://x.com/ctatedev/status/2122816672625275030`。`--thread` を付けると画像の URL が落ちるので、付けない。
2. skill を読む：`scout repo-read vercel-labs/agent-browser skill-data/modern-web-guidance/SKILL.md --ref 25b5d76`
3. skill を加えた commit のファイルを確かめる：`gh api repos/vercel-labs/agent-browser/commits/bc348f0 --jq '.files[].filename'`
4. npx の許可を外した commit を確かめる：`gh api repos/vercel-labs/agent-browser/commits/495a987 --jq '.files[].patch' | ugrep allowed-tools`
5. 公開の guide を数える：`scout repo-tree GoogleChrome/modern-web-guidance` の出力から、`skills/modern-web-guidance/guides/` の下の `.md` をカテゴリごとに数える。
6. 元の repo の採点を数える：`scout repo-tree GoogleChrome/modern-web-guidance-src` の出力から、`guides/<カテゴリ>/<guide>` ごとに、配下に `guide.md`、`expectations.md`、`grader.ts` を持つディレクトリを数える。
7. 手元の版を確かめる：`agent-browser --version` と `agent-browser skills list`

## 見直す条件

- skill に、4種類のほかの検査が足されたとき。
- Chrome の公開 skill か npm のパッケージに、guide ごとの期待の一覧が入ったとき。
- 手元で、この skill を自分のページに当てたとき。
