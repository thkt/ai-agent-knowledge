# Stop hook に完了判定のコマンドを持たせると、agent の「完了」を Claude Code と Codex で同じ1本のスクリプトで強制できる

## 結論

Stop hook から、repo ごとに決めた完了判定のコマンドを実行する。exit 0 なら止めるのを許し、それ以外なら出力を理由に付けて停止を差し止める。同じスクリプトを Claude Code と Codex の両方に登録すると、どちらでも agent が完了判定を満たすまで作業を続けた。

この知見は検証だけで、`~/.claude` と `~/.codex` の設定には入れていない。

## 採用した理由

agent が自分で「完了」と言って止まることを、Workflow を組まずに防げる。次に何をするかは agent が決め、終わったかどうかは完了判定のコマンドが決める。

| できるようになったこと | 仕組み |
| --- | --- |
| 1本のスクリプトで両方の CLI を制御する | Claude Code と Codex の Stop hook は、どちらも `{"decision":"block","reason":"..."}` で停止を差し止める。Codex は exit 0 のときプレーンテキストを受け付けないため、スクリプトは常に JSON を出す |
| 未完了の理由を、そのまま次の指示にする | 完了判定のコマンドが出力した末尾20行を `reason` に入れる。Claude Code は続ける理由として、Codex は新しい user prompt として受け取る |
| 用途ごとに hook を書き換えない | 完了判定のコマンドを `.git/agent-loop/until` に書くだけで、テスト、lint、Hill-Climb の `status` などに切り替わる |
| climb 中でない作業を止めない | `until` が無い repo では、何もせずに止めるのを許す |

この形は、[Hill-Climb の照合](../sources/hill-climb.md)で組み立てた Hill-Climb 専用の hook を汎用化したものである。

### 完了判定のコマンドに求める性質

| 性質 | 理由 |
| --- | --- |
| 完了なら exit 0、未完了なら 0 以外で終わる | スクリプトは終了コードだけで判定する |
| 未完了のとき、残っている作業を出力する | 出力の末尾20行が、agent が次に受け取る指示になる |
| 60秒以内に終わる | スクリプトは60秒で打ち切り、警告を出して止めるのを許す。hook の timeout は90秒にした |
| ファイルを書かない | スクリプトは、判定の出力、HEAD、`git status --porcelain` が変わらないまま8回差し止めたら止めるのを許す。判定のたびにファイルが変わると、この上限に届かない |

### 用途の例

| 用途 | `until` に書くコマンド |
| --- | --- |
| テストと lint を通す | `npm test && npm run lint` |
| 監査で見つかった指摘を0件にする | 指摘ごとに書いた失敗するテストを実行するコマンド |
| Hill-Climb の climb を終わらせる | `out=$(python3 /path/to/climb.py status <climb名>); printf '%s\n' "$out"; printf '%s\n' "$out" \| grep -qE '^(BAR REACHED\|STUCK)'` |

Hill-Climb の `status` は予算切れを出力しない。予算で止めたい場合は、`until` に `log.jsonl` の step 数の判定を足す。足さない場合、予算を使い切った後は、変化のない差し止めが8回続いた時点で止まる。

### スクリプト

採用するときは、このスクリプトの正本を `~/.claude/hooks/` に移し、このページからはリンクする。

```python
#!/usr/bin/env python3
"""Stop hook for Claude Code and Codex: keep the agent working until a done-command exits 0."""
import hashlib
import json
import subprocess
import sys
from pathlib import Path

CAP = 8  # same limit Claude Code applies; Codex documents none
TIMEOUT = 60  # keep below the hook's own timeout


def reply(payload):
    print(json.dumps(payload))  # Codex rejects plain text on exit 0
    sys.exit(0)


def git(*args, cwd):
    return subprocess.run(["git", *args], cwd=cwd, capture_output=True, text=True)


event = json.load(sys.stdin)
if event.get("background_tasks"):
    reply({})
top = git("rev-parse", "--show-toplevel", cwd=event.get("cwd") or ".")
if top.returncode:
    reply({})
root = top.stdout.strip()
# Under .git so a cloned repo cannot plant a command, and per worktree.
loop_dir = Path(root, git("rev-parse", "--git-path", "agent-loop", cwd=root).stdout.strip())
until = loop_dir / "until"
if not until.is_file():
    reply({})
command = until.read_text().strip()
try:
    done = subprocess.run(command, shell=True, cwd=root, capture_output=True, text=True, timeout=TIMEOUT)
except subprocess.TimeoutExpired:
    reply({"systemMessage": f"agent-loop: done-command timed out after {TIMEOUT}s; letting the agent stop"})
if done.returncode == 0:
    reply({})

output = (done.stdout + done.stderr).strip()
progress = hashlib.sha256("\n".join([
    output,
    git("rev-parse", "HEAD", cwd=root).stdout,
    git("status", "--porcelain", cwd=root).stdout,
]).encode()).hexdigest()
state_file = loop_dir / "state.json"
state = json.loads(state_file.read_text()) if state_file.exists() else {}
blocks = state.get("blocks", 0) + 1 if state.get("progress") == progress else 1
if blocks > CAP:
    state_file.unlink(missing_ok=True)
    reply({"systemMessage": f"agent-loop: {CAP} blocks in a row with no change; letting the agent stop"})
state_file.write_text(json.dumps({"progress": progress, "blocks": blocks}))
tail = "\n".join(output.splitlines()[-20:])
reply({"decision": "block", "reason": f"Not done: `{command}` exited {done.returncode}.\n{tail}"})
```

### 使い方

