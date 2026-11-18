# 約30行の題材では、切り離しと不変条件の一覧を先に入れても、agent が書くテストの検出数は上がらなかった

## TL;DR

- 約30行の関数では、判定の切り離しと不変条件の一覧を先に入れても、agent が書くテストが壊した実装を拾う数は上がらなかった。
- 費用は、いきなり書かせた場合の約4倍だった。
- 拾う数を上げたのは、完了判定が走ると伝えて agent に見直させた段だけだった。
- 各進め方は3回で、差は偶然の範囲に収まる（2026-11-14、`claude-haiku-5-5`）。

## 結論

判定と I/O が混ざった約30行の関数に、agent にテストを書かせた。先に判定を切り離し、不変条件の一覧を書かせてからテストを書かせても、壊した実装を拾う数は、いきなり書かせた場合を上回らなかった。費用は約4倍だった。拾う数を上げたのは、「完了判定が走る」と伝えたうえで agent に見直させた段だけだった。

この結果は、[measure-tests-before-adding-practices.md](measure-tests-before-adding-practices.md) の順番の根拠の1つである。設定には入れていない。

## 採用した理由

段を固定の順番で積むかどうかを、題材で先に確かめられた。

| 分かったこと | 観察 |
| --- | --- |
| 各段は実行でき、次の段の入力になる | 切り出した `decide.js` は、実行できた8回すべてで、参照実装と3,000件の入力で一致した。不変条件の一覧は、どの回も10項目だった |
| 切り離しと一覧は、拾う数を上げなかった | いきなり書かせた6回は、10種中 10、10、10、10、10、9 を拾った。切り離しと一覧の後に書かせた6回は、8、9、10、9、10、10 を拾った |
| 完了判定を伝えた段は、拾う数を上げた | 6回とも、その段の後に10種すべてを拾った。Stop hook は6回とも1回目で通し、差し止めなかった。agent は止まる前に自分でテストを足した。6回中2回は、実装を壊して確かめるスクリプトを自分で書いた |
| 段が実行されなくても、次の段は気づかない | 1回、切り離しの段が権限の拒否で何もせずに exit 0 で終わった。次の段は、切り離す前の `checkout.js` のテストを書き、スイートは通った |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| 切り離し、不変条件の一覧、テストの順で書かせる | 拾う数は上がらず、費用は1回あたり約0.08ドルで、いきなり書かせた場合の約0.02ドルの約4倍だった |
| 切り離しだけを先に入れる | 有効な2回は 10 と 8 で、いきなり書かせた場合を上回らなかった |
| 仕様書を外して差を出す | 仕様書が無い条件でも、いきなり書かせた3回は 10、10、9 を拾った。差は出なかった |
| この題材の結果だけで順番を決める | 題材は約30行で、規則が8つしか無い。どの進め方も10種近くを拾い、差が出る余地が小さかった。実プロジェクトで測り直した |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-14 |
| モデル | `claude-haiku-5-5`（`--model haiku`）。1回だけ、利用記録に `claude-opus-5-5` も現れた |
| ツール | Claude Code 2.1.325、Node.js v26.13.1（`node --test`）、git、jq、rsync |

### 再現手順

題材は `checkout.js`、`SPEC.md`、`package.json` の3ファイルからなる。`package.json` は `{ "type": "module" }` である。

```js
// checkout.js
import fs from "node:fs";

export function checkout(cartPath, couponsPath, outPath) {
  const cart = JSON.parse(fs.readFileSync(cartPath, "utf8"));
  if (!cart.items || cart.items.length === 0) {
    console.error("checkout failed: empty cart");
    throw new Error("empty cart");
  }
  let subtotal = 0;
  for (const item of cart.items) subtotal += item.price * item.qty;
  console.log(`subtotal: ${subtotal}`);
  const coupons = JSON.parse(fs.readFileSync(couponsPath, "utf8"));
  const now = Date.now();
  let best = null;
  for (const c of coupons) {
    if (!(cart.couponCodes || []).includes(c.code)) continue;
    if (!(now < Date.parse(c.expiresAt))) continue;
    if (!(subtotal >= c.minSubtotal)) continue;
    let d = c.type === "percent" ? Math.floor((subtotal * c.value) / 100) : c.value;
    d = Math.min(d, subtotal);
    if (best === null || d > best.discount || (d === best.discount && c.code < best.code)) {
      best = { code: c.code, discount: d };
    }
  }
  const discount = best ? best.discount : 0;
  if (best) console.log(`coupon ${best.code}: -${discount}`);
  const shipping = subtotal - discount >= 5000 ? 0 : 500;
  const order = {
    subtotal,
    discount,
    couponCode: best ? best.code : null,
    shipping,
    total: subtotal - discount + shipping,
  };
  fs.writeFileSync(outPath, JSON.stringify(order, null, 2));
  console.log(`total: ${order.total}`);
  return order;
}
```

`SPEC.md` は、入力と出力の形に加えて、次の8つの規則を書く。

