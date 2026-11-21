# 状態機械を Lean の Veil で書くと、XState と同じ欠陥が見つかり、直したモデルの性質はクライアントの数を決めずに証明できた

## TL;DR

- [XState で確かめた通知の題材](state-machine-check-in-app-language.md)を Veil で書くと、欠陥2つの最短の手順も、クライアント1〜5つの状態数（9〜6,249）も、XState と一致した。
- 直したモデルの「各クライアントへの表示は1回まで」は、クライアントの数を決めずに証明できた。証明に足りない不変条件2行は agent が足した。Sonnet 5.5 で 0.25 USD・30秒、Haiku 5.5 で 0.017 USD・60秒だった（各1回）。
- Veil の証明は、既定では SMT ソルバーの答えを信用するだけで、定理が `sorryAx` に依存する。`set_option veil.smt.trust false` をファイルの先頭に書くと、`#gen_theorems` で足した定理が依存する公理は、Lean の標準の3つだけになった。
- 費用は、Veil のビルドに4分19秒、ディスクに 4.6GB だった。Veil の不変条件の自動合成（`#updr`）は、この性質では止まった。
- 「いつか必ず起こる」性質と、実装がモデルに合っているかの確かめは、試していない（2026-11-21）。

## 結論

状態機械の性質を、有限の場合だけでなく、主体の数を決めずに示したいときは、Veil で書く。先に `#model_check` で欠陥を探し、直したモデルを `#check_invariants` で証明する。

証明を正しさの根拠にするときは、`veil.smt.trust` を `false` にする。そのうえで `#gen_theorems` で定理にし、`sorryAx` に依存する定理が無いことを確かめる。

この知見は題材での検証だけで、`~/.claude` の設定にも、手元の repo にも入れていない。

## 採用した理由

XState の全列挙は、クライアントの数を決めた場合しか確かめられない。Veil は、同じ題材で同じ欠陥を見つけたうえで、数を決めない証明まで1つの言語で進められた。