1. 強制を始めるときは、repo のルートで `mkdir -p "$(git rev-parse --git-path agent-loop)"` を実行する。
2. 完了判定のコマンドを `"$(git rev-parse --git-path agent-loop)/until"` に書く。
3. 強制をやめるときは、`until` を消す。

### 両 CLI の差

| 項目 | Claude Code | Codex |
| --- | --- | --- |
| 登録先 | settings の `hooks.Stop`。検証では `--settings` でファイルを渡した | `hooks.json` の `hooks.Stop` |
| 実行前の承認 | settings ファイルの hook と同じ workspace trust | `/hooks` で hook ごとに trust する。`<repo>/.codex/` に置くなら project の trust も要る |
| 連続続行の上限 | 組み込みで8回。Claude がツールを呼ぶたびにリセットされる | ドキュメントに記述が無い。スクリプトが、変化のない差し止めを8回まで数える |
| 既存の Stop hook | 2026-11-13 時点で無い | `herdr-agent-state.sh idle` がある。検証では同時に起動し、herdr は Completed、このスクリプトは Blocked になり、続行が通った |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| Workflow で、agent を呼ぶループを組む | 停止の判定は決定論的にできる。ただし step ごとに agent を起動し直すため、コードの読み直しの分だけトークンが増える。並列に書き込めないため、Workflow を使っても速くならない |
| Claude Code の `/goal` | 評価するのは軽量なモデルで、会話の記録だけを読み、コマンドを実行しない（[/goal](https://code.claude.com/docs/en/goal)）。agent の出力が評価モデルを納得させれば止まれる。Codex では使えない |
| CLI ごとの外側ループ（`codex exec resume --last`、`claude -p --continue`） | CLI ごとに別のループが要る。Stop hook なら同じスクリプトを両方に登録できる |
| Hill-Climb 専用の hook | 完了判定が `climb.py status` に固定される。汎用化しても、`until` に `status` の判定を書けば同じ動きになる |
| `stop_hook_active` で続行を1回に限る | 完了まで何度も差し止めるのが正常な動きなので、2回目の停止で強制が外れる |
| `until` を作業ツリーに置く | repo に commit された `until` が、clone した人のセッションで実行される。`.git/` の下なら commit できない |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-13 |
| モデル | Claude Code は `claude-haiku-5-5`（`--model haiku`）。Codex は `gpt-6.1-sol`（reasoning effort high） |
| ツール | Claude Code 2.1.323、codex-cli 0.168.0、Python 3、git |

### 再現手順

1. 上のスクリプトを `agent-loop-stop.py` として保存する。
2. 空の git repo を作り、空の commit を1つ入れる。
3. `.git/agent-loop/until` に `test -f done.txt || { echo "Create a file named done.txt in the repo root containing the word ok."; exit 1; }` を書く。
4. スクリプトだけを確かめる。`printf '{"cwd":"<repo>"}' | python3 agent-loop-stop.py` を、次の状態でそれぞれ実行する。
   - `until` が無い
   - `until` がある
   - 入力に `"background_tasks":[1]` を足す
   - 同じ状態で9回続けて呼ぶ
   - 9回目の後にファイルを1つ足して呼ぶ
   - `done.txt` を作ってから呼ぶ
5. Claude Code で確かめる。`{"hooks":{"Stop":[{"hooks":[{"type":"command","command":"python3 <path>/agent-loop-stop.py","timeout":90}]}]}}` を settings ファイルに書く。
6. 新しい repo で手順2〜3を行い、`claude -p --model haiku --settings <settings> --permission-mode acceptEdits "Reply with the single word hello. Do not create files unless asked."` を実行する。
7. Codex で確かめる。新しい repo で手順2〜3を行い、`.codex/hooks.json` に手順5と同じ JSON を書く。
8. `codex exec -s workspace-write --dangerously-bypass-hook-trust -c 'projects."<repo>".trust_level="trusted"' "Reply with the single word hello. Do not create files unless asked." < /dev/null` を実行する。
9. 手順6と8の後に、それぞれの repo の `done.txt` を読む。

### 結果

- 手順4のスクリプトの判定は、次のとおりだった。
  - `until` が無いと `{}` を返した。
  - `until` があると `decision: block` を返し、`reason` にコマンドの出力が入った。
  - `background_tasks` があると `{}` を返した。
  - 同じ状態での9回目は、`systemMessage` を付けて止めるのを許した。
  - ファイルを足すと、回数が1に戻った。
  - `done.txt` を作った後は `{}` を返した。
- 手順6の Claude Code では、「hello とだけ答えよ」という指示の後も停止が差し止められた。agent は `done.txt` を `ok` の内容で作ってから止まった。
- 手順8の Codex では、`hello` と答えた直後に `hook: Stop Blocked` が出た。agent は `printf 'ok\n' > done.txt` を実行し、2回目の Stop で止まった。既存の herdr の Stop hook は、同じ回に `Completed` になった。
- 手順8で `< /dev/null` を付けないと、`codex exec` は「Reading additional input from stdin...」と出して標準入力を待ち続け、5分で打ち切られた。
- 確かめていないこと：
  - agent が `until` を消したり書き換えたりして強制を外すか。
  - 完了判定のコマンドが実際のテストスイートのように時間がかかる場合の振る舞い。

## 見直す条件

- Claude Code か Codex の Stop hook の入出力が変わったとき。特に、Codex が連続続行の上限を文書化したとき。
- `~/.codex/hooks.json` の herdr の Stop hook が `continue: false` を返すようになったとき。続行の判定より優先される。
- agent が `until` を消して強制を外す例を観察したとき。`until` への書き込みを PreToolUse hook で拒否する案を検討する。
- このスクリプトを `~/.claude/hooks/` に入れたとき。正本をそちらに移し、このページのスクリプトをリンクに置き換える。
