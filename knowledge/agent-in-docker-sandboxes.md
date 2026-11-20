# Docker Sandboxes で agent を動かすと、外への通信を宛先ごとに許可でき、Claude のログインのトークンは VM に入らない

## TL;DR

- `sbx policy init deny-all` のもとでは、許可していない宛先への HTTPS、HTTP、IP 宛ての TCP、DNS、UDP がすべて止まった。どの拒否も `sbx policy log` に残った。宛先を1つ許可すると、その宛先だけが通った。
- Claude のサブスクリプションで `/login` しても、VM の中の `~/.claude/.credentials.json` に入っていたのは `sk-ant-oat01-proxy-managed` という代わりの値だった。本物はホスト側に置かれ、その状態で VM の中の `claude -p` が応答した。
- [apple/container での検証](agent-in-apple-container.md)で残った2つの課題を、Docker Sandboxes は埋める。2つとは、通信先を絞ることと、認証情報を agent の環境の外に置くことである。
- 代わりに、VM には自分の `~/.claude` が入らず、sbx が用意した設定が入る。その設定は、権限の確認を飛ばす bypass を承認済みにしてある。VM の `~/.claude.json` には、アカウントのプロフィール（メールアドレスなど）の項目もある。
- SSH エージェントの転送は、ホストのエージェントに鍵が無かったため確かめていない。検証日は 2026-11-20、sbx は 0.54.0、モデルは `claude-haiku-5-5`。

## 結論

agent を閉じ込める環境として、Docker Sandboxes（CLI は `sbx`）は、外への通信を宛先ごとに許可でき、Claude のログインのトークンを VM の外に置ける。この構成は検証だけで、日々の作業には使っていない。

## 採用した理由

apple/container では、通信先を絞れず、トークンを container の volume に置く必要があった。Docker Sandboxes では、どちらもホスト側のプロキシが受け持つ。

| できるようになったこと | 仕組み |
| --- | --- |
| 外への通信を、宛先ごとに許可する | `sbx policy init deny-all` で全拒否から始め、`sbx policy allow network <host> --sandbox <name>` で宛先を足す。DNS もサンドボックスの中のリゾルバが同じポリシーで判定する |
| 止めた通信を、後から確かめる | `sbx policy log <name>` に、宛先、経路、どの規則で止めたかが残る |
| Claude のログインのトークンを VM の外に置く | `/login` の結果は、ホストの secret store に `anthropic (oauth configured)` として入る。VM の中には代わりの値だけが置かれ、プロキシが送信時に本物へ差し替える |
| agent の種類に要る許可を、サンドボックス単位で足す | Claude 用の kit は、作ったサンドボックスにだけ Anthropic の宛先の許可を足す。shell のサンドボックスからは `api.anthropic.com` も 403 だった |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| apple/container で続ける | `--internal` のネットワークは Issue #2062 で外への TCP が漏れ、通信先を絞れなかった。トークンは container の volume に置く必要があった（[agent-in-apple-container.md](agent-in-apple-container.md)） |
| `sbx policy init balanced` で始める | よく使う開発の宛先をまとめて許可するので、何が通るかを確かめにくい。docs は、既定の許可に `*.googleapis.com` のような広いワイルドカードが入ると書く。今回は deny-all で始め、許可を1つずつ足した |
| `sbx` を Claude Code の中から初めて起動する | sbx の CLI は、デーモンが止まっていると自分で起動する。Claude Code の中から呼ぶと、デーモンが `CLAUDE_CODE_*` の環境変数を引き継いだ。今回は、最小限の環境変数で `sbx daemon restart` してから確かめた |

## 検証

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-20 |
| モデル | `claude-haiku-5-5`（`claude -p --model claude-haiku-5-5`） |
| ツール | sbx 0.54.0（Homebrew の `docker/tap/sbx`）、macOS 27.2.1（Apple silicon）、VM の中の Claude Code 2.1.329 |

### 再現手順

1. sbx を入れる：`brew trust docker/tap` を実行し、続けて `brew install docker/tap/sbx` を実行する。
2. Docker アカウントでサインインする：`sbx login`
3. 通信ポリシーを全拒否で初期化する：`sbx policy init deny-all`
4. 中身が README だけの git repo を `<repo>` に作る。
5. shell のサンドボックスを作る：`sbx create --name trial-shell shell <repo>`
6. 中から外へつなぐ：`sbx exec trial-shell -- bash -lc '<コマンド>'` で、次を1つずつ実行する。
   - `curl -sS -o /dev/null -w "%{http_code}\n" https://example.com`（`http://example.com` と `-k https://1.1.1.1` も同じ形で試す）
   - `ssh -o BatchMode=yes -T git@github.com`
   - `nslookup example.com 1.1.1.1`