| できるようになったこと | 観察 |
| --- | --- |
| XState と同じ欠陥を、同じ最短の手順で見つける | 「行き止まり」は `send → expire` の2手、「同じ端末に2回表示する」は `send → drop(0) → reconnect(0)` の3手で見つかった。どちらも XState と同じ手順である |
| モデルが XState と同じであることを確かめる | 直したモデルの状態数は、クライアント1〜5つで 9、49、249、1,249、6,249 だった。XState の数と一致した |
| クライアントの数を決めずに性質を示す | クライアントを `type client` として宣言し、数を決めずに `#check_invariants` を通した。XState で確かめたのは、クライアント5つ（6,249 状態）までである |
| 証明に足りない不変条件を agent に足させる | `#check_invariants` は「`idle` で `shown` が1の状態から `send` で2になる」という反例を出した。agent はこれを読み、`invariant` を2行足して通した |
| 証明をソルバーの答えに頼らない形にする | `veil.smt.trust` を `false` にすると、`#gen_theorems` が足した 260 個の定理が依存する公理は、`propext`、`Classical.choice`、`Quot.sound` だけだった。Veil は、この定理を `addDecl` で環境に足すので、Lean のカーネルが証明を検査する |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| 既定の `veil.smt.trust`（`true`）のまま、証明できたとみなす | 「Trusting SMT solver for 18 goals」という警告が出て、`#gen_theorems` が足した定理のうち 18 個が `sorryAx` に依存した。この設定では、正しさの根拠が SMT ソルバー（cvc5）の答えになる |
| `set_option veil.smt.trust false in` を `#check_invariants` の直前に付ける | 警告が消えなかった。ファイルの先頭（`#gen_spec` より前）に書くと消えた |
| `#updr` で不変条件を自動で合成する | 「IC3 stopped: ... Tactic `assumption` failed」で止まった。[Synthesizer-Guide](https://github.com/verse-lab/veil/blob/v4.35.0/docs/Synthesizer-Guide.md) の Limitations は、量化した添字に対する算術（今回の `shown C ≤ 1`）の不変条件は、この形では見つからないと書く |
| `lake env lean <ファイル>` で、repo の外のファイルとして走らせる | `#model_check` は動いた。`#check_invariants` は、cvc5 のネイティブ実装を読み込めずに異常終了した（`Could not find native implementation of external declaration 'cvc5.TermManager.new'`）。`Examples/` の下に置き、`lake build` でモジュールとしてビルドすると動いた |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-21 |
| モデル | 不変条件を足す agent に、`claude-sonnet-5-5` と `claude-haiku-5-5` を使った（Claude Code 2.1.329 の `claude -p`）。題材の Veil のコードは、Claude Code（`claude-opus-5-5`）のセッションで書いた |
| ツール | Veil v4.35.0（commit `00a1f980`）、Lean 4.35.0、macOS 27.2.1（Apple silicon） |

### 再現手順

題材は、XState の題材と同じ、複数のクライアントに届く1件の通知である。Veil は1回の検査で違反を1つしか報告しない。そこで、XState の `fixed` の1つの旗を、`fixRedeliver`（再配信の欠陥）と `fixExpire`（行き止まりの欠陥）の2つに分けた。

```lean
-- Trial/Notifications.lean
import Veil

veil module Notifications

type client
param fixRedeliver : Bool
param fixExpire : Bool

enum phase_t = { idle, delivering, expired, completed }

individual phase : phase_t
individual sent : Bool
relation online : client → Bool
relation acked : client → Bool
function shown : client → Nat

#gen_state

after_init {
  phase := idle
  sent := false
  online C := true
  acked C := false
  shown C := 0
}

action send {
  require phase = idle
  sent := true
  shown C := if online C then shown C + 1 else shown C
  phase := delivering
}

action drop (c : client) {
  require phase = idle ∨ phase = delivering
  require online c
  online c := false
}

action reconnect (c : client) {
  require phase = idle ∨ phase = delivering
  require ¬ online c
  online c := true
  if sent ∧ ¬ acked c ∧ (¬ fixRedeliver ∨ shown c = 0) then
    shown c := shown c + 1
}

action ack (c : client) {
  require phase = delivering
  require online c ∧ shown c > 0 ∧ ¬ acked c
  acked c := true
  if ∀ d, acked d then
    phase := completed
}

action expire {
  require phase = delivering
  phase := expired
}

safety [shown_at_most_once] shown C ≤ 1

termination phase = completed ∨ (fixExpire ∧ phase = expired)

state_constraint [bounded] shown C ≤ 2

#gen_spec

#model_check interpreted { client := Fin 2, fixRedeliver := false, fixExpire := false } {} (sequential := true)
#model_check interpreted { client := Fin 2, fixRedeliver := false, fixExpire := true } {} (sequential := true)
#model_check interpreted { client := Fin 2, fixRedeliver := true, fixExpire := true } {} (sequential := true)

end Notifications
```

XState の `always`（全員が確認したら `done` へ）は、`ack` の中で最後の確認のときに `completed` へ移す形で書いた。`termination` は XState の終了状態に当たり、それ以外で出る遷移の無い状態を、Veil は行き止まりとして報告する。`state_constraint` は、XState の `stopWhen` と同じく、無限に続く再配信を打ち切るために置いた。ただし `stopWhen` は条件に当たった状態を検査してから打ち切り、`state_constraint` は条件を破る状態を検査せずに捨てる。今回は `shown` が2の状態を検査できるよう、上限を2にした。

証明用の `Examples/Trial/NotificationsProof.lean` は、上のファイルを次のように変えたものである。

- モジュール名を `NotificationsProof` にし、`param` の2行と `state_constraint` を消す。
- `reconnect` の条件を `sent ∧ ¬ acked c ∧ shown c = 0` にする。
- `termination` を `phase = completed ∨ phase = expired` にする。
- 最後の3行を、`#model_check interpreted { client := Fin 2 } {} (sequential := true)` と `#check_invariants` にする。

agent への依頼文は次のとおり。

```text
The file Examples/Trial/NotificationsProof.lean in this repository is a Veil specification of a transition system. Running `lake build Examples.Trial.NotificationsProof` fails: `#check_invariants` reports that the safety clause `shown_at_most_once` is not preserved by every action, and prints a counterexample to induction.

Make `#check_invariants` pass by adding `invariant` clauses to the file. Do not change `after_init`, the actions, the `safety` clause, the `termination` clause, or the `#model_check` and `#check_invariants` lines. Do not add `sorry`, `axiom`, `admit`, or `set_option`.

You are done when `lake build Examples.Trial.NotificationsProof` exits 0.
```

1. Veil を取得する：`git clone https://github.com/verse-lab/veil.git` を実行し、`git checkout v4.35.0` で版を固定する。
2. Veil をビルドする：`lake build`。elan が Lean 4.35.0 を `~/.elan` に入れる。
3. 上の `Trial/Notifications.lean` を保存し、`lake env lean Trial/Notifications.lean` で3つのモデル検査を実行する。
4. 3つ目の `#model_check` だけを残し、`Fin 2` を `Fin 1` から `Fin 5` に替えて、状態数と `lake env lean` の実行時間を記録する。比べるために、`#model_check` を消したファイルの実行時間も記録する。
5. `Examples/Trial/NotificationsProof.lean` を保存し、`lake build Examples.Trial.NotificationsProof` で反例が出ることを確かめる。
6. 最後の行を `#updr` に替えたファイルをビルドし、不変条件が合成されるかを見る。
7. Veil の repo で、Claude Code 側の環境変数を引き継がないように `env -i HOME USER LOGNAME SHELL PATH TMPDIR LANG` で起動する。`claude -p "<依頼文>" --model <モデル> --output-format json --allowedTools Read Edit "Bash(lake build Examples.Trial.NotificationsProof)"` を実行する。
8. agent が変えた行を、変える前のファイルとの `diff` で確かめる。`sorry`、`axiom`、`admit`、`set_option` が無いことを `ugrep` で確かめる。
9. 変えたファイルの先頭（`import Veil` の次）に `set_option veil.smt.trust false` を書き、もう一度ビルドする。
10. 手順9のファイルと、`set_option` を書かないファイルのそれぞれで、`#check_invariants` の後に、下の `#gen_theorems` と `#eval` を足してビルドする。`MODNAME` はモジュール名に替える。
11. 変える前のファイルに戻し、モデルを替えて手順7から10をやり直す。

```lean
#gen_theorems

open Lean Elab Command in
#eval show CommandElabM Unit from do
  let env ← getEnv
  let mut names := #[]
  for (n, ci) in env.constants.map₂.toList do
    if ci matches .thmInfo _ then
      if (`MODNAME).isPrefixOf n then names := names.push n
  let mut axs : Std.HashMap Name Nat := {}
  for n in names do
    for a in (← liftCoreM (Lean.collectAxioms n)) do
      axs := axs.insert a (axs.getD a 0 + 1)
  logInfo m!"theorems: {names.size}; axioms: {axs.toList}"
