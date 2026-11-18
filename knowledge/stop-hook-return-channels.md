# Stop hook で agent を続けさせる返し方のうち、Claude Code と Codex の両方で例外なく通ったのは exit 0 と JSON の `decision: block` だった

## TL;DR

- Stop hook の返し方は、続けさせる、止まるのを許す、強制的に止める、の3つの役割に分かれる。
- Claude Code と Codex の両方に登録するスクリプトは、exit 0 で `{"decision":"block","reason":"..."}` を返す。
- exit 2 は、stderr に `No such file or directory` があると、Claude Code が差し止めずに agent を止めた。公式ドキュメントは、この扱いを例外として書いている。
- 各ケースの実行は1回で、Codex は一部のケースしか試していない（2026-11-14）。

## 結論

Stop hook の返し方は、役割で3つに分かれる。続けさせる、止まるのを許す、強制的に止める、である。exit 2 と stderr、`decision: "block"`、`additionalContext` の3つは、どれも「続けさせる」役割を担う。両方の CLI に登録するスクリプトは、exit 0 で `{"decision":"block","reason":"..."}` を返す。

この知見は検証だけで、`~/.claude` と `~/.codex` の設定には入れていない。

| 役割 | 返し方 | 使う場面 |
| --- | --- | --- |
| 続けさせる | exit 0 と `{"decision":"block","reason":"..."}` | 両方の CLI で同じスクリプトを使うとき。テストの出力を理由として渡すとき |
| 続けさせる | exit 2 と stderr | シェル1行で済ませるとき。渡す出力にファイルの欠落を示す行が入らないとき |
| 続けさせる | exit 0 と `hookSpecificOutput.additionalContext` | Claude Code だけで、失敗ではなく案内として渡すとき |
| 止まるのを許す | exit 0 で何も出さない | 完了判定が通ったとき。対象外の repo のとき |
| 強制的に止める | exit 0 と `{"continue": false}` | 差し止めより優先して、その場で終わらせるとき |

