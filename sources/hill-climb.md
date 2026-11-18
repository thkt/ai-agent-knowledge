# Hill-Climb の考え方と根拠（2026-11）

Hill-Climb は、coding agent が出す「完了」を、スクリプトが測ったスコアで裏付けさせる skill である。その仕組みを実装と照合した。あわせて、Claude Code と Codex の Stop hook で停止の判定を共通化する案を組み立てた。どちらも自分たちが採用した判断ではない。

## TL;DR

- スコアが改善し、guard がすべて通った変更だけを commit し、それ以外は作業ツリーごと戻す。
- スクリプトが防ぐのは、ロック対象の改変、改善しない変更、guard の失敗の3つだけである。期待値のハードコードは検出しない。
- README の実行例の数値は自己申告で、再現していない。Stop hook で停止の判定を共通化する案は、採用していない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Bil0000/Hill-Climb](https://github.com/Bil0000/Hill-Climb)（Claude Code plugin と、`SKILL.md` を読む agent 向けの skill） |
| 著者 | Bil0000 |
| 公開日 | リポジトリ作成は 2026-11-02。v1.0.0 は 2026-11-02、v1.1.0 は 2026-11-12 |
| 照合した版 | `main` の `84b9590`（2026-11-12、v1.1.0） |
| 照合日 | 2026-11-13 |
| 照合した範囲 | README、`SKILL.md`、`climb.py`、`judge.py`、`examples/demo.sh`、Issue #12。Claude Code と Codex の hooks の公式ドキュメント |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 種別 |
| --- | --- | --- |
| README | [README.md](https://github.com/Bil0000/Hill-Climb/blob/84b95909383379d4a84ad02ecadf928aba6b3b9b/README.md) | 問題設定、導入手順、実行例 |
| SKILL | [skills/hill-climb/SKILL.md](https://github.com/Bil0000/Hill-Climb/blob/84b95909383379d4a84ad02ecadf928aba6b3b9b/skills/hill-climb/SKILL.md) | agent が従う手順 |
| climb | [skills/hill-climb/scripts/climb.py](https://github.com/Bil0000/Hill-Climb/blob/84b95909383379d4a84ad02ecadf928aba6b3b9b/skills/hill-climb/scripts/climb.py) | スコアを測り、変更を commit するか戻すかを決めるスクリプト |
| judge | [skills/hill-climb/scripts/judge.py](https://github.com/Bil0000/Hill-Climb/blob/84b95909383379d4a84ad02ecadf928aba6b3b9b/skills/hill-climb/scripts/judge.py) | 正解が1つに決まらない対象を、モデルの採点で数値にするスクリプト |
| #12 | [Holdout is readable by the climber](https://github.com/Bil0000/Hill-Climb/issues/12) | holdout を climber が読めることを指摘する Issue。2026-11-13 時点で open |
| cc-hooks | [Claude Code: Hooks reference](https://code.claude.com/docs/en/hooks) の Stop 節 | Claude Code の Stop hook の入出力 |
| cc-goal | [Claude Code: /goal](https://code.claude.com/docs/en/goal) | `/goal` の評価のしかた |
| codex-hooks | [Codex: Hooks](https://developers.openai.com/codex/hooks) の Stop 節 | Codex の Stop hook の入出力 |

## 中心の主張

**agent は変更を書き、残すかどうかはスクリプトが決める。** agent が `climb.py try "<案>"` を呼ぶたびに、climb はチェックを実行してスコアを測る。スコアが改善し、guard がすべて通った変更だけを commit し、それ以外は作業ツリーごと戻す。

README は、agent に監査で見つかったバグ 12 件を直させたときに起きた次の4つを、この仕組みで防ぐ対象に挙げる。

- テストの期待値を書き換えて「直した」ことにする
- 頼んでいない既存機能を壊す
- context が compact されると、試した案を忘れて同じ案を繰り返す
- 変更が1つの大きな diff になり、どの変更がどれを直したか追えない

## 考え方と根拠

| 考え方 | 資料と根拠 | 照合の結果 |
| --- | --- | --- |
| ロック対象（テスト、ベンチマーク、fixture）が変わったら採点しない | README。climb の `fingerprint`（L237-249）がロック対象を sha256 で要約し、`cmd_try`（L666-670）が init 時の値と比べる | 実装がある。採点の後にも、チェックや guard がロック対象を書き換えていないかを比べ直す（L728-732） |
| スコアが改善しない変更は作業ツリーごと戻す | README。climb の `undo`（L331-334） | 実装がある。`git reset --hard HEAD` と `git clean -fdq` を、climb が触る全リポジトリで実行する。新規ファイルも消える |
| スコアが改善しても、guard（既存テスト、typecheck、lint）が1つでも落ちたら戻す | README。climb の `scored_step`（L716-723） | 実装がある。guard を指定しないと、init が警告を出す（L636-639） |
| 採点したコードだけを、1 step 1 commit で残す | README。climb の `snapshot`（L337-）と `commit`（L392-） | 実装がある。採点前に専用の index へ作業ツリーを記録し、その tree を commit する。commit は `--no-verify` で、pre-commit hook を通らない（L400） |
| 状態は context ではなくディスクに置く | README。SKILL「Running for a long time」（L504-511） | 実装がある。card は `card.json`、step の記録は `log.jsonl`、チェックの出力は `last.txt` に書く。`status` が呼ばれるたびにそれらを読み直す |
| まぐれの改善を勝ちと数えない | README。climb の `measure`（L149-168）と `better`（L259-261） | 実装がある。`--runs N` は1回目をウォームアップとして捨て、残りの中央値を取る。`--min-gain` は数値か割合で、改善の下限を決める |
| 改善が止まったら blocker review を求める | README。climb の `cmd_status`（L806-811） | 実装がある。改善のない step が5回続くと `PLATEAU` を出す。review を2回記録しても改善がなければ `STUCK` を出す |
| 正解が1つに決まらない対象もスコアにできる | README。judge | 実装がある。`claude -p` と `codex exec` を読み取り専用で呼び、10点満点の採点の中央値を最終行に出す |
| 小さな試行で効果が出た | README の Real runs。12件の失敗 → 0、190.5 ms → 0.66 ms | 自己申告で、再現していない。README 自身も「benchmark ではない」と書く |

## 限界

- **スクリプトが機械的に防ぐのは3つだけである。** ロック対象の改変、スコアが改善しない変更、guard の失敗である。テストの入力だけを特別扱いする実装や、期待値のハードコードは検出しない。SKILL「Never cheat the check」（L365-378）も、`try` が keep しても失敗とみなす、と書く。
- **holdout を climber が読める。** #12 によれば、holdout は repo の中の `.hill-climb/<name>/holdout/` に置かれる。`--lock` は書き換えを止めるが、読むことは止めない。SKILL は読むことを禁じるが、climb にそれを強制する実装は無い。
- **チェックがクラッシュしても勝ちになり得る。** `--direction min` と `--score` を組み合わせたときに起きる。チェックが exit 0 で終わり、正規表現に一致する行を出さないと、スコアは 0 になる（L138-139）。guard を指定しないと init が警告を出すのは、このためである（L637）。

## 停止を誰が決めるか

Hill-Climb は orchestration engine を持たない。step を回すのは agent 自身のループで、climb はそのループから毎回呼ばれる判定役である。

| 役割 | 担当 | 根拠 |
| --- | --- | --- |
| 次に何を試すか | agent が SKILL §4 の手順に従って決める。1 step は「1案、1変更、1回の `try`」 | SKILL L334-358 |
| 変更を残すか戻すか | climb が決め、agent は介入できない | climb `cmd_try`、`scored_step` |
| 止めどき | climb が出力する `BAR REACHED`、`STUCK`、`Budget used` を agent が読んで判断する | climb L661、L806-809 |
| セッションをまたいで続けること | クライアントの goal モードか loop モードに、「status が bar reached か STUCK と言うまで続けよ」というプロンプトを渡す | SKILL L507-510 |

止めどきの判断が agent に残っているため、agent は `status` を読まずに「完了」と言って止まれる。

## Stop hook で停止の判定を共通化する案（未採用・未検証）

この節は、照合者が上の照合結果から組み立てた案である。どの設定にも入れていない。スクリプトの判定だけを手元で確かめ、CLI に登録しての検証はしていない。

この案を汎用化し、Claude Code と Codex に登録して検証した結果は、[stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) にある。

### 判定役を Stop hook に置く理由

Claude Code と Codex は、どちらも Stop hook で agent の停止を差し止められる。hook が `climb.py status` を自分で実行して終了を判定すれば、止めどきの判断を agent から取り上げられる。

| 項目 | Claude Code（cc-hooks） | Codex（codex-hooks） |
| --- | --- | --- |
| 続行させる出力 | stdout に `{"decision":"block","reason":"..."}`。exit 2 と stderr でもよい | 同じ |
| `reason` の扱い | Claude に、続ける理由として渡る | 新しい user prompt として送られる |
| exit 0 のときの stdout | JSON か空 | JSON が必要。プレーンテキストは不正になる |
| 入力の `stop_hook_active` | Stop hook による続行の最中なら true | この turn がすでに Stop で続行されていれば true |
| 連続続行の上限 | 8回。Claude がツールを呼ぶたびにリセットされる。`CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` で変えられる | 照合したページに記述が無い。未確認 |
| 実行前の承認 | settings ファイルの hook と同じ workspace trust | `/hooks` で hook ごとに trust する。`<repo>/.codex/` に置くなら project の trust も要る。一回限りの実行は `--dangerously-bypass-hook-trust` で省ける |
| 置き場所 | `.claude/settings.json` か `~/.claude/settings.json` の `hooks.Stop` | `<repo>/.codex/hooks.json` か `~/.codex/hooks.json` の `hooks.Stop` |
| 同じ event に複数の hook がある場合 | 未照合 | 同時に起動する。どれかが `continue: false` を返すと、続行の判定より優先される |

両方に同じ1本のスクリプトを登録すれば、判定の規則は1か所にまとまる。

### `/goal` を使わない理由

cc-goal によれば、`/goal` は prompt-based の Stop hook である。評価するのは軽量なモデルで、会話の記録だけを読み、コマンドを実行しない。そのため、agent の出力が評価モデルを「完了した」と納得させれば止まれる。

command の Stop hook は `climb.py status` を自分で実行し、ディスク上の状態から判定する。SKILL L507-510 が勧める goal モードより厳しい。

### 判定の規則

| 条件 | 判定 | 理由 |
| --- | --- | --- |
| `.hill-climb/enforce` が無い | 止めてよい | Stop hook は scope 内の全セッションで動く。climb 中でない作業を止めないよう、強制は opt-in にする |
| 実行中のバックグラウンドタスクがある（cc の `background_tasks`） | 止めてよい | SKILL L352-356 は、時間のかかる `try` をバックグラウンドで実行し、終了通知を待つよう書く。待っている間に続行させると、`try` と次の編集が重なる |
| `card.json` に `hard` があり、step 数か経過時間が上限に達した | 止めてよい | `status` は予算切れを出力しない。予算切れは `try` が `Fail` を投げるだけで、`log.jsonl` にも残らない（climb L655-661） |
| `status` の出力に `BAR REACHED` か `STUCK` の行がある | 止めてよい | climb L806-809 が出す終了の合図である |
| `status` の出力に `PLATEAU` の行がある | 続行させる。理由として blocker review と `climb.py review` を求める | review を記録しないと `STUCK` に届かず、climb の側からは終われない |
| 上のどれにも当たらない | 続行させる | climb が終了を出していない |
| `status` が失敗した。または step 数が増えないまま8回続けて差し止めた | 止めてよい。`systemMessage` で警告を出す | 黙って止めると強制が消えたことに気づけない。止めないと、進まない差し止めが続く。Codex の上限は未確認なので、cc と同じ8回を自前で数える |

予算の判定は `--hard` の予算だけを対象にする。`--hard` が無い予算では、climb の `gaining`（L297-302）が直近の改善を見て続行を許すことがある。これを hook に写すと同じ規則が2か所にでき、climb の更新に hook が追従できなくなる。

### スクリプト案

`~/.claude/hooks/climb-stop.py` と `~/.codex/hooks/climb-stop.py` のどちらかに置き、両方の CLI から同じ1本を呼ぶ。

```python
#!/usr/bin/env python3
"""Stop hook for Claude Code and Codex: keep a hill climb running until climb.py says it is over."""
import json
import os
import re
import subprocess
import sys
import time
from pathlib import Path

CAP = 8  # same limit Claude Code applies; Codex documents none


def reply(payload):
    print(json.dumps(payload))  # Codex rejects plain text on exit 0
    sys.exit(0)


def warn_and_allow(message):
    reply({"systemMessage": f"climb-stop: {message}"})


event = json.load(sys.stdin)
if event.get("background_tasks"):
    reply({})
top = subprocess.run(["git", "rev-parse", "--show-toplevel"], cwd=event.get("cwd") or ".",
                     capture_output=True, text=True)
marker = Path(top.stdout.strip(), ".hill-climb", "enforce")
if top.returncode or not marker.is_file():
    reply({})
name = marker.read_text().strip()
folder = marker.parent / name
try:
    card = json.loads((folder / "card.json").read_text())
    log = folder / "log.jsonl"
    steps = sum(json.loads(line)["kind"] == "step" for line in log.read_text().splitlines() if line) if log.exists() else 0
except (OSError, ValueError, KeyError) as error:
    warn_and_allow(f"cannot read climb {name!r}: {error}")

# Copy of the hard-budget rule in climb.py cmd_try (L655-661). climb.py is canonical.
hours = (time.time() - card["started"]) / 3600 if card.get("started") else 0
if card.get("hard") and ((card.get("max_steps") and steps >= card["max_steps"])
                         or (card.get("max_hours") and hours >= card["max_hours"])):
    reply({})

status = subprocess.run([sys.executable, os.environ["CLIMB"], "status", name],
                        cwd=marker.parent.parent, capture_output=True, text=True)
if status.returncode:
    warn_and_allow(f"climb.py status failed: {status.stderr.strip()[-300:]}")
if re.search(r"^(BAR REACHED|STUCK)", status.stdout, re.M):
    reply({})

state_file = marker.with_name("enforce-state.json")
state = json.loads(state_file.read_text()) if state_file.exists() else {}
blocks = state.get("blocks", 0) + 1 if state.get("steps") == steps else 1
if blocks > CAP:
    state_file.unlink(missing_ok=True)
    warn_and_allow(f"{CAP} blocks in a row with no new step; letting the agent stop")
state_file.write_text(json.dumps({"steps": steps, "blocks": blocks}))

if re.search(r"^PLATEAU", status.stdout, re.M):
    reply({"decision": "block", "reason": "PLATEAU. Do a blocker review, log it with climb.py review, then continue with the next try."})
reply({"decision": "block", "reason": "The climb is not over. Read climb.py status and continue with the next try."})
```

### 登録と使い方

1. スクリプトを置き、実行権限を付ける。
2. Claude Code の `hooks.Stop` に次の hook を足す。`CLIMB` には、導入した `climb.py` の実際のパスを入れる。plugin で導入した場合は、`~/.claude/skills/` の下ではなく plugin のディレクトリの下にある。

   ```json
   {"type": "command", "command": "CLIMB=$HOME/.claude/skills/hill-climb/scripts/climb.py python3 $HOME/.claude/hooks/climb-stop.py", "timeout": 30}
   ```

3. Codex の `hooks.Stop` に同じ command を足し、`/hooks` で trust する。
4. climb を `--hard` と `--max-steps` か `--max-hours` を付けて `init` する。
5. 強制を始めるときは、`echo <climb 名> > .hill-climb/enforce` を実行する。climb は `/.hill-climb/` を `info/exclude` に足す（L413-420）ため、このファイルは作業ツリーを汚さない。
6. 強制をやめるときは、`.hill-climb/enforce` を消す。

### 既存の設定への影響

| 対象 | 影響 |
| --- | --- |
| `~/.claude/settings.json` | 2026-11-13 時点で、hooks は空である。追加する Stop hook はこれだけになる |
| `~/.codex/hooks.json` | 2026-11-13 時点で、Stop に `herdr-agent-state.sh idle` が登録されている。Codex は両方を同時に起動する。herdr のスクリプトには `continue` も `echo` も `printf` も無いため、続行の判定を上書きしないと見込む。herdr を変えたら確かめ直す |
| climb の kit | `enforce` と `enforce-state.json` は `.hill-climb/` の直下に置く。climb ごとのディレクトリには置かないため、`save_kit`（L443-449）が kit に取り込まない |
| 予算の規則 | hook の予算判定は climb L655-661 の写しである。climb の予算の規則が変わったら、hook も合わせる |

### 検証の手順

1. `examples/demo.sh` の1〜56行目を、`trap` の行を除いて実行する。一時ディレクトリが消えない状態で climb `cart-audit` が作られる。
2. その repo で `echo cart-audit > .hill-climb/enforce` を実行する。
3. Claude Code を起動し、`try` を1回も実行せずに turn を終えさせる。Stop hook が続行を差し止め、`reason` が届くことを確かめる。
4. 何もせずに止まろうとする turn を9回続けさせる。9回目に `systemMessage` の警告が出て止まることを確かめる。
5. `cart.py` を直して `BAR REACHED` まで進める。そこで止まれることを確かめる。
6. Codex で `/hooks` から trust した後、3〜5を繰り返す。

## 照合の結果

- **版：** `main` の HEAD は `84b9590`（2026-11-12）で、v1.1.0 の release commit である。
- **README の主張と実装：** 「ロック対象の改変の拒否」「改善しない変更の巻き戻し」「guard の失敗による巻き戻し」「1 step 1 commit」「状態のディスク保存」には、climb にそれぞれ実装がある。行番号は「考え方と根拠」に書いた。
- **自己申告で再現していないもの：** README の Real runs の数値。
- **Claude Code と Codex への対応：** 確認の質問のツールは、SKILL L68-69 が Claude Code の `AskUserQuestion` と Codex の `request_user_input` を書き分けている。subagent の無い harness では、同じ作業を inline で行うよう書く（L155）。Codex で実際に動かした記録は、README にも Issue にも無い。
- **Stop hook の仕様：** cc-hooks の Stop 節と codex-hooks の Stop 節を読んだ。続行の出力形式、`stop_hook_active`、Codex の JSON の要件、cc の8回の上限は、「判定役を Stop hook に置く理由」の表に書いたとおりである。Codex の連続続行の上限は、照合したページに記述が見つからなかった。
- **案の検証：** スクリプト案の判定は、Stop hook の入力 JSON を手で渡して確かめた。CLI に登録しての検証はしていない。確かめた結果は次のとおり。
  - climb は `examples/demo.sh` の1〜56行目から `trap` を除いて作った。
  - `enforce` が無いと `{}` を返した。
  - `enforce` があり、終了の合図が無いと `decision: block` を返した。
  - `background_tasks` が空でないと `{}` を返した。
  - step が増えないまま9回目の呼び出しで、`systemMessage` を付けて止めるのを許した。
  - `try` で `BAR REACHED` に達した後は `{}` を返した。
  - Claude Code と Codex に登録しての確認は「検証の手順」の3〜6で、まだ行っていない。

## 照合の手順

1. 版を固定する：`gh api repos/Bil0000/Hill-Climb/commits/main --jq .sha`
2. 公開日と release を読む：`gh api repos/Bil0000/Hill-Climb --jq .created_at` と `gh api repos/Bil0000/Hill-Climb/releases --jq '.[] | [.tag_name,.published_at] | @tsv'`
3. 構成と README を読む：`scout repo-overview Bil0000/Hill-Climb` と `scout repo-tree Bil0000/Hill-Climb`
4. スクリプトと SKILL を読む：`scout repo-read Bil0000/Hill-Climb skills/hill-climb/scripts/climb.py`。`SKILL.md`、`judge.py`、`examples/demo.sh` も同じ方法で読む。
5. climb の判定箇所を探す：`grep -nE "def (undo|commit|snapshot|fingerprint|cmd_try|scored_step|cmd_status)|PLATEAU|STUCK|Budget used" climb.py`
6. Issue を読む：`gh issue view 12 -R Bil0000/Hill-Climb`
7. Codex の Stop 節を読む：`scout fetch https://developers.openai.com/codex/hooks` の「##### Stop」と「Common output fields」
8. Claude Code の Stop 節を読む：https://code.claude.com/docs/en/hooks の「Stop」と「Stop decision control」。`/goal` は https://code.claude.com/docs/en/goal の「How evaluation works」。hooks のページは約 290,000 文字あり、`scout fetch` は 100,000 バイトで切れる。WebFetch を `offset` 付きで使う。
9. 手元の Stop hook を確かめる：`python3 -c "import json; print(json.load(open('$HOME/.claude/settings.json')).get('hooks', {}).get('Stop'))"` と `cat ~/.codex/hooks.json`

## 見直す条件

- Hill-Climb の `main` に `84b9590` より後の commit が入ったとき。特に、climb の予算の規則（L655-661）、`status` の終了の合図（L806-809）、`.hill-climb/` の配置が変わったとき。
- #12 が閉じたとき。holdout の置き場所が repo の外に移ると、「限界」の2つ目が変わる。
- Claude Code か Codex の Stop hook の仕様が変わったとき。特に、Codex が連続続行の上限を文書化したとき。
- `~/.codex/hooks.json` か `~/.claude/settings.json` の Stop hook を変えたとき。
