# vercel-labs/deepsec の、候補を正規表現で絞ってからエージェントに脆弱性を調べさせる仕組みの考え方と根拠（2026-11）

Vercel Labs の deepsec は、コーディングエージェントにコードベース全体の脆弱性を探させるハーネスである。README は、最高性能のモデルを最大の思考量で使い、大きなコードベースでは数千〜数万ドルかかると書く。文書が述べる仕組みを、リポジトリのコードで照合した。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- 中心の主張は、安い正規表現で候補の箇所を拾い、エージェントには候補のあるファイルだけを調べさせ、所見は別のパスで調べ直して誤検知を落とすことである。
- AI に作らせた検出規則は、コードとして実行せず、データとして検証してから使う。正規表現は長さ 500 文字までで、後方参照と後読みを拒否する。再スキャンで 500 ファイルを超える規則、または全ソースの 20% を超える規則は外す。入口を覆えない間は、費用のかかる調査を始めない。ここまではコードで確かめた。
- 文書との差もあった。再検証の「git の履歴を見る」は、そのファイルの直近 3 か月の `git log` 10 行をプロンプトに貼るだけで、直ったかどうかの判定はモデルに任されている。分析済みのファイルに新しい候補が増えても、既定の `process` は調べ直さない。
- 文書の数字（再検証後の HIGH 以上の誤検知率が約 10〜29%、Opus で 100 ファイルあたり $25〜60）を測るコードとテストは、リポジトリに無い。

