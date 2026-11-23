# 判定の入力がそろっているなら、Lean で証明した CLI は 1 回の起動にまとめて呼ぶ

## TL;DR

- Lean のビルド時に判定関数の性質を証明し、その関数を組み込んだ CLI を TypeScript から呼べる。
- 単純な判定 100 件を、1 件ずつ起動すると合計の中央値は 578.09 ms（初回）と 334.38 ms（追試）だった。100 件を 1 回の起動にまとめると 4.35 ms（初回）と 3.09 ms（追試）だった。どちらの回も、まとめると 100 倍以上短い。
- 測ったのは 100 件の合計である。1 件ずつの遅延の分布、起動だけの時間、常駐プロセスとの往復の時間は測っていない。
- 証明が保証するのは、渡した状態に対する判定までである。呼び出す側が誤った状態を渡すことは防げない。
- 検証日は 2026-11-23、モデルは `gpt-6.2-astra`（初回）と `claude-fable-5-2`（追試）。

## 結論

試作には CLI が使いやすい。入力がすでにそろっているなら、1 回の起動にまとめて渡す。入力がいつ届くか分からない用途では、先に、1 回の呼び出しに許せる待ち時間を決める。1 件ずつの起動は、100 件の合計から換算して 1 件あたり 5.78 ms（初回）と 3.34 ms（追試）だった。同期の呼び出しでは、その間イベントループも止まる。

## 採用した理由

Lean で定義した `canStart` を実行ファイルからそのまま呼べば、TypeScript に判定の規則を写さずに済む。ビルド時に「開始を許可するのは状態が idle のときに限り、idle なら必ず許可する」を証明する。実行時に要るのは、状態を渡して真偽値を受け取る処理だけである。

代わりに増える手間がある。

| 場面 | 増える手間 |
| --- | --- |
| 利用者がソースからビルドする | 利用者の環境に Lean のツールチェーンが要る |
| ビルド済みの CLI を配る | 配る側が、対応する OS とアーキテクチャごとにビルドし、動作を確かめる |
| 実行時 | 外部の実行ファイルが無い、起動できない、という失敗の扱いを決める |

今回確かめたのは、手元の macOS（arm64）での実行までである。実行時の依存を持たないアプリ（自作のエージェントハーネスの Cantle がそうである）では、この手間が採否を左右する。

証明が扱うのは、渡された状態に対する判定までである。たとえば、呼び出す側が実行中の状態を誤って idle と伝えれば、関数は正しく true を返し、二重起動は防げない。二重起動まで扱うには、開始、中断の要求、停止の完了による状態の遷移を定義し、呼び出す側がその結果を反映する順番も確かめる必要がある。

