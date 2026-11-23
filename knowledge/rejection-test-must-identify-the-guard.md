# 依頼の拒否を確かめるテストは、拒否した仕組みを見分けないと、片方の検査を壊しても通る

## TL;DR

- 自作のエージェントハーネス（Cantle）で、中断を受けた依頼 A の停止が終わる前に依頼 B を送るテストを書いた。元の実装は、B をセッションの検査で拒否した。
- `session/cancel` で実行中の印も外すように実装を壊しても、B はモデルへの要求まで進まなかった。A が持つ状態ディレクトリのロックが、B を拒否したためである。
- この壊し方では、エラーコード、`/実行中/` の文言、B の時点のモデルへの要求の数では差が出なかった。`/同じセッション/` の文言で差が出た。既存の ACP のテスト 41 件は、壊した実装でも全部通った。
- 確かめたのは、壊し方 1 種類、A がモデルの応答を待っている場面、同じプロセスの中だけである。検証日は 2026-11-23、モデルは `claude-fable-5-2`。

## 結論

依頼の拒否を確かめるテストには、どの仕組みが拒否したかを見分ける観測を入れる。Cantle では、応答の文言がそれにあたる。このテストは Cantle にはまだ入れていない。

## 採用した理由

Cantle には、依頼の重なりを止める仕組みが 2 つあり、守る区間が違う。

| 仕組み | 位置 | 拒否する時点 | 応答 |
| --- | --- | --- | --- |
| セッションの検査 | [src/acp.ts:159](https://github.com/thkt/cantle/blob/495ed43/src/acp.ts#L159) | 依頼の処理に入る前 | `-32603`「同じセッションの依頼が実行中です。」 |
| 状態ディレクトリのロック | [src/acp.ts:293](https://github.com/thkt/cantle/blob/495ed43/src/acp.ts#L293)、[src/store.ts:58](https://github.com/thkt/cantle/blob/495ed43/src/store.ts#L58) | 履歴の記録と推論を始める前 | `-32603`「別の処理が実行中です。…」 |

セッションの検査を壊すと、B は自分の controller をセッションに入れてから、ロックで拒否される。つまり、B はモデルへの要求までは進まないが、ACP の依頼の処理には入っている。この違いは、エラーコードと `/実行中/` の文言では見えない。どちらの応答も同じコードで、どちらの文言も「実行中」を含むためである。

ロックが B を止めたのは、A が同じロックのパスを持っていたからである。別のプロセスや、別の状態ディレクトリでの動作は測っていない。

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| エラーコードだけを確かめる | 元の実装も壊した実装も `-32603` を返した |
| 文言を `/実行中/` で確かめる | 壊した実装でも通った |
| B の時点のモデルへの要求の数だけを確かめる | 元の実装も壊した実装も 1 だった |

A の停止を待つ間、セッションが A の controller を持ち続けているかを直接観測する方法もある。この方法なら、B の拒否の理由を使わずに同じ壊し方を拾えるはずだが、試していない。

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-23 |
| モデル | `claude-fable-5-2` |
| ツール | Cantle の main `495ed43`、Bun 1.4.4（Cantle の `mise.toml` が固定する版）、macOS（arm64） |

### 再現手順

1. Cantle を取得し、依存を入れる：`git clone https://github.com/thkt/cantle.git && cd cantle && git checkout 495ed43 && bun install --frozen-lockfile`

2. `test/acp.test.ts` の末尾に、次のテストを足す。A のモデルへの要求は、`stop` を呼ぶまで終わらない。

   ```ts
   test('probe 中断を受けた依頼の停止が終わる前に届いた次の依頼は拒否する', async () => {
     let stop!: (error: Error) => void;
     let started!: () => void;
     const entered = new Promise<void>((resolve) => {
       started = resolve;
     });
     let calls = 0;
     let aborted = false;
     const w = wire(async (_body, signal) => {
       if (++calls > 1) return sse('gpt-6.1-sol', [text('次の依頼の回答')]);
       started();
       return new Promise<Response>((_resolve, reject) => {
         stop = reject;
         signal.addEventListener('abort', () => (aborted = true), { once: true });
       });
     });
     const c = await client({ loggedIn: true, ...w });
     try {
       const sessionId = await session(c);
       const pending = c.request('session/prompt', prompt(sessionId));
       await entered;
       c.send({ method: 'session/cancel', params: { sessionId } });
       const during = await c.request('session/prompt', prompt(sessionId, '停止待ち中の依頼'));
       assert.equal(aborted, true);
       assert.equal(during.error?.code, -32603);
       assert.match(during.error!.message, /同じセッション/);
       assert.equal(w.bodies.length, 1);
       stop(new Error('合成の停止完了'));
       assert.equal((await pending).result.stopReason, 'cancelled');
       assert.equal((await c.request('session/prompt', prompt(sessionId, '次の依頼'))).result.stopReason, 'end_turn');
       assert.equal(w.bodies.length, 2);
     } finally {
       stop?.(new Error('合成通信の後始末'));
       await c.close();
     }
   });
   ```

3. テストを実行する：`bun --no-env-file test test/acp.test.ts -t 'probe'`

4. `src/acp.ts` の `session/cancel` の処理を、実行中の印も外すように壊す。

   ```ts
   if (method === 'session/cancel') {
     const cancelled = sessions.get(targetId)!;
     cancelled.controller?.abort();
     cancelled.controller = undefined;
     return {};
   }
   ```

5. 手順 3 をもう一度実行する。

6. 足したテストの `/同じセッション/` を `/実行中/` に変え、`bun --no-env-file test test/acp.test.ts` でファイル全体を実行する。

### 結果

| 実装 | B への応答 | B の時点のモデルへの要求の数 | `/同じセッション/` のテスト | `/実行中/` のテスト |
| --- | --- | ---: | --- | --- |
| 元の実装 | `-32603`「同じセッションの依頼が実行中です。」 | 1 | 通る | 通る |
| 壊した実装 | `-32603`「別の処理が実行中です。…」 | 1 | 落ちる | 通る |

壊した実装で `test/acp.test.ts` の全体を実行すると、既存の 41 件と、`/実行中/` に変えたテストの合わせて 42 件が通った。どちらの実装でも、A を解放した後の依頼は完了した。

## 見直す条件

- Cantle が、応答にエラーの種類を表す項目を持つようになったとき。文言より確かな見分け方になる。
- 別の壊し方（検査の行を消す、`finally` で印を外し忘れる、など）や、ツールの実行中と許可待ちの場面で測ったとき。
- 状態ディレクトリのロックの持ち方が変わったとき。