```text
1. An empty cart (no items) is an error. The function throws `Error("empty cart")` and writes no order.
2. The subtotal is the sum of `price * qty` over all items.
3. A coupon is a candidate only when the customer entered its code and it is valid.
4. A coupon is valid when the current time is before `expiresAt` and the subtotal is at least `minSubtotal`. At exactly `expiresAt` the coupon is expired. At exactly `minSubtotal` the coupon is valid.
5. A percent coupon discounts `subtotal * value / 100`, rounded down. A fixed coupon discounts `value`. A discount never exceeds the subtotal.
6. At most one coupon applies: the candidate with the largest discount. On a tie, the code that sorts first alphabetically wins.
7. Shipping is 0 when the amount after the discount is 5000 or more, and 500 otherwise.
8. The total is `subtotal - discount + shipping`.
```

参照実装 `decide.js` は、`checkout.js` の判定の行だけを `export function decide(cart, coupons, now)` に移したものである。ファイルの読み書き、`Date.now()`、`console` の行を除き、戻り値は `order` と同じ形にする。

壊した実装は、次の10種である。`from` を `to` に置き換える。同じ置き換えを、`checkout.js` と参照実装の `decide.js` の両方にかける。

```js
const EMPTY_RETURN = "return { subtotal: 0, discount: 0, couponCode: null, shipping: 500, total: 500 };";
const MUTATIONS = [
  ["M01", "minimum-subtotal boundary", "subtotal >= c.minSubtotal", "subtotal > c.minSubtotal"],
  ["M02", "expiry boundary", "now < Date.parse(c.expiresAt)", "now <= Date.parse(c.expiresAt)"],
  ["M03", "discount cap", "d = Math.min(d, subtotal);", "d = d;"],
  ["M04", "largest discount wins", "best === null || d > best.discount || (d === best.discount && c.code < best.code)", "best === null"],
  ["M05", "shipping uses the discounted amount", "subtotal - discount >= 5000 ? 0 : 500", "subtotal >= 5000 ? 0 : 500"],
  ["M06", "percent rounds down", "Math.floor((subtotal * c.value) / 100)", "Math.round((subtotal * c.value) / 100)"],
  ["M07", "empty cart throws", 'throw new Error("empty cart");', EMPTY_RETURN],
  ["M08", "tie-break by code", "c.code < best.code", "c.code > best.code"],
  ["M09", "free-shipping boundary", "subtotal - discount >= 5000", "subtotal - discount > 5000"],
  ["M10", "only entered codes apply", "if (!(cart.couponCodes || []).includes(c.code)) continue;", ""],
];
```

進め方は5つある。各段は、別々の `claude -p` のセッションで実行する。

| 進め方 | 仕様書 | 段 |
| --- | --- | --- |
| S | あり | 切り離し、不変条件の一覧、テスト、完了判定つきの見直し |
| B2 | あり | 切り離し、テスト |
| B | あり | テスト |
| NS | なし | S と同じ4段 |
| NB | なし | テスト |

各段の prompt は、次のとおり。切り離しと完了判定つきの見直しの prompt は、仕様書の有無で変えない。

```text
切り離し: checkout.js mixes the pricing decision with file I/O and the system clock. Extract the decision into a pure function in a new file decide.js: `export function decide(cart, coupons, now)`, where `cart` and `coupons` are the parsed JSON values and `now` is epoch milliseconds (a number). It must return `{ subtotal, discount, couponCode, shipping, total }` and throw `new Error("empty cart")` for an empty cart. It must not read files, the clock, or the console. Make checkout.js call decide() and keep only I/O there. Do not change behavior. Do not write tests in this step.

不変条件の一覧: Read SPEC.md and decide.js. Write INVARIANTS.md: a numbered list of at most 10 things that must never break in the pricing decision. Each item is one sentence that states a condition and the expected result, taken from SPEC.md, not from the code. Do not write tests in this step and do not change code.

テスト（一覧あり）: Read INVARIANTS.md and SPEC.md. Write tests under test/ using node:test for decide.js so that every invariant in INVARIANTS.md has at least one test whose name starts with the invariant number, like `INV-3: ...`. Take expected values from SPEC.md, never by running the code and copying its output. Run `node --test` and make it pass. Do not change decide.js or checkout.js.

テスト（一覧なし、B2）: Read SPEC.md. Write tests under test/ using node:test for decide.js. Take expected values from SPEC.md, never by running the code and copying its output. Run `node --test` and make it pass. Do not change decide.js or checkout.js.

テスト（B）: Read SPEC.md. Write tests under test/ using node:test for checkout.js. Take expected values from SPEC.md, never by running the code and copying its output. Run `node --test` and make it pass. Do not change checkout.js.

不変条件の一覧（NS）: Read decide.js. There is no written spec, so the code is the only source of the rules. Write INVARIANTS.md: a numbered list of at most 10 things that must never break in the pricing decision. Each item is one sentence that states a condition and the expected result, including the boundary cases the code decides. Do not write tests in this step and do not change code.

テスト（NS）: Read INVARIANTS.md and decide.js. Write tests under test/ using node:test for decide.js so that every invariant in INVARIANTS.md has at least one test whose name starts with the invariant number, like `INV-3: ...`. Run `node --test` and make it pass. Do not change decide.js or checkout.js.

テスト（NB）: There is no written spec. Read checkout.js. Write tests under test/ using node:test for checkout.js that pin down its current behavior, including boundaries. Run `node --test` and make it pass. Do not change checkout.js.

完了判定つきの見直し: Run `node --test` and confirm the suite passes. A completion check runs when you finish. If it reports broken variants of decide.js that your tests do not catch, strengthen the tests under test/ until it passes. Do not change decide.js or checkout.js.
```