| 項目 | 内容 |
| --- | --- |
| 出典 | [vercel-labs/deepsec](https://github.com/vercel-labs/deepsec) の README、`docs/architecture.md`、`docs/faq.md` |
| 著者・発行元 | Vercel Labs |
| 公開日 | リポジトリの作成は 2026-04-30 |
| 照合した版 | main `9444b89`（2026-11-19） |
| 照合日 | 2026-11-22 |
| 照合した範囲 | README、`docs/architecture.md`、`docs/faq.md` の全文と、下の表で引いたコード（`packages/deepsec/src/setup/`、`packages/scanner/src/`、`packages/processor/src/`、`packages/core/src/`、`packages/deepsec/src/sandbox/`） |
| この資料を根拠にした Issue | なし |

## 中心の主張

**費用の安い段階で候補を絞り、費用の高い段階では候補だけを調べ、所見は別のパスで確かめる。** `scan` は正規表現でファイルごとに候補の箇所を拾い、AI を使わない（2,000 ファイルで約 15 秒）。`process` は候補のあるファイルをまとめてエージェントに渡し、所見を JSON で返させる。`revalidate` は所見を読み直して、真か誤検知か、すでに直っているかを付ける。文書は、再検証で誤検知率がたいてい半分以下になると書く（[architecture.md L135](https://github.com/vercel-labs/deepsec/blob/9444b89/docs/architecture.md?plain=1#L135)）。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| AI に作らせた検出規則を、コードではなくデータとして受け取る | 規則は厳格な JSON として受け取り、生成されたコードを評価せずにコンパイルする。正規表現やグロブの複雑さ、例の欠け、slug の衝突、パスの外への参照、ヒットの爆発を拒否する（[architecture.md L74](https://github.com/vercel-labs/deepsec/blob/9444b89/docs/architecture.md?plain=1#L74)） | 実装されている。正規表現は 500 文字まで、グロブは 240 文字まで（[declarative-matcher.ts L6-L7](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/scanner/src/declarative-matcher.ts#L6-L7)）で、後方参照と後読みを拒否する（[L18](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/scanner/src/declarative-matcher.ts#L18)、[L20](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/scanner/src/declarative-matcher.ts#L20)）。差：ヒットの爆発はコンパイルの時点では調べない。再スキャンの後に、500 ファイルを超える規則と、全ソースの 20% を超える規則を外す（[coverage.ts L94-L96](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/setup/coverage.ts#L94-L96)）。ReDoS の検査は、よくある指数時間の形を拒否する保守的な検査だと、コード自身が書く（[L21-L23](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/scanner/src/declarative-matcher.ts#L21-L23)） |
| 入口を覆えない間は、費用のかかる調査を始めない | 規則の生成と再スキャンは最大 2 回で、覆えていない間は有料の調査を始めない（[architecture.md L78](https://github.com/vercel-labs/deepsec/blob/9444b89/docs/architecture.md?plain=1#L78)） | 実装されている。`maxAttempts = 2` で繰り返し（[coordinator.ts L619-L620](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/setup/coordinator.ts#L619-L620)）、覆えなければ `SCAN_COVERAGE_INSUFFICIENT` で止まる（[L781](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/setup/coordinator.ts#L781)）。`process` を呼ぶのはその後である（[L843](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/setup/coordinator.ts#L843)）。差：照合用のエージェントの報告では、回数は再開のたびに 0 に戻り、1 回の試行の中で修正の再依頼が 1 回ある |
| 作業の単位を 1 ファイルにし、記録は追記だけにする | 1 ファイルに 1 つの記録を持ち、再スキャンは候補を足し、再調査は所見を slug と題で重複を除いて足す。何も上書きせず消さない | 部分的に実装されている。所見の重複は slug と、前後の空白を除いて小文字にした題で判定する（[processor/index.ts L692](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/index.ts#L692)）。差：再スキャンは分析済みのファイルの状態を戻さない。コードは注釈だけで処理が無い（[scanner/index.ts L353-L354](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/scanner/src/index.ts#L353-L354)）。そのため、分析済みのファイルに増えた候補を、既定の `process` は調べない |
| 複数の作業者が同じファイルを取り合わない | `lockedByRunId` でファイルを原子的に確保する | 部分的に実装されている。原子性はプロジェクト単位の `mkdir` のロック（`.process.lock`、[run.ts L351](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/core/src/run.ts#L351)）で作り、その中で `lockedByRunId` を書く。照合用のエージェントの報告では、`--reinvestigate` やファイルの指定があると、他の実行が確保したファイルも取る |
| プロンプトをバッチごとに組み立てる | 検出した技術スタックに合わせてプロンプトを変える | 実装されている。共通部分に、バッチ内のファイルの言語で絞った技術スタックの注意と、ヒットした脆弱性の種類ごとの注意を足す（[processor/index.ts L256](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/index.ts#L256)、[L272](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/index.ts#L272)）。1 つの節が 6,000 文字を超えると要約に置き換える（[assemble.ts L14](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/prompt/assemble.ts#L14)） |
| 所見を別のパスで調べ直す | エージェントがコードを読み直し、git の履歴で直ったかを確かめ、true-positive、false-positive、fixed、uncertain を付ける | 部分的に実装されている。判定は 6 種類で、`accepted-risk` と `duplicate` もある（[schemas.ts L21-L28](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/core/src/schemas.ts#L21-L28)）。差：git の履歴は、そのファイルの `git log --oneline --since="3 months ago" -n 10` をプロンプトに貼るだけである（[shared.ts L1045](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/agents/shared.ts#L1045)）。直ったかどうかを調べるコードは無く、判定はモデルに任されている |
| エージェントを差し替え、所見をまとめる | codex（既定、gpt-5.5）、claude（Opus）、pi に対応し、プロンプトと出力の schema は共通。別のエージェントで調べ直した所見は重複を除いてまとめる | 実装されている。差：設定でモデルを指定しないとき、`process` はどのエージェントでも `claude-opus-4-8` を使う（[processor/index.ts L283](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/processor/src/index.ts#L283)）。重複の判定は slug と題だけで、どのエージェントの所見かは見ない |
| Sandbox で動かすとき、認証情報を外に置く | API キーは Sandbox の外で注入され、作業者の通信先はエージェントの API に絞られる。`.git` は送らない | 実装されている。Sandbox の中の値は代わりの文字列（`deepsec-sandbox-brokered-credential`）で、外へ出る通信の `Authorization` を firewall が本物に差し替える（[sandbox/setup.ts L51-L61](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/sandbox/setup.ts#L51-L61)）。送る tarball は `.git` を除く（[upload.ts L18](https://github.com/vercel-labs/deepsec/blob/9444b89/packages/deepsec/src/sandbox/upload.ts#L18)） |

## 文書の記述と実装の差

| 文書の記述 | 照合した版での状態 |
| --- | --- |
| 再検証後、HIGH 以上の誤検知率は約 10〜29%（[faq.md L108-L110](https://github.com/vercel-labs/deepsec/blob/9444b89/docs/faq.md?plain=1#L108-L110)） | 測るコードとテストは無い。`metrics` は利用者自身のデータから真と誤検知の数を数えるだけである |
| Opus で、100 ファイル $25〜60、500 ファイル $130〜300、2,000 ファイル $500〜1,200（[faq.md L68-L72](https://github.com/vercel-labs/deepsec/blob/9444b89/docs/faq.md?plain=1#L68-L72)） | 測るコードとテストは無い |
| 再検証は git の履歴で直ったかを確かめる | 直近 3 か月の `git log` 10 行を貼るだけである。Sandbox では `.git` を送らないので、Sandbox での再検証ではこの履歴も空になると考えられる（照合者の推測。動かしていない） |
| 再スキャンは候補を足す | 足すが、分析済みのファイルの状態は戻さないので、新しい候補は既定の `process` で調べられない |

## 限界

- **精度と費用の数字は、文書の記述だけである。** 手元で動かしておらず、リポジトリにも測定の記録は無い。
- **費用が大きい。** README は、大きなコードベースで数千〜数万ドルかかると書く。FAQ は、まず `--limit 50` で見積もるよう勧める。
- **deepsec 自体を、シェルを持つコーディングエージェントとして扱う必要がある。** README は、信頼できる入力（自分のソースコード）で動かす前提だと書く。依存や同梱のコードからのプロンプトインジェクションの懸念には、Sandbox で動かすことを勧める。
- **分散実行は Vercel Sandbox が前提である。** 手元だけで動かす方法もあるが、照合した範囲では Sandbox を使わない場合の通信の制限は書かれていない。

## 照合の結果

- **照合した主張：** 「考え方と根拠」と「文書の記述と実装の差」の各行は、引いた行で確かめた。「差」の欄のうち「照合用のエージェントの報告では」と書いた 2 か所は、エージェントの報告で、引いた行は確かめていない。
- **読んでいないもの：** `docs/` のうち getting-started、configuration、plugins、models、writing-matchers、data-layout、reviewing-changes。`samples/` と website。
- **範囲外：** 料金の経路（Vercel AI Gateway、各社の API キー）と、Vercel のプロジェクトへの接続。エージェントの運用の判断に関わらないため。

## 照合の手順

1. 照合した版を取得する：`git clone --depth 1 https://github.com/vercel-labs/deepsec.git`。`git log -1` が `9444b89` であることを確かめる。新しい版で照合し直すときは、その commit を記録する。
2. 文書を読む：README、`docs/architecture.md`、`docs/faq.md`。
3. 規則の検証を確かめる：`packages/scanner/src/declarative-matcher.ts` の上限と拒否の条件、`packages/deepsec/src/setup/coverage.ts` の `matcherMaximumFiles` と `matcherMaximumSourceRatio` を読む。
4. 覆えない間に止まることを確かめる：`packages/deepsec/src/setup/coordinator.ts` で `maxAttempts` と、`services.process` を呼ぶ位置を読む。
5. 再検証の git の履歴を確かめる：`packages/processor/src/agents/shared.ts` で `git log` を組み立てる行を読む。
6. 数字の根拠を探す：`ugrep -rn -e '10–29' -e '25–60' .` で、文書以外に出てこないことを確かめる。

## 見直す条件

- 誤検知率や費用を測るコード、または公開の測定結果が入ったとき。
- 再検証が、直ったかどうかをコードで調べるようになったとき。
- 再スキャンが、分析済みのファイルの状態を戻すようになったとき。