```

### 結果

手順3のモデル検査は、次のとおりだった。

| 旗 | 結果 | 手順 |
| --- | --- | --- |
| 両方とも欠陥あり | `deadlock` | `send → expire`（2手） |
| 再配信の欠陥だけ | `safety_failure (violates: shown_at_most_once)` | `send → drop(c=0) → reconnect(c=0)`（3手）。`shown` が `{0 ↦ 2, 1 ↦ 1}` になった |
| 両方とも直した | `No violation (explored 49 states)` | ― |

手順4の状態数と時間は、次のとおりだった。時間は `lake env lean` 全体の実行時間で、`#model_check` の無いファイルは 4.02 秒だった。

| クライアントの数 | 状態数 | 時間 |
| --- | --- | --- |
| 1 | 9 | 4.68 秒 |
| 2 | 49 | 4.56 秒 |
| 3 | 249 | 4.78 秒 |
| 4 | 1,249 | 4.60 秒 |
| 5 | 6,249 | 4.85 秒 |

手順5の `#check_invariants` は、`send` が `shown_at_most_once` を保たないと報告した。反例の前の状態は、`phase = idle`、`sent = false`、`shown = {0 ↦ 1}` だった。ほかの4つの action と初期状態は通った。

手順7から10の agent の結果は、次のとおりだった。どちらも1回ずつの実行である。