7. 止めた記録を読む：`sbx policy log trial-shell`
8. 1つだけ許可する：`sbx policy allow network example.com --sandbox trial-shell`。続けて、手順6の `https://example.com` と、`https://example.org` を試す。
9. Claude のサンドボックスを作る：`sbx create --name trial-claude claude <repo>`。`sbx policy ls` と `sbx policy inspect <kit の policy ID>` で、kit が足した許可を見る。
10. ターミナルで `sbx run --name trial-claude` を実行し、Claude Code の中で `/login` を済ませて `/exit` で抜ける。
11. VM の中のトークンを、値を表示せずに確かめる。

    ```bash
    sbx exec trial-claude -- python3 -c "
    import json,os
    c=json.load(open(os.path.expanduser('~/.claude/.credentials.json')))['claudeAiOauth']
    for k in ('accessToken','refreshToken'): print(k, c[k][13:]=='proxy-managed', len(c[k]))"
    ```

12. ホスト側の置き場所を見る：`sbx secret ls`
13. VM の中から Claude を呼ぶ：`sbx exec trial-claude -- claude -p "Reply with exactly: ok" --model claude-haiku-5-5`
14. 片付ける：`sbx rm -f trial-shell trial-claude`

### 結果

外への通信は、次のとおりだった。

| 手順 | 結果 | `sbx policy log` の記録 |
| --- | --- | --- |
| `https://example.com` | 403。本文は「Approval required for example.com:443」 | `example.com:443`、forward proxy、該当する規則なし |
| `http://example.com` | 403 | `example.com:80`、forward proxy |
| `-k https://1.1.1.1` | 403。curl が報告した接続先は `1.1.1.1` ではなく VM の内部のアドレスで、プロキシが受けていた | `1.1.1.1:443`、forward proxy |
| `ssh -T git@github.com` | 名前を解決できずに失敗した | `github.com`、DNS proxy |
| `nslookup example.com 1.1.1.1` | DNS サーバーに届かなかった | `1.1.1.1:53`、UDP proxy |
| `https://api.anthropic.com`（shell から） | 403 | `api.anthropic.com:443` |
| 手順8の後の `https://example.com` | 200 | 確かめていない |
| 手順8の後の `https://example.org` | 403 | 確かめていない |

- Claude 用の kit が足した許可は、`api.anthropic.com`、`claude.com`、`platform.claude.com`、`downloads.claude.ai`、`mcp-proxy.anthropic.com`、`deb.debian.org`、`dhi.io` だった。どれもそのサンドボックスにだけ効き、kit が管理するので CLI では編集できない。
- 手順11では、アクセストークンもリフレッシュトークンも長さ26で、`sk-ant-oat01-` または `sk-ant-ort01-` の後が `proxy-managed` だった。手順12は `anthropic (oauth configured)` を返し、手順13は `ok` を返した。
- VM の `~/.claude` には、sbx が用意したものが入っていた。
  - 446行の CLAUDE.md で、先頭は `<!-- sbx:guidance start -->` だった。
  - settings.json は `hooks` と `bypassPermissionsModeAccepted` を持っていた。
  - skills は空だった。ホストの `~/.claude` の rules、skills、hooks は入っていなかった。
- VM の `~/.claude.json` の `oauthAccount` は、メールアドレス、氏名、組織名のキーを持っていた。トークンは VM の外に置かれるが、アカウントのプロフィールは VM の中に入る。
- SSH エージェントの転送は既定で有効（`ssh.agentForwardingEnabled` が `true`）だった。ホストのエージェントには鍵が無く、デーモンのログは「エージェントには届くが鍵が無い」と記録した。VM の中では `SSH_AUTH_SOCK` が空で、`ssh-add -l` は終了コード2で失敗した。
- `sbx policy --help`（v0.54.0）は、宛先の名前で決める規則がすり抜けられる場合を3つ挙げる。いずれも、拒否した宛先が、許可した宛先と同じ IP アドレスを共有する場合である。
  - プロキシが復号しない接続では、暗号化された中の Host を確かめない。
  - CONNECT のトンネルを通すか、`HTTP_PROXY` を使わずに送る平文の HTTP は、接続ごとに最初のリクエストの Host しか確かめない。
  - 宛先の名前を示さない接続は、そのアドレスに解決した名前のどれかが許可されていれば通る。

  これら3つは再現していない。
- 2つのサンドボックスの image で、ディスクを 2.467GB 使った。
- 確かめていないこと：
  - `.credentials.json` 以外のファイルに、本物のトークンが書かれていないか。VM のホーム全体を探したが、探し方が隠しディレクトリを含んでいなかった。
  - SSH エージェントに鍵があるときに、VM から署名を頼めるか。`ssh.agentForwardingEnabled` を `false` にすると止まるか。
  - `--clone` での作業と、作業ツリーを読み書きでマウントしたときの Git hooks の扱い。
  - ホストの `~/.claude` の rules を VM に持ち込む方法。docs は、skills だけを `sbx skills import` で共有ストアに入れられると書く。
  - 実際の repo（依存のインストールや長いテスト）での速さ。

## 見直す条件

- sbx を 0.54.0 から上げたとき。特に、`sbx policy --help` のすり抜けの記述が変わったとき。
- SSH エージェントに鍵を入れて使うようになったとき。転送は既定で有効なので、`ssh.agentForwardingEnabled` を確かめ直す。
- Claude Code の認証方式が変わったとき。特に、`.credentials.json` の形が変わったとき。
- 日々の作業で使うと決めたとき。速さと、`~/.claude` の rules を持ち込む方法を確かめる。
