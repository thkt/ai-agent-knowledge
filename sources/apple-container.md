# apple/container の考え方と根拠（2026-11）

apple/container は、Mac で Linux の container を container ごとの軽量 VM で動かす CLI である。agent の実行環境を閉じ込める用途に使えるかを判断するため、その主張を文書と Issue で照合した。自分たちが採用した判断ではない。手元で動かした結果は [agent-in-apple-container.md](../knowledge/agent-in-apple-container.md) にある。

| 項目 | 内容 |
| --- | --- |
| 出典 | [apple/container](https://github.com/apple/container) |
| 著者・発行元 | Apple |
| 公開日 | 最新 release は 1.7.1（2026-11-06） |
| 照合した版 | `main` の `c05a560`（2026-11-12）。文書は main 向けで、release ごとの文書は tag 側にあると README が書く |
| 照合日 | 2026-11-13 |
| 照合した範囲 | README、`docs/technical-overview.md`、`docs/container-machine.md`、`docs/host-integration.md`、`docs/networking.md`、`docs/command-reference.md`、`skills/container/SKILL.md`、Issue #2062、PR #2509 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 種別 |
| --- | --- | --- |
| overview | [docs/technical-overview.md](https://github.com/apple/container/blob/c05a5604c5f784c41274ed8032def13e4aeb8e35/docs/technical-overview.md) | 仕組みと制約 |
| machine | [docs/container-machine.md](https://github.com/apple/container/blob/c05a5604c5f784c41274ed8032def13e4aeb8e35/docs/container-machine.md) | 開発環境向けの container machine |
| host | [docs/host-integration.md](https://github.com/apple/container/blob/c05a5604c5f784c41274ed8032def13e4aeb8e35/docs/host-integration.md) | SSH agent の転送と、container からホストのサービスへの接続 |
| network | [docs/networking.md](https://github.com/apple/container/blob/c05a5604c5f784c41274ed8032def13e4aeb8e35/docs/networking.md) | ネットワークと DNS |
| skill | [skills/container/SKILL.md](https://github.com/apple/container/blob/c05a5604c5f784c41274ed8032def13e4aeb8e35/skills/container/SKILL.md) | agent 向けの skill |
| #2062 | [--internal (hostOnly) networks allow arbitrary outbound TCP](https://github.com/apple/container/issues/2062) | `--internal` の分離の抜け道。2026-11-13 時点で open |
| #2509 | [fix: enforce TCP isolation on --internal networks](https://github.com/apple/container/pull/2509) | #2062 の修正 PR。2026-11-13 時点で open |

## 中心の主張

**container ごとに軽量 VM を1台立てる。** 一般的な方法は、Linux VM を1台立てて全 container をその中で動かす。overview によれば、apple/container は container ごとに VM を分けるため、次の性質を持つ。

- 分離：container ごとに VM と同じ分離が得られる。
- 共有するファイル：ホストのファイルは、その container にマウントした分だけが VM に入る。VM を共有する方式では、どの container で使うかもしれないファイルを、すべて VM にマウントしておく必要がある。
- 互換性：OCI image を使い、作り、registry と push / pull できる。

## 考え方と根拠

| 考え方 | 資料と根拠 | 照合の結果 |
| --- | --- | --- |
| container ごとに VM を立てる | overview。Virtualization framework と vmnet を使う | 原文に記述がある。手元の 1.1.0 では、`alpine` の container が約1.6秒で起動した |
| マウントした分だけがホストから見える | overview の Privacy | 手元の 1.1.0 で、worktree だけを `-v` でマウントした container から `/Users` が見えないことを確かめた |
| container machine は開発環境で、sandbox ではない | machine の冒頭の WARNING。ホームを読み書き可能で同じパスにマウントし、SSH agent を転送し、Linux capability をすべて付け、passwordless sudo を設定する | 原文に記述がある。文書は「信頼する image にだけ使う」と書く |
| `--internal` のネットワークは外へ出られない | `container network create --internal` のヘルプは「host-only に制限する」と書く | #2062 は、ホストで IP forwarding と NAT が有効だと、外への TCP が通ると報告する。default gateway が host-only でも設定されるためで、修正の #2509 は open である。手元のホストは `net.inet.ip.forwarding=0` で、TCP も DNS も外に出なかった |
| container から解放したメモリはホストに返らない | overview の「Releasing container memory to macOS」 | 原文に記述がある。メモリを多く使う container は、ときどき再起動が要ると書く |
| macOS 26 以降が対象である | README と overview | 原文に記述がある。macOS 15 では、container 間の通信と複数のネットワークが使えない。手元の macOS 27.2.1 では動いた |
| agent は Docker のコマンドから推測しない | skill は「サブコマンドを Docker から推測するな。`container <group> --help` で確かめよ」と書く | 原文に記述がある。誤ったサブコマンドは plugin loader に渡り、サービスが止まっているかのようなエラーを返すと書く |
| ホストのサービスへの接続は DNS の domain で行う | host。`sudo container system dns create <domain> --localhost <ip>` | 原文に記述がある。packet filter の規則は再起動で消え、Private Relay が無効になると書く。手元では試していない |

## 照合の結果

- **版：** 照合した文書は main の `c05a560` で、手元の CLI は 1.1.0（commit `5973b9c`）だった。最新 release の 1.7.1 とは版が離れている。`container machine` などの文書の機能は、1.1.0 にサブコマンドがあることだけを確かめた。
- **#2062 の再現：** 再現していない。手元のホストは IP forwarding が無効で、Issue が挙げる条件を満たさない。
- **手元の検証で文書に無かったこと：** ホストで VPN を有効にしていると、container から外への TLS のハンドシェイクが `Connection reset by peer` でリセットされた。HTTP は通り、container の MTU は最初から 1280 だった。VPN を切ると HTTPS が通った。どの VPN のどの設定が原因かは特定していない。

## 照合の手順

1. 版を固定する：`gh api repos/apple/container/commits/main --jq .sha`
2. release を読む：`gh api repos/apple/container/releases/latest --jq '.tag_name+" "+.published_at'`
3. README と構成を読む：`scout repo-overview apple/container` と `scout repo-tree apple/container`
4. 文書を読む：`scout repo-read apple/container docs/technical-overview.md`。`docs/container-machine.md`、`docs/host-integration.md`、`docs/networking.md`、`docs/command-reference.md`、`skills/container/SKILL.md` も同じ方法で読む。
5. Issue と PR の状態を読む：`gh issue view 2062 -R apple/container --json state,title` と `gh pr view 2509 -R apple/container --json state,title`
6. 手元の版を読む：`container --version` と `sw_vers -productVersion`
7. ホストの IP forwarding を読む：`sysctl -n net.inet.ip.forwarding`

## 見直す条件

- #2062 が閉じたとき。`--internal` で外への通信を止められるかが変わる。
- 手元の CLI を 1.1.0 から上げたとき。
- container の文書が、VPN を使うホストでの通信について記述を加えたとき。