本番で証明済みの関数を呼ぶ必要があるかも、先に決める。TypeScript の実装を本番で動かし、Lean は検証のときだけ使うなら、本番の起動の費用は生じない。その場合は、Lean のモデルで示した性質が TypeScript の実装でも成り立つ根拠が別に要る。Cantle での選び方は [notes/test-combination-for-cantle-cancel.md](../notes/test-combination-for-cantle-cancel.md) に書いた。

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| 軽い判定を繰り返す経路で、毎回 `execFileSync` で起動する | 100 回の同期の起動にかかった合計の中央値は、初回 578.09 ms、追試 334.38 ms だった。各 `execFileSync` が返るまで、呼び出し元のイベントループが止まる |
| 非同期の `execFile` に替える | 呼び出し元はほかのイベントを処理できるが、プロセスを起動する回数は同じである（[Node.js の子プロセス API](https://nodejs.org/api/child_process.html)）。測ってはいない |

入力が 1 件ずつ届き、毎回の起動が遅すぎる場合の候補は、Lean の子プロセスを 1 度だけ起動し、標準入力と標準出力で 1 件ずつやり取りする方法である。この方法は測っていないので、採用も却下もしていない。導入するときは、プロセス間の往復の時間を測り、子プロセスが終了したときに返事を待っている呼び出しをどう終えるかを決める。

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-23 |
| モデル | `gpt-6.2-astra`（初回）、`claude-fable-5-2`（追試） |
| ツール | Lean 4.36.0、Bun 1.4.5、macOS（arm64）。追試は同じ Mac の、コマンドごとの sandbox の中で実行した |

### 再現手順

1. Lean 4.36.0 と Bun を用意する（[Lake のプロジェクト作成手順](https://lean-lang.org/functional_programming_in_lean/Hello___-World___/Starting-a-Project/)）。

2. 空のディレクトリでプロジェクトを作る。

   ```sh
   lake +leanprover/lean4:v4.36.0 new gatebench exe
   cd gatebench
   ```

3. `Main.lean` を次の内容に置き換える。引数 1 個につき、判定の結果を 1 行出力する。

   ```lean
   inductive Phase where
     | idle | running | cancelling

   def canStart : Phase → Bool
     | .idle => true
     | _ => false

   theorem canStart_iff_idle (p : Phase) : canStart p = true ↔ p = .idle := by
     cases p <;> simp [canStart]

   def main (args : List String) : IO Unit := do
     for arg in args do
       let phase ← match arg with
         | "idle" => pure Phase.idle
         | "running" => pure Phase.running
         | "cancelling" => pure Phase.cancelling
         | _ => throw (IO.userError "unknown phase")
       IO.println (canStart phase)
   ```

4. 同じディレクトリに、次の内容で `bench.ts` を保存する。どちらの呼び方でも、100 件の結果とその順番を期待と照合する。

   ```ts
   import { execFileSync } from "node:child_process";

   const phases = Array.from({ length: 100 }, (_, i) => ["idle", "running", "cancelling"][i % 3]);
   const expected = phases.map(p => String(p === "idle")).join("\n");
   function call(args: string[]) {
     return execFileSync("./.lake/build/bin/gatebench", args, { encoding: "utf8" }).trim();
   }
   for (let round = 0; round < 6; round++) {
     for (const batch of [false, true]) {
       const start = performance.now();
       const result = batch ? call(phases) : phases.map(p => call([p])).join("\n");
       const ms = performance.now() - start;
       if (result !== expected) throw Error("unexpected output");
       if (round > 0) console.log(batch ? "batch" : "single", ms.toFixed(2));
     }
   }
   ```

5. `lake build` を実行する。

6. `bun bench.ts` を実行する。

### 結果

同じ 100 件を同じ順に処理した。両方の呼び方を 1 回ずつ捨ててから、5 回ずつ測った。単位は 100 件あたりのミリ秒で、ビルドの時間は含まない。

| 回 | 呼び方 | 1 回目 | 2 回目 | 3 回目 | 4 回目 | 5 回目 | 中央値 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 初回 | 1 件ずつ 100 回起動 | 578.09 | 599.55 | 647.10 | 511.37 | 502.15 | 578.09 |
| 初回 | 100 件まとめて 1 回起動 | 7.72 | 3.11 | 4.35 | 3.58 | 5.13 | 4.35 |
| 追試 | 1 件ずつ 100 回起動 | 393.18 | 285.96 | 398.94 | 334.38 | 319.23 | 334.38 |
| 追試 | 100 件まとめて 1 回起動 | 2.51 | 6.26 | 3.42 | 2.98 | 3.09 | 3.09 |

どの回も、ビルドは通り、100 件すべてで期待した結果が返った。測った区間には、起動と終了、引数と標準出力の受け渡し、判定、Bun の側の文字列の処理が入る。1 件ずつの合計は、初回と追試で 1.7 倍違った。そのため、1 件あたりの値は目安にとどまる。まとめると 100 倍以上短い、という点は 2 回とも同じだった。

## 見直す条件

本番で呼ぶ頻度、1 回で扱える件数、許せる遅延が決まったら、本番と同じ OS とランタイムで測り直す。入力が 1 件ずつ届く用途では、まとめて渡した 100 件の時間から、常駐プロセスに 1 件送って返事を待つ時間は推定できない。