| モデル | ターン | 時間 | 費用 | 足した不変条件 | `veil.smt.trust false` |
| --- | --- | --- | --- | --- | --- |
| `claude-sonnet-5-5` | 5 | 29.6 秒 | 0.249 USD | `phase = idle → ¬ sent`、`phase = idle → shown C = 0` | 警告なしで通った |
| `claude-haiku-5-5` | 10 | 60.5 秒 | 0.017 USD | `phase = idle ↔ ¬ sent`、`¬ sent → ∀ c, shown c = 0` | 警告なしで通った |

- 時間と費用は、`claude -p` の JSON の `duration_ms` と `total_cost_usd` である。
- どちらの `diff` も、`safety` の後に `invariant` の行を足しただけだった。`sorry`、`axiom`、`admit`、`set_option` は無かった。
- 既定の設定では、Sonnet の版で「Trusting SMT solver for 17 goals」、Haiku の版で「18 goals」という警告が出た。`veil.smt.trust false` では、どちらの版も警告が出なかった。そのファイルのビルドは、Sonnet の版が4.7秒、Haiku の版が5.1秒だった。
- 手順10で、モジュールの名前空間にある定理は 260 個だった。`veil.smt.trust false` では、Sonnet の版も Haiku の版も、依存する公理は `propext`、`Classical.choice`、`Quot.sound` だけだった。既定の設定の Haiku の版では、18 個の定理が `sorryAx` に依存した。既定の設定の Sonnet の版は、公理を集めていない。
- Veil は、`#gen_theorems` で証明済みの検証条件を `addDecl` で定理として環境に足す（`Veil/Core/Tools/Verifier/Server.lean` の `addProvenVCTheorem`）。`#check_invariants` だけでは定理は足されず、`#print axioms` で名前を引けなかった。
- agent には、Veil の repo の `.claude/CLAUDE.md` と `tla-to-veil` skill、利用者の `~/.claude` の設定が読み込まれていた。
- 手順6の `#updr` は、ソルバーを6回呼んだ後、「IC3 stopped」で止まった。
- 手順2のビルドは4分19秒だった。増えたディスクは、Veil の `.lake` が 1.8GB、Lean 4.35.0 のツールチェーンが 2.8GB だった。

確かめていないこと：

- 行き止まりが無いことは、クライアント5つまでのモデル検査で確かめただけで、証明していない。
- 「いつか必ず起こる」性質。Veil には `#temporal_check` と `prove_temporal_by` があるが、試していない。
- 実装がモデルに合っているか。XState の題材で行った、モデルから作った経路で実装を動かす確かめを、Veil では行っていない。
- 題材より大きいモデルと、実際のコードベースの状態機械。
- agent の試行は、モデルごとに1回である。

## 見直す条件

- Veil を v4.35.0 から上げたとき。特に、`veil.smt.trust` の既定値と、`#updr` が扱える範囲が変わったとき。
- 手元の repo の状態機械で試したとき。特に、主体の数を決めない証明に、agent が足す不変条件が2行で収まらなかったとき。
- 「いつか必ず起こる」性質が要るとき。`#temporal_check` を試し、結果をこのページに足す。
- 実装をモデルと突き合わせる方法を試したとき。
