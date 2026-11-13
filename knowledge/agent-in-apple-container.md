# apple/container の通常の container で agent を動かすと、ホストで触れられるのはマウントした worktree だけになる

## 結論

Claude Code を apple/container の通常の container（`container run`）で動かすと、agent がホストで触れられるのはマウントした worktree だけになった。認証は container の中でログインし、名前付き volume に残した。[Stop hook のループ](stop-hook-done-command.md)も container の中でそのまま動いた。

この構成は検証だけで、日々の作業には使っていない。

## 採用した理由

ホストの sandbox（macOS の Seatbelt）とは別の層で、agent が触れる範囲を VM の境界で決められる。

| できるようになったこと | 仕組み |
| --- | --- |
| agent からホストの `$HOME` を隠す | container ごとに VM が立ち、`-v` でマウントしたディレクトリだけが VM に入る |
| ホストの Keychain や `~/.claude` を渡さずに認証する | container の中で `claude auth login` を実行し、認証情報を volume `agent-claude` に残す。トークンはチャットにもホストのファイルにも出ない |
| 承認を求めない実行の影響を、worktree の中に限る | Codex のヘルプは `--dangerously-bypass-approvals-and-sandbox` を「外側で sandbox された環境のためのもの」と書く。container がその外側の sandbox に当たる |
| hook をホストから配る | hook のスクリプトを `readonly` でマウントし、`--settings` で登録する。agent が書き換えを試みたときの挙動は確かめていない |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| `container machine` | ホームを読み書き可能でマウントし、SSH agent を転送し、Linux capability をすべて付け、passwordless sudo を設定する（[apple-container.md](../sources/apple-container.md)）。agent を閉じ込める用途には合わない |
| `claude setup-token` のトークンを環境変数で渡す | トークンを秘密入力欄からホストを経由して渡すことになる。container の中でログインすれば、その経路が要らない |
| ホストの `~/.codex/auth.json` を複製してマウントする | ChatGPT ログインの refresh token ごと渡すことになる。Codex にも `codex login --device-auth` があり、container の中でログインできる |
| `--internal` のネットワークで外への通信を止める | apple/container の Issue #2062 が open で、ホストで IP forwarding と NAT が有効だと外への TCP が通る。agent は API に HTTPS でつなぐため、外への通信を止めると agent 自体が動かない。通信先を絞るには別の手段が要る |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-13 |
| モデル | `claude-haiku-5-5`（`--model haiku`） |
| ツール | apple/container の CLI 1.1.0（commit `5973b9c`）、macOS 27.2.1、image の中の Claude Code 2.1.323 と codex-cli 0.169.0 |

### 再現手順

1. `container system start --enable-kernel-install` を実行する。
2. 次の Dockerfile から `container build -t local/agent-sandbox:poc .` で image を作る。

   ```dockerfile
   FROM node:22-bookworm-slim
   RUN apt-get update && apt-get install -y --no-install-recommends git ripgrep ca-certificates python3 && rm -rf /var/lib/apt/lists/*
   RUN npm install -g @anthropic-ai/claude-code @openai/codex && npm cache clean --force
   RUN useradd -m -u 501 agent
   USER agent
   WORKDIR /work
   ```

3. `container volume create agent-claude` を実行する。
4. volume の所有者を agent に変える：`container run --rm -u 0 -v agent-claude:/home/agent/.claude local/agent-sandbox:poc chown -R 501:1001 /home/agent/.claude`
5. ターミナルでログインする：`container run -it --rm -e CLAUDE_CONFIG_DIR=/home/agent/.claude -v agent-claude:/home/agent/.claude local/agent-sandbox:poc claude auth login`
6. 空の git repo を作り、[Stop hook のページ](stop-hook-done-command.md)の再現手順3と同じ `until` を `.git/agent-loop/until` に書く。
7. Stop hook のスクリプトと、それを登録する `settings.json` を1つのディレクトリに置く。`settings.json` の command は `python3 /opt/hooks/agent-loop-stop.py` にする。
8. 次を実行する。

   ```bash
   container run --rm -e CLAUDE_CONFIG_DIR=/home/agent/.claude \
     -v agent-claude:/home/agent/.claude -v <repo>:/work \
     --mount type=bind,source=<hooks>,target=/opt/hooks,readonly \
     local/agent-sandbox:poc claude -p --model haiku --permission-mode acceptEdits \
     --settings /opt/hooks/settings.json \
     "Reply with the single word hello. Do not create files unless asked."
   ```

9. ホスト側で `<repo>/done.txt` を読む。
10. 分離と書き込みの速さを確かめる：`container run --rm -v <dir>:/work alpine sh -c 'ls /Users; ...'`。小さいファイル2000個の作成と200MBの書き込みを、`/tmp` とマウント先で busybox の `time` で測る。

### 結果

- 手順8は9.5秒で終わった。container の起動を含む。
  - agent は「hello」とだけ答えて止まろうとし、Stop hook に差し止められた。
  - その後 `done.txt` を `ok` の内容で作って止まった。
  - ホスト側の `done.txt` の所有者は `thkt` だった。container の中では、マウントしたファイルは `agent` の所有に見え、`git` も動いた。
- 手順5の後、`claude auth status` は `loggedIn: true`、`authMethod: claude.ai` を返した。
- 手順10では次を確かめた。
  - container から `/Users` は見えなかった。
  - マウントは virtiofs だった。
  - 小さいファイル2000個の作成は、マウント先で0.38秒、`/tmp` で0.02秒だった。
  - 200MBの書き込みは、マウント先で0.16秒、`/tmp` で0.05秒だった。
- ホストで VPN を有効にしていると、container から外への HTTPS が TLS のハンドシェイクでリセットされた。HTTP は通った。VPN を切ると、`api.anthropic.com` に TLS でつながった。
- volume は作った直後は root の所有で、agent ユーザーは書き込めなかった。手順4が要る。
- 確かめていないこと：
  - `--dangerously-skip-permissions` での実行。こちらの権限ルールで拒否されたため、`acceptEdits` で代わりに行った。
  - Codex を container の中で動かすこと。
  - 外への通信の行き先を絞る方法。
  - 実際の repo（依存のインストールや長いテスト）での速さ。

## 見直す条件

- apple/container の CLI を 1.1.0 から上げたとき。
- apple/container の Issue #2062 が閉じたとき。`--internal` で通信を止められるかが変わる。
- VPN を使ったまま container の HTTPS が通る設定が見つかったとき。
- Claude Code か Codex の認証方式が変わったとき。特に、`CLAUDE_CONFIG_DIR` の扱いが変わったとき。
- `--dangerously-skip-permissions` を container の中で実行して確かめたとき。
