# vercel-labs/emulate の、外部 API を状態つきで手元に立てて CI と通信を止めた sandbox で使う考え方と根拠（2026-11）

Vercel Labs の emulate は、外部サービスの API を手元で動かす偽のサーバー群である。README は「状態を持ち、本番と同じ振る舞いの API。モックではない」と書く。LLM の API を規則で答える AI サービスが、自作のエージェントハーネス（Cantle）のテストに使えるかという観点で、主張をコードで照合した。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- 中心の主張は、外部 API の代わりに、状態を持つ偽のサーバーを手元に立てれば、CI や通信を止めた sandbox でも本物と同じ手順でテストできることである。15 のサービス（GitHub、Slack、Stripe、AI など）を持ち、自前の API の偽物も書ける。
- AI サービスは OpenAI の Responses、Chat Completions と Anthropic の Messages を受け、規則で決まった答えを返す。規則は上から順に調べ、最初に合ったものが答える。使える回数、誤りの注入、途切れと停止、厳格モード、呼出しの記録は、コードで確かめた。
- 「本番と同じ」の確かめ方は、公式の SDK（`openai`、Vercel AI SDK、`@anthropic-ai/sdk`）を偽サーバーに向けて、SDK が読めるかを見るテストである。本物の API の応答を録って比べるテストは、AI サービスには見当たらなかった。
- Cantle にそのまま当てるには差がある。Responses 側は `function` 以外のツールを拒否し、Cantle が送る `namespace` のツールを受けない。Cantle は API の URL を固定で持つ。照合日は 2026-11-22 で、手元では動かしていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [vercel-labs/emulate](https://github.com/vercel-labs/emulate) の README |
| 著者・発行元 | Vercel Labs（README の表示は LABS EXPERIMENT） |
| 公開日 | リポジトリの作成は 2026-03-20 |
| 照合した版 | main `66dc87d`（2026-11-17）。Cantle は main `45a1196` |
| 照合日 | 2026-11-22 |
| 照合した範囲 | README の全文と、AI サービスのコード（`packages/@emulators/ai/src/` の `rules.ts`、`engine.ts`、`dispatch.ts`、`types.ts`、`codecs/`、`__tests__/` の契約テストの冒頭）。Cantle の `src/http.ts`、`src/claude.ts`、`src/tools.ts` |
| この資料を根拠にした Issue | なし（Cantle への導入の候補として読んだ） |

## 中心の主張

**外部 API への依存を、状態を持つ偽のサーバーに置き換え、テストを通信なしで繰り返せるようにする。** `npx emulate` で 15 のサービスが `127.0.0.1` の 4000〜4014 番に立つ。初期データは seed の設定（YAML/JSON）で与え、`reset`、`snapshot`、`restore` で戻せる。自前の API の偽物は `defineEmulator` で書き、同じ仕組みで動かせる。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| LLM の答えを、モデルではなく規則で決める | 同じ要求には毎回同じ答えを返す。規則は上から順に調べ、最初に合ったものが答える。`times` で使える回数を決める | 実装されている。規則ごとの残りの回数を `times` から作る（[rules.ts L93-L97](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/rules.ts#L93-L97)）。決め方は「回数の残る最初の規則、無ければ既定の答えか厳格モードの失敗」とコードの注釈が書く（[engine.ts L156](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/engine.ts#L156)）。規則の変種は、設定の乱数の種と規則の ID と回数から選ぶので、同じ種なら同じ答えになる（[engine.ts L119](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/engine.ts#L119)、[L137-L139](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/engine.ts#L137-L139)） |
| 規則に無い要求を見逃さない | `ai.strict: true` にすると、どの規則にも合わない要求を 400 にし、足りない規則を返す | 実装されている。厳格モードでは、合う規則が無いと失敗を返す（[engine.ts L182-L185](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/engine.ts#L182-L185)） |
| 失敗の経路もテストする | 429 などの誤りの注入、途中の切断、停止、初回の応答までの遅延を規則で書ける | 実装されている。規則の故障は `disconnect` と `stall` を持つ（[types.ts L116-L117](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/types.ts#L116-L117)）。規則の項目に `error`、`fault`、`timing` がある（[rules.ts L39](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/rules.ts#L39)） |
| 何が答えたかを後から確かめる | すべての要求を、どの規則が答えたかとともに `GET /_ai/calls` に記録する | 実装されている（[dispatch.ts L182](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/dispatch.ts#L182)） |
| 本番と同じ振る舞いにする | 「Not mocks」「production-fidelity」 | 部分的。AI サービスの契約テストは、公式の SDK を偽サーバーに向けて動かす。Chat と Responses は `openai` と Vercel AI SDK の v3・v6（[contract-chat.test.ts L1-L6](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/__tests__/contract-chat.test.ts#L1-L6)）、Messages は `@anthropic-ai/sdk`（[messages.test.ts L1](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/__tests__/messages.test.ts#L1)）である。確かめているのは SDK が読めることで、本物の API の応答を録って比べるテストは見当たらなかった |
| 前の応答を参照する会話を続けられる | 応答は既定で保存し、`previous_response_id` で続けられる。未知の ID は空の文脈として扱い、要求は成功させる | 実装されている（[openai-responses.ts L680](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/codecs/openai-responses.ts#L680)）。本物の API と違い、未知の ID でも失敗しない |

## Cantle に当てはめたときの差

| 差 | 根拠 |
| --- | --- |
| Responses 側は、`function` 以外のツールを拒否する。Cantle は OpenAI へ `type: 'namespace'` のツールを送るので、そのままでは受け付けられない | emulate は `tool.type !== "function"` を未対応として返す（[openai-responses.ts L274-L276](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/codecs/openai-responses.ts#L274-L276)）。Cantle のツールは `namespace` で包む（[cantle src/tools.ts L28-L31](https://github.com/thkt/cantle/blob/45a1196/src/tools.ts#L28-L31)） |
| 暗号化した reasoning を扱わない | AI サービスのソースに `encrypted` の語が無い。Cantle の CLI の説明は、OpenAI 側で暗号化した reasoning を使うと書く |
| Messages 側は、入力の `thinking` と `redacted_thinking` をテキストに変える | [anthropic-messages.ts L250-L252](https://github.com/vercel-labs/emulate/blob/66dc87d/packages/@emulators/ai/src/codecs/anthropic-messages.ts#L250-L252)。Cantle の Claude の経路が送る thinking の扱いと合うかは確かめていない |
| Cantle は API の URL を固定で持つ。偽サーバーへ向けるには、URL を差し替える口が要る | OpenAI は `https://api.openai.com/v1`（[cantle src/http.ts L3](https://github.com/thkt/cantle/blob/45a1196/src/http.ts#L3)）、Claude は `https://api.anthropic.com/v1/messages`（[cantle src/claude.ts L8](https://github.com/thkt/cantle/blob/45a1196/src/claude.ts#L8)）。Cantle のテストは今、`fetch` を差し替える模擬通信（`evaluation/mock.ts` の `mockFetch`）を使う |
| 偽サーバーでの成功を、実アカウントでの成功として扱えない | Cantle の OUTCOME の Constraints は「模擬通信の成功を、実アカウントでの推論成功として扱わない」と定める。emulate も模擬通信の一種である |

## 限界

- **手元で動かしていない。** 起動、SDK からの接続、規則の動作は、コードと README の照合だけである。
- **AI サービス以外は照合していない。** GitHub、Slack などの「本番と同じ」は、README の記述だけである。
- **状態は既定でメモリに置く。** 永続化は選んで有効にする（README の Custom emulators の節）。
- **実験の段階である。** README の表示は LABS EXPERIMENT で、照合日の時点で open の Issue が 110 件あった。

## 照合の結果

- **照合した主張：** 「考え方と根拠」と「Cantle に当てはめたときの差」の各行は、引いた行で確かめた。ただし「暗号化した reasoning を扱わない」は、語が無いことからの判断である。
- **読んでいないもの：** AI サービス以外のサービスのコード、`docs/` とサイト（emulate.dev）、`tests/contracts/` の契約テスト、契約テストの本体（冒頭の import だけを読んだ）。
- **範囲外：** OAuth の各サービスの流れ、portless との連携。Cantle のテストに関わらないため。

## 照合の手順

1. 照合した版を取得する：`git clone --depth 1 https://github.com/vercel-labs/emulate.git`。`git log -1` が `66dc87d` であることを確かめる。新しい版で照合し直すときは、その commit を記録する。
2. README の「AI API」の節を読む。
3. 規則の決め方を確かめる：`packages/@emulators/ai/src/rules.ts` と `engine.ts` の、回数と変種と厳格モードの行を読む。
4. 契約テストが何を確かめるかを見る：`__tests__/contract-chat.test.ts`、`contract-responses.test.ts`、`messages.test.ts` の import を読む。
5. Cantle との差を確かめる：`codecs/openai-responses.ts` のツールの種類の検査と、Cantle の `src/tools.ts`、`src/http.ts`、`src/claude.ts` を読む。
6. 暗号化した reasoning の扱いを探す：`ugrep -rn encrypted packages/@emulators/ai/src` で、何も出ないことを確かめる。

## 見直す条件

- emulate の Responses 側が `namespace` のツールか、暗号化した reasoning を受けるようになったとき。
- Cantle が API の URL を差し替える口を持ったとき。
- 本物の API の応答と比べる契約テストが入ったとき。