exit 0 と 2 以外の終了コードは hook の失敗として扱われ、どちらの CLI でも agent は止まった。hook が失敗したときに続行へ倒す設定は、Stop hook には無い。Claude Code の `onFailure: "block"` は、Stop hook では効かないと [hooks reference](https://code.claude.com/docs/en/hooks) に書かれている。

## 採用した理由

`decision: "block"` を選ぶと、exit 2 と `additionalContext` で起きた3つの問題を避けられる。

| 防げたこと | 観察した違い |
| --- | --- |
| テストの出力で続行が外れる | Claude Code は、exit 2 の stderr に `No such file or directory` があると、差し止めずに agent を止めた。同じ文字列を `reason` に入れた JSON は、続行した |
| Codex で hook が失敗する | Codex は `additionalContext` を返した hook を Failed とし、agent を止めた。`decision: "block"` は両方で続行した |
| 渡す文面に hook のコマンド行が混ざる | Claude Code は、exit 2 の stderr の前に `[<hook のコマンド行>]: ` を付けて agent に渡した。`reason` は、そのまま渡した |

[stop-hook-done-command.md](stop-hook-done-command.md) のスクリプトは、常に exit 0 で JSON を返す。そのスクリプトの返し方は、この結果と合っている。

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| exit 2 と stderr を既定にする | Claude Code で、stderr に `cat: <path>: No such file or directory` が入ると agent が止まった。同じ stderr に指示文を足しても、指示は届かなかった。失敗したテストは、この文字列を出力しうる |
| `additionalContext` を既定にする | Codex で hook が Failed になり、agent が止まった |
| 用途で返し方を分ける。続行は exit 2、停止は JSON | exit 2 と `decision: "block"` は、どちらも続けさせる返し方だった。止めるのは、exit 0 で何も出さないか、`continue: false` である |
| exit 1 で未完了を知らせる | どちらの CLI でも agent は止まり、stderr は agent に届かなかった |

## 検証

きっかけは、[Lydia Hallie の投稿](https://x.com/lydiahallie/status/2121313048820617645)である。投稿は、Stop hook でやり直させるには exit 2 で終わること、exit 1 は non-blocking であることを述べる。

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-14 |
| モデル | Claude Code は `claude-haiku-5-5`（`--model haiku`）。Codex は `gpt-6.1-sol`（reasoning effort high） |
| ツール | Claude Code 2.1.325、codex-cli 0.169.0、bash、jq、git |

### 再現手順

hook は、ケースごとに1本の bash スクリプトにする。ケース K 以外は、次の共通部分で始める。

```bash
#!/bin/bash
REPO="$1"; LOG="$2"
INPUT=$(cat)
active=$(printf "%s" "$INPUT" | jq -r ".stop_hook_active")
note() { echo "$(date +%T) stop_hook_active=$active -> $1" >> "$LOG"; }
if [ -f "$REPO/done.txt" ]; then note "allow (done.txt exists)"; exit 0; fi
```

共通部分の後に、ケースごとの本体を続ける。`MSG(X)` は、`Not done. Create a file named done.txt in the repo root containing the single word X.` という1行を指す。各本体の先頭では、`note` でケースの説明を `$LOG` に書く。

| ケース | 本体 |
| --- | --- |
| A | `echo 'MSG(A-STDERR)' >&2; exit 2` |
| B | `echo 'MSG(B-STDERR)' >&2; exit 1` |
| C | `jq -n --arg r 'MSG(C-REASON)' '{decision:"block",reason:$r}'; exit 0` |
| D | `echo 'MSG(D-STDOUT)'; exit 0` |
| E | `cat "$REPO/done.txt" >/dev/null; exit 2` |
| E2 | `cat "$REPO/done.txt" >/dev/null; echo 'MSG(E2-STDERR)' >&2; exit 2` |
| F | ケース C の `jq` の行、`echo 'MSG(F-STDERR)' >&2`、`exit 2` の順に書く |
| G | `python3 -c 'raise RuntimeError("MSG(G-CRASH)")'` |
| H | `jq -n --arg r 'MSG(H-REASON)' '{continue:false,stopReason:"H: stopped by hook",decision:"block",reason:$r}'; exit 0` |
| I | `jq -n --arg r 'MSG(I-CONTEXT)' '{hookSpecificOutput:{hookEventName:"Stop",additionalContext:$r}}'; exit 0` |
| J | `sh -c 'echo "MSG(J-STDERR)"; exit 1' >&2 \|\| { echo failed; exit 2; }` |
| K | 共通部分の最後の行（`done.txt` の判定）を外す。`echo 'Not done. Reply with the single word hello again. Do not call any tools.' >&2; exit 2` |
| L | `err=$(cat "$REPO/done.txt" 2>&1 >/dev/null)` の後、`$err` と改行と `MSG(L-REASON)` をつないだ文字列を `reason` に入れ、ケース C と同じ形で出力する |

1. ケースごとに空の git repo を作り、空の commit を1つ入れる。
2. Claude Code 用に、ケースごとの settings ファイルを書く。内容は `{"hooks":{"Stop":[{"hooks":[{"type":"command","command":"bash <hook> <repo> <log>","timeout":30}]}]}}` とする。
3. 各 repo で `claude -p --model haiku --settings <settings> --permission-mode acceptEdits --output-format stream-json --verbose "Reply with the single word hello. Do not create files unless asked." < /dev/null > <out>.jsonl` を実行する。
4. ケース A、B、C、E2、I について、Codex 用の repo を手順1と同じに作る。
5. 各 repo で `codex exec -s workspace-write --dangerously-bypass-hook-trust -c 'hooks.Stop=[{hooks=[{type="command",command="bash <hook> <repo> <log>",timeout=30}]}]' "Reply with the single word hello. Do not create files unless asked." < /dev/null` を実行する。
6. ケースごとに、`<log>` の行数、repo の `done.txt` の中身を読む。
7. Claude Code の通知を `jq -r 'select(.type=="system" and .subtype=="notification") | .key' <out>.jsonl` で読む。
8. Claude Code が agent に渡した文面を `jq -r 'select(.type=="user") | .message.content[]? | select(.type=="text") | .text' <out>.jsonl` で読む。

Claude Code は、別の Claude Code のセッションの Bash から、sandbox の中で起動した。Codex は、sandbox の中では `failed to initialize in-process app-server client: Operation not permitted` で起動できず、sandbox の外で起動した。

### 結果

「続行」は、hook が2回走り、`done.txt` にその経路の単語が書かれたことを指す。「止まる」は、hook が1回だけ走り、`done.txt` が作られなかったことを指す。

| ケース | 返し方 | Claude Code | Codex |
| --- | --- | --- | --- |
| A | exit 2 と stderr | 続行。`A-STDERR` | 続行。`A-STDERR` |
| B | exit 1 と stderr | 止まる | 止まる。`hook: Stop Failed` |
| C | exit 0 と `decision: "block"` | 続行。`C-REASON` | 続行。`C-REASON` |
| D | exit 0 と平文の stdout | 止まる。通知は無い | 実行していない |
| E | exit 2 で、stderr がファイルの欠落だけ | 止まる | 実行していない |
| E2 | exit 2 で、stderr にファイルの欠落と指示文 | 止まる | 続行。`E2-STDERR` |
| F | exit 2 と stderr と、stdout の `decision: "block"` | 続行。`F-REASON` | 実行していない |
| G | hook が例外で落ちる（exit 1） | 止まる | 実行していない |
| H | `continue: false` と `decision: "block"` | 止まる | 実行していない |
| I | exit 0 と `additionalContext` | 続行。`I-CONTEXT` | 止まる。`hook: Stop Failed` |
| J | 投稿の例の形 | 続行。`J-STDERR` | 実行していない |
| K | 常に exit 2 | hook が9回走り、9回目の後に終了 | 実行していない |
| L | exit 0 と `decision: "block"` で、`reason` にファイルの欠落と指示文 | 続行。`L-REASON` | 実行していない |

- Claude Code が agent に渡した文面は、経路で形が違った。
  - exit 2 は `Stop hook feedback:` の次の行に、`[<hook のコマンド行>]: <stderr>` を渡した。
  - `decision: "block"` は `Stop hook feedback:` の次の行に、`reason` をそのまま渡した。
  - ケース J の stdout の `failed` は、渡されなかった。
  - ケース I の文面は、stream-json の出力に user のテキストとして現れなかった。agent は `I-CONTEXT` を書いた。
- Claude Code の通知 `stop-hook-error` は、ケース A、B、C、E、E2、F、G、H、J、K、L で出た。ケース D と I では出なかった。
- ケース H では、`reason` が `Stop hook feedback:` として出力に現れたが、agent はツールを呼ばずに終わった。`stopReason` の文字列は、stream-json の出力に現れなかった。
- ケース K では、`stop-hook-block-cap` の通知と、`A hook blocked the turn from ending 9 consecutive times — overriding and ending turn.` で始まるメッセージが出た。
- ケース E、E2、F の結果は、公式ドキュメントの記述と合う（2026-11-16 に照合）。[hooks reference](https://code.claude.com/docs/en/hooks) は、`Stop`、`SubagentStop`、`TaskCompleted` と plugin の `UserPromptSubmit` について例外を書く。hook が標準出力に何も出さずに exit 2 で終わり、stderr がファイルの欠落を示すと、Claude Code は差し止めない失敗として扱う。ケース F は標準出力に JSON があり、続行した。
- `stop_hook_active` は、どちらの CLI でも1回目が `false`、続行後が `true` だった。
- Codex では、`~/.codex/hooks.json` の既存の Stop hook が同じ回に走り、`Completed` になった。[stop-hook-done-command.md](stop-hook-done-command.md) は、この hook を herdr のものと記録している。
- 確かめていないこと：
  - 各ケースは1回だけ実行した。モデルは上の表の2つだけである。
  - Codex のケース D〜H、J〜L。
  - Claude Code がファイルの欠落と判定する文字列の範囲。試したのは、`cat` が出す `No such file or directory` だけである。ドキュメントも、この文字列を例に挙げるだけで、範囲を書かない。
  - `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` で上限が変わるか。
  - 対話モードでの表示。`claude -p` と `codex exec` だけで実行した。
  - `SubagentStop` で同じ結果になるか。ドキュメントは同じ扱いと書くが、実行していない。

## 見直す条件

- Claude Code か Codex のバージョンが上がったとき。特に、Claude Code が exit 2 のファイル欠落の扱いを変えたとき。
- Codex が `additionalContext` を受け付けるようになったとき。
- 実際のテストスイートを完了判定にして、`decision: "block"` の `reason` が長くなった場合の振る舞いを観察したとき。