1. 進め方ごとに、題材を空の git repo に置いて commit する。仕様書なしの進め方では `SPEC.md` を除く。
2. 各段を `claude -p --model haiku --permission-mode acceptEdits --allowedTools "Bash(node:*),Bash(npm test:*)" --output-format stream-json --verbose "<prompt>" < /dev/null` で実行し、段ごとに commit する。
3. 切り離しの後に、agent の `decide.js` と参照実装を、生成した3,000件の入力で比べる。戻り値の JSON か、投げた例外のメッセージが違えば不一致とする。
4. テストを書く段の後に、拾う数を測る。repo を複製し、`decide.js`（B と NB では `checkout.js`）を壊した実装に差し替え、`node --test` を実行する。終了コードが 0 以外なら拾ったとする。
5. 手順4の対照として、何も差し替えない場合と、参照実装に差し替えた場合に、`node --test` が通ることを確かめる。
6. 完了判定つきの見直しでは、`--settings` で Stop hook を渡す。hook は手順4を実行し、拾われない種があれば exit 0 で `{"decision":"block","reason":"..."}` を出力する。`reason` には、拾われない種の説明と置き換えの前後を書く。
7. 段ごとの出力の `result` から、`num_turns` と `total_cost_usd` を読む。

Claude Code は、別の Claude Code のセッションの Bash から、sandbox の外で起動した。同じ日の [stop-hook-return-channels.md](stop-hook-return-channels.md) の検証では、sandbox の中から起動したセッションの Bash が `Sandbox is required but failed to initialize` で失敗した。agent が `node --test` を実行できるよう、外で起動した。

### 結果

テストを書く段の直後に、10種のうち拾った数は次のとおり。

| 進め方 | 仕様書 | 1回目 | 2回目 | 3回目 |
| --- | --- | --- | --- | --- |
| B | あり | 10 | 10 | 10 |
| B2 | あり | 10 | 8 | 無効 |
| S | あり | 8 | 9 | 10 |
| NB | なし | 10 | 10 | 9 |
| NS | なし | 9 | 10 | 10 |

- 拾われなかったのは `M05` と `M06` だけだった。`M05` は S の1回目、B2 の2回目、NS の1回目で、`M06` は S の1回目と2回目、B2 の2回目、NB の3回目で拾われなかった。
- B2 の3回目は、切り離しの段で Read と Bash が権限で拒否され、`decide.js` が作られなかった。セッションは exit 0 で終わった。次の段は `checkout.js` のテストを16件書き、`node --test` は通った。`decide.js` を差し替える測り方では、10種のどれも拾わなかった。`checkout.js` を差し替えて測り直すと、10種すべてを拾った。
- S と NS の6回は、完了判定つきの見直しの後、10種すべてを拾った。Stop hook の記録は、6回とも `call=1 allow` の1行だった。
- 完了判定つきの見直しで、テストは S で 23 から 26、25 から 27、20 から 21 に増えた。NS では 23 のまま、26 から 28、19 から 21 だった。S の1回目と NS の2回目は、`mutate.tmp.mjs` と `mutate-tmp.mjs` を repo に書いた。
- 切り離した `decide.js` は、作られた8回すべてで、参照実装との不一致が3,000件中0件だった。
- 費用は、B の2回目と3回目が 0.017 ドルと 0.023 ドルだった。S の2回目と3回目は、4段の合計で 0.074 ドルと 0.082 ドルだった。NB の3回は 0.023、0.026、0.020 ドルで、NS の3回は 0.080、0.084、0.083 ドルだった。1回目の S、B2、B は費用を記録していない。
- B2 の2回目の切り離しの段は、31ターンで 0.50 ドルかかり、利用記録に `claude-opus-5-5` が現れた。原因は調べていない。
- 段を分けた進め方では、Bash の権限の拒否が1回の進め方あたり9〜13件起きた。切り離しが実行されなかった B2 の3回目だけは3件だった。B では0件、NB では1件だった。
- 確かめていないこと：
  - モデルは `claude-haiku-5-5` だけで、各進め方は3回である。差は偶然の範囲に収まる。
  - 壊した実装は手で作った10種で、どれも1箇所の置き換えである。
  - Stop hook が差し止める経路は、一度も通っていない。
  - 切り離しと一覧が、テストの実行時間や読みやすさに与える影響。

## 見直す条件

- 規則が数十ある題材か、仕様がコードから読み取りにくい題材で、同じ5つの進め方を比べたとき。
- `claude-haiku-5-5` 以外のモデルで比べたとき。
- 完了判定を伝えずに hook だけを付け、差し止めの経路が通る条件で試したとき。
