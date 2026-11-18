# 常時稼働の agent の部品と、Touka、Pi、Dots での形（2026-11）

Anthropic の記事は、人が見ていない間も動く agent を、5つの部品に分けて説明する。その部品が、実装と製品でどういう形になっているかを、Touka、Pi、OpenAI の Dots の文書で照合した。Touka は Cloudflare の上の実装、Pi は Touka の土台の agent harness、Dots は製品である。記事がループの出どころとして引く2024年の記事と、その記事が現在の考え方として挙げる Managed Agents の記事も読んだ。生成したコードを動かす場所の実例として、AWS の Nitro Sandbox の発表、ガイド、見本も読んだ。自分たちが採用した判断ではなく、どれも手元では動かしていない。

## TL;DR

- 記事の言う常時稼働は、利用者から見た体験の呼び名である。agent はほとんどの時間眠り、イベントで起きる。周りの部品は、イベントの受け口、タイマー、メモリ、権限、送り口の5つである。
- Touka、Dots、Managed Agents は、どれも認証情報を agent の環境の外に置く。Dots は、人が見ていない間の調査を、読み取り専用の道具に限る。
- ハーネスは、モデルが自分ではできないことの仮定を埋め込む。その仮定はモデルが良くなると古くなる、と Managed Agents の記事が書く。
- AWS の見本は、Claude Code を pod ごとの VM に入れ、Issue を読んで PR を開くまでを通す。出ていく通信は、モデルと GitHub だけに絞る。
- 測定は、費用の値と応答時間の値の2つだけである。どれも手元では動かしていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | 次の表の10件 |
| 照合日 | 2026-11-17 |
| この資料を根拠にした Issue | なし |

| 略称 | 資料 | 版と公開日 | 読んだ範囲 |
| --- | --- | --- | --- |
| 記事 | [Building always-on agents](https://claude.com/ja/resources/articles/building-always-on-agents)（William Hu、Anthropic） | 2026-11-13 | 全文 |
| 2024年の記事 | [Building effective agents](https://www.anthropic.com/research/building-effective-agents)（Anthropic） | 2024-12-19。冒頭に、道具の状況は当時から変わったという注記がある | 全文 |
| Managed Agents の記事 | [Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents)（Anthropic） | 2026-04-08 | 冒頭から Conclusion まで |
| Touka | [L4Ph/touka](https://github.com/L4Ph/touka)（MIT） | main `7d01aac`、2026-11-08。作成は 2026-10-27 | `README.md`、`AGENTS.md`、`CONTEXT.md`、`GLOSSARY.md`、`docs/mission.md`、`docs/connections.md`、`server/src/sandbox/README.md` の全文。`docs/devin-parity.md` の冒頭35行。`.r5/issues/devin-parity/06-*.json` |
| Pi | [earendil-works/pi](https://github.com/earendil-works/pi)（MIT）と [pi-chat](https://github.com/earendil-works/pi-chat) | main `0377a58`、2026-11-16 | `README.md` と `packages/coding-agent/docs/skills.md` の全文。`settings.md` の Resources の節。`packages/durable/README.md`（1,155行）のうち、冒頭、Concepts、Sleeping and Waking Up、Storage、Abort and Subagents の冒頭。pi-chat の README の冒頭150行 |
| Dots | [Introducing dots](https://openai.com/index/introducing-dots/)（OpenAI） | ページに日付が無い。報道は 2026-09-29 の発表と書く。2026-11-03 の更新の注記がある | 全文 |
| AWS の発表 | [Introducing AWS Nitro Sandboxes on Amazon EKS, ECS and EC2](https://aws.amazon.com/jp/about-aws/whats-new/2026/11/nitro-sandboxes-eks-ecs-ec2/) | ページに日付が無い。URL は 2026年11月 | 全文 |
| AWS のガイド | [EKS の Sandboxing](https://docs.aws.amazon.com/eks/latest/userguide/sandboxing.html) と [EC2 の Nitro Sandboxes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-ec2-nitro-sandboxes.html) | 版の記載なし。2026-11-17 に取得した | EKS の概要と入門の入口。EC2 の概要と、Firecracker で動かす手順のうち、ゲストの image を作る手順の冒頭まで |
| AWS の見本 | [aws-samples/sample-eks-nitro-sandbox](https://aws-samples.github.io/sample-eks-nitro-sandbox)（MIT-0） | repo の作成は 2026-10-09 | 入口のページの前提条件までと、coding agent の入門の手順6まで |
| 報道 | [Anthropic's answer to Dots and Muse is already inside Claude](https://thenewstack.io/claude-answer-to-dots-muse/)（The New Stack） | 2026-10-03 | 全文。Muse と Grok Bot の記述は、この記事と検索結果の要約による |

## 中心の主張

**常時稼働の agent は、常に動いてはいない。** 記事は、agent のループ自体はほかの agent と同じで、違いは周りの部品にあると書く。1回分の作業を run と呼ぶ。run は、イベントを受け取り、作業し、起きたことを保存して終わる。部品は、agent がいつ起きるか、run の間で何を引き継ぐか、成果をどう利用者に届けるかを決める。

記事は、実用になった理由を2つ挙げる。モデルが、利用者の意図と、確認を取るべき場面を判断できるようになったこと。長い作業の費用が下がったことである。ある長い作業のベンチマークで、Opus 5.5 は Opus 5 より、解けた課題1件あたり78%安く、典型的な作業では40%安かったと書く。この値は再現していない。

## ループの形と、ハーネスの仮定

記事は、agent のループを2024年の記事に委ねている。2024年の記事の原則と、Managed Agents の記事が足した見方を並べる。

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| workflow と agent を分ける | workflow は、モデルと道具を、あらかじめ決めたコードの経路でつなぐ。agent は、モデルが自分の進め方と道具の使い方を決める | 2024年の記事に記述がある |
| 最も単純な形から始める | 多くの用途は、検索と例を付けた1回の呼び出しで足りる。複雑さは、成果が実際に良くなるときだけ足す。agent 的な仕組みは、時間と費用を払って成果を上げる | 2024年の記事に記述がある。根拠は、数十のチームと働いた経験で、測定は無い |
| workflow の型は5つある | 連鎖、振り分け、並列、指揮と作業役、評価と改善。連鎖では、途中にコードの検査を置ける | 2024年の記事に記述がある。型ごとの効果の数字は無い |
| agent は、環境の結果を見ながら道具を使うループである | 各手順で、道具の結果やコードの実行から事実を得る。繰り返しの上限などの止める条件を置く。代償は、費用と、誤りの積み重なりである | 2024年の記事に記述がある |
| 道具の定義に、prompt と同じだけ手をかける | SWE-bench の agent では、全体の prompt より道具の調整に時間を使った。相対パスで間違えたので、絶対パスを必須にすると、間違えなくなった | 2024年の記事に記述がある。回数は示されない |
| ハーネスは、モデルが自分ではできないことの仮定を埋め込む | 仮定は、モデルが良くなると古くなるので、頻繁に問い直す。Sonnet 4.5 は、context の上限が近づくと作業を早く切り上げたので、ハーネスに context のリセットを足した。同じハーネスを Opus 4.5 で使うと、その振る舞いは消えていて、リセットは不要になっていた | Managed Agents の記事に記述がある。例は1件である |
| 記録、ハーネス、サンドボックスを分ける | 記録は、起きたことの追記だけの log である。ハーネスは、モデルを呼び、道具の呼び出しを振り分けるループである。サンドボックスは、コードを動かす場所である。記録を外に出すと、ハーネスは落ちても、新しいものが記録を読んで続きから再開できる | Managed Agents の記事に記述がある。最初の token までの時間が、中央値で約60%、95パーセンタイルで90%超下がったと書く。再現していない |
| 認証情報は、構造で届かなくする | 生成したコードが動くサンドボックスから、トークンに届かないようにする。権限を狭くするだけの対策は、狭いトークンでは何もできないという仮定に頼るので、当てにしない。Git のトークンは初期化のときに remote へ組み込み、MCP の呼び出しは、保管庫から認証情報を取る proxy を通す | Managed Agents の記事に記述がある |

2024年の記事の原則は、照合した範囲では、新しい記事に否定されていない。変わったのは重みづけである。2024年の記事は、agent を使わない選択を先に置く。記事と Managed Agents の記事は、長く動く agent を前提にする。

## 5つの部品と、実装での形

| 部品 | 記事の勧め | Touka | Dots |
| --- | --- | --- | --- |
| イベントの受け口 | 出どころが agent を直接呼ばず、永続する一覧に書いてから worker を起こす。応答は速く返す。重複は出どころの ID で落とす。同時に動く worker は1つにし、溜まったイベントはまとめて1回の run にする | Trigger が、署名の検証、許可した人の判定、会話への対応付けを持つ。永続に受理してから ACK を返し、同じ操作の ID を2回入れない。会話ごとに Durable Object が1つ付く | ChatGPT、Slack、Teams から話しかける。受け口の作りは書かれていない |
| タイマー | 追いかけ（agent が自分に置く期限）と、予定（決まった間隔）の2種類を、受け口とは別の保存先に置く。スケジューラが期限のものを受け口に入れる | 持たない。文書は「一般の自動化の製品は無い」と書き、該当のチケットは6つの条件がすべて未完である | 待機中に、助けになることを自分で探すと書く。予定の作りは書かれていない |
| メモリ | 調べ直せないものだけを持つ。ほかの仕組みにあるものは、場所だけを持つ。持つのは、利用者の指示、利用者についてのメモ、送った内容、作業の進みである | 持たない。文書は、会話の履歴は共有のメモリではないと書く | フィードバックから学び、好みと良し悪しの基準を覚えると書く |
| 権限 | 認証情報は、agent の環境の外の proxy が、出ていく要求に付ける。危ない操作は分類器が審査する。承認が要る呼び出しは保留し、run を終え、利用者の答えをイベントとして受けて再開する | container は認証情報を持たない。外への通信は1つの出口を通り、grant のある repo への要求にだけ GitHub のトークンが付く。GitHub の権限は read、write、reply、ci、workflow の5段階である。merge と deploy の操作は公開していない。保留して承認を待つ仕組みは、読んだ範囲に無い | 専用のクラウドの計算機で動き、利用者の計算機は、つなぐと決めるまで別である。待機中の調査は読み取り専用の道具に限り、送信も変更もできない。行動の規則は、許可、承認、禁止の3種類で、行動は自動で審査される。パスワードの変更などは常に人が行う |
| 送り口 | 送る内容を別の表に置き、配る役が届ける。多くの場合は、直接送る道具1つで足りる。送る量は eval で調整し、少なく始める | 返信の記録を会話の側に持ち、キューが再送を起こす。外部への書き込みは、ちょうど1回を保証しないと書く | 進み、質問、判断が要ることを、Dot の側から伝えると書く |

Touka の設計には、記事に無い点が3つある。

- **呼び出せることと、書き込めることを分ける。** 入口（Trigger）と、外部サービスの読み書き（Connector）を別の部品にする。Trigger から始まった会話でも、書き込みには、サーバーが検証した grant が要る。
- **核は、どこから呼ばれたかを知らない。** 受け取るのは、会話の ID、操作の ID、入力だけである。呼び出し元の名前を、モデルへの prompt に入れない。
- **完了の定義に、本番での確認を求める。** 連携を登録した、道具を呼べた、単体テストが通った、のどれも、機能が揃った証拠にしないと書く。

## Pi の2つの使い方

Touka は、実行のループを Pi Durable に、モデルの呼び出しを pi-ai に任せている。Pi には CLI もあり、できることが違う。

| | Pi の CLI（`pi-coding-agent`） | Pi Durable |
| --- | --- | --- |
| 形 | 対話、print と JSON の出力、RPC、SDK を持つ CLI | ライブラリ。会話、モデルの応答、道具の呼び出しを、見せる前に保存する。途中でプロセスが落ちると、保存先を開き直して続きから再開する |
| 動く場所 | Node.js 22.19 以上 | 保存先は5種類ある。メモリ、ブラウザの IndexedDB、Node の SQLite、Node の JSONL、Cloudflare の Durable Object である |
| skill | Agent Skills の仕様を実装する。`~/.agents/skills/` と `.agents/skills/` を既定で読み、設定の `skills` でほかの場所を足せる | README に skill の記述は無い |
| subagent | 組み込まれていない。README は、subagent と plan mode を省いたと書く | 道具として自分で定義する例が載っている |
| 予定での起動 | 記述は無い | task を眠らせ、時刻か信号で起こせる。Node では timer が起こし、Durable Object ではホストが起こす |
| 成熟 | 公開の CLI である | README の冒頭が Experimental と書く。API は予告なく変わる |

## 生成したコードを動かす場所：Nitro Sandbox と、coding agent の見本

AWS の Nitro Sandbox は、pod や task を1つずつ別の軽量 VM に入れる機能である。Nitro の hypervisor が、Sandbox の間のメモリと CPU を分ける。発表は、agent が生成したコードを、1台のインスタンスの中で分けて動かす用途を挙げる。

動かし方は3通りある。

| 方法 | やること | 条件 |
| --- | --- | --- |
| EKS | pod の定義に `runtimeClassName` を足す。値は `isolated-sandbox` か `confidential-sandbox` である | Kubernetes 1.36 以降。対応するインスタンスタイプ |
| ECS | task ごとに Sandbox で動かす | ガイドは読んでいない |
| EC2 で直接 | 親のインスタンスで Firecracker を動かし、ゲストの image を自分で作る | Nitro Enclaves を有効にして起動する。Amazon Linux 2023 の `kernel6.18`（`6.18.51-120.162` 以降）。Firecracker 1.18 以降。起動時に確保するメモリのプール |

モードは2つある。isolated は、親を信頼し、中の処理を信頼しない。壊れた container から、ノードとほかの pod を守る。confidential は、親を信頼しない。親は Sandbox のメモリを読めず、構成証明を通った Sandbox にだけ、KMS が鍵を渡す。

見本の coding agent は、Claude Code が GitHub の Issue を読み、直し、PR を開くまでを通す。

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| agent の全体を、1つの Sandbox に入れる | coding agent は、任意のコードを実行し、インターネットと話す。prompt injection が、この組み合わせを情報の持ち出しに変える。届く先を、pod の定義の隣にすべて書く | 見本に記述がある |
| モデルの認証に、長く使うキーを置かない | モデルは Amazon Bedrock で、EKS Pod Identity で認証する。役割は `bedrock:InvokeModel*` だけを許す | 見本に記述がある |
| repo の認証は、1つの repo に絞った短いトークンにする | fine-grained のトークンを、期限を短くし、1つの repo だけに絞る。権限は、Contents と Pull requests の読み書きと、Issues の読み取りである。Secret としてファイルで mount し、コマンドラインに置かない | 見本に記述がある。トークンは Sandbox の中にある。Touka と Managed Agents が、認証情報を外に置くのとは違う |
| 通信は、許可した先だけにする | 入ってくる通信はすべて拒否する。出ていく通信は、DNS、Pod Identity の認証情報のアドレス、Bedrock、`github.com`、`api.github.com` の HTTPS だけである | 見本に記述がある。名前が解決するアドレスを許すので、同じアドレスから配られる別の名前にも届くと、見本が断る |
| 起動の前に、制限が働いていると確かめる | entrypoint は、許可した先が応答し、許可していない先に届かないことを確かめる。確かめられなければ、約1分待ってから、agent の起動を拒む | 見本に記述がある。entrypoint の本文は読んでいない |
| VM の境界の内側も固める | root でない利用者で動かし、権限をすべて落とし、root のファイルシステムを読み取り専用にする。書けるのは、ホーム、作業用のディレクトリ、`/tmp` の3つで、pod が終わると捨てる | 見本に記述がある |
| 通信を完全に外す指定がある | EKS では、注釈1つで、ネットワークの装置を持たない Sandbox になる。DNS も、インスタンスのメタデータも、Pod Identity の認証情報も無い | EKS のガイドに記述がある。内部の制御用の経路は残るので、完全な隔離ではないと書く |
| Sandbox から、親の認証情報に届きうる | 親が Sandbox の通信を中継すると、Sandbox は親のメタデータのサービスから、IAM の役割の認証情報を読める。IMDSv2 と hop limit では防げない。親で、`169.254.169.254` への通信を拒否する | EC2 のガイドの考慮点に記述がある |

見本の pod は、2 vCPU と 4GiB で動く。実行のコマンドは `claude -p` で、使える道具を `Bash,Edit,Read,Write` に絞る。Issue に印が付いたら pod を起こす部分は、見本に含まれない。利用者が自分で pod を起こす。

## 製品の3つ

dots、bots、muse と並べて呼ばれるのは、2026年の8月から9月に出た3つの製品である。

| 名前 | 提供元 | 形 | 出どころ |
| --- | --- | --- | --- |
| Dots | OpenAI | 専用のクラウドの計算機とブラウザを持ち、4,000超のアプリにつながる。Pro、Business Premium、Enterprise から提供する。組織の中で決まった責任を持つ specialist dots は、preview である | Dots の発表 |
| Grok Bot | xAI | 8月に出た。全体で1つのクラウドの計算機とログインを共有する。xAI の文書は、Grok Bot をセキュリティの境界として扱わないよう書く | 報道だけ |
| Muse | Meta | 9月に出た個人向けの agent で、WhatsApp の中で使える | 報道だけ |

報道は、Anthropic の側を、名前の付いた製品ではなく機能として挙げる。予定で動く Cowork、Slack に独自の身元と監査の記録で参加する Claude Tag、長く動く agent を預かる Managed Agents である。これらの文書は読んでいない。

## 限界

- **測定は2つだけである。** 記事の費用の値と、Managed Agents の記事の応答時間の値のほかに、数値の根拠は無い。どちらも Anthropic の自己申告である。Touka と Dots は、効果の件数を示さない。
- **Nitro Sandbox には、運用の制限がある。** 親のインスタンスを止めると、Sandbox も止まる。スナップショット、一時停止、ライブマイグレーションは無い。動作中に vCPU とメモリを変えられない。上限は、親の vCPU 1つにつき5個、親1台で896個である。flex のインスタンスと、GPU の受け渡しは使えない。
- **Nitro Sandbox の性能は、数字で示されない。** 発表は「EC2 に近い性能」と書く。見本は、起動済みの pool から1秒未満で引き当てられると書く。測り方は読んでいない。
- **見本は、モデルを Bedrock 経由で使う。** 料金は Bedrock の従量になる。EKS のクラスタ、container の registry、GitHub の repo が要る。
- **2024年の記事は、例が古い。** 挙がる framework と、Haiku 4.5、Sonnet 4.5 は、当時のものである。記録を外に持つこと、落ちた後の再開、予定での起動は扱わない。
- **Touka は新しい。** 作成から3週間の個人のプロジェクトで、スターは43である。自動テストは、モデルと外部サービスを偽物で試す。container の起動や TLS の差し替えは、配置した Worker でだけ確かめると書く。
- **Touka は、機能が揃っていないと自分で書く。** 目標は Devin と機能で並ぶことで、文書は、まだ並んでいないと明記する。
- **Pi は、権限を制限しない。** README は、ファイル、プロセス、通信、認証情報のどれも制限する仕組みを持たず、起動した利用者の権限で動くと書く。対策として、micro-VM、Docker、サンドボックスの3つの形を挙げる。
- **pi-chat は Slack に対応していない。** Discord と Telegram だけで、最後の push は 2026-06-05 である。
- **Dots の発表は、製品の紹介である。** 例は OpenAI の社内と初期の利用者のもので、件数は無い。発表自身が、Dot は間違えうるので、重要な作業は必ず確かめるよう書く。
- **Muse と Grok Bot は、一次資料を読んでいない。** 発表日、利用者数、機能は、報道と検索結果の要約による。

## 手元の資料との関係（未採用）

この節は、照合者が手元の設定と資料を読んで突き合わせた結果である。何も作っておらず、どの設定も変えていない。

きっかけは、公開の情報からプロダクトへのフィードバックを集め、それを元に draft PR を作る agent を自作できるか、という問いである。3つの製品ほどの自律は求めない。

| 資料の考え方 | 手元の資料 |
| --- | --- |
| 常時稼働は、常駐を意味しない | [thkt/feedback-loop#187](https://github.com/thkt/feedback-loop/issues/187) の Non-goal は「常駐する監視プロセスを作ること」である。予定で起きて終わる形は、この Non-goal に当たらない、と照合者は読んだ |
| 待機中の調査は、読み取り専用にする | 集める段と、実装する段を分ける案にあたる。公開の投稿は誰でも書けるので、投稿を材料としてだけ扱い、実装に進むのは人が選んだ Issue だけにする |
| 調べ直せないものだけを持つ | 見た投稿は、中身ではなく URL を記録すれば足りる。#187 の、回数に届かない訂正を候補として記録する部分も、同じ形にできる |
| 入口と権限を分ける | [rethinking-claude-code-best-practices-hallie.md](rethinking-claude-code-best-practices-hallie.md) の講演は、保証が要るものを deny rule、hook、サンドボックスで強制すると述べる。Touka は、同じ考えを grant と出口で実装している |
| workflow と agent を分ける | `~/.claude` の `build` と `audit` は、スクリプトが経路を決める workflow である。`implement` の skill は、モデルが進め方を決める側にあたる。公開の情報を集めて候補を出す段は、集める、分類する、一覧にする、という決まった手順なので、workflow で足りる。モデルに経路を任せるのは、実装の段だけになる |
| ハーネスの仮定は、モデルが良くなると古くなる | `~/.claude` の `ablate` の skill は、ハーネスの要素を1つずつ外して走らせ直し、結果が動くかを判定する。同じ問いへの手段である。手元の workflow のどの段が不要になっているかは、調べていない |
| 記録、ハーネス、サンドボックスを分ける | Touka は、記録を Durable Object に、実行を container に、認証情報を出口に分けている。Pi Durable は、見せる前に保存し、落ちたら続きから再開する。どちらも、Managed Agents の記事の分け方と対応する |
| 各段が止められ、証拠を渡す | [pstack-overnight-and-self-starting-loops.md](pstack-overnight-and-self-starting-loops.md) の benny は、Slack の報告を選別する自動化と、再現して draft PR を出す自動化の2つである。目標の形と同じで、入力が Slack か公開の情報かが違う |
| 起動の前に、制限が働いていると確かめる | [astryx-agent-evals.md](astryx-agent-evals.md) の eval は、採点の前に、落ちるはずの入力が落ちることを確かめ、違えば実行を止める。同じ考えを、通信の制限に当てている |
| 見本の指示は、最小の変更を求める | 見本の prompt は、テストが通る最小の変更で直すよう書く。[pstack-fix-root-causes.md](pstack-fix-root-causes.md) の投稿は、最小の変更を求めると応急処置が広がると述べる。見本を使うなら、見直す対象になる |
| 生成したコードを、VM の境界で分ける | [agent-in-apple-container.md](../knowledge/agent-in-apple-container.md) は、手元の Mac で、container ごとに VM を分けた。置き場所が違い、こちらは EC2 と EKS の費用がかかる |
| Pi の CLI は `~/.agents/skills/` を読む | 手元の `~/.agents/skills` には `implement` と `scoping` がある（2026-11-17 に一覧を確かめた）。中身が Pi で最後まで動くかは、確かめていない。`~/.claude` の `build` は、Claude Code の Workflow と reviewer の agent を使うので、Pi には移せない |

動かす場所の候補は4つある。どれも試していない。

| 案 | 作る量 | 確かめていないこと |
| --- | --- | --- |
| Claude Code の予定の実行に載せる | 最も少ない見込み | クラウドの実行環境から、`~/.claude` の skill と、`xr` や `scout` を使えるか |
| Pi の CLI を、container の中で cron から呼ぶ | 中程度 | `implement` の skill が Pi で最後まで動くか |
| Pi Durable の上に作る（Touka の形） | 多い | API が変わる頻度。質問の往復と、落ちても続きから再開する性質が要るか |
| AWS の見本を土台にする | 多い。EKS のクラスタ、Bedrock、registry、GitHub のトークンを用意する | 費用。手元の skill を image に入れる方法。Issue に印が付いたら pod を起こす部分の作り |

どの案より先に、対象を1つ決めて手で1回集め、件数、問題の報告の割合、Issue にできた割合を数える必要がある。pstack の4つの条件の1つ目が、手で一度やってあることだからである。この計測は行っていない。

## 照合の結果

- **照合した主張：** 記事、2024年の記事、Managed Agents の記事、Touka、Pi、Dots の各行は、上の表の「読んだ範囲」に記述がある。
- **動かしていないもの：** Touka、Pi、Dots。
- **読んでいないもの：** ECS のガイド。EKS の、confidential の設定、ネットワークの隔離、資源の制限、考慮点のページ。EC2 の構成証明のページと、Firecracker の手順の後半。見本の `Dockerfile`、`entrypoint.sh`、ほかの入門。Touka のコード、`docs/operations.md`、`docs/agent-parity-inventory.md`、`server/src/README.md`。Pi のコード、`packages/durable/README.md` の残りの節、`docs/spec.md`、`extensions.md`、`containerization.md`。Dots の安全性の記事と system card。Cowork、Claude Tag、Managed Agents の文書。Managed Agents の記事が引く、長く動く agent のハーネスの記事2件と、context の扱いの記事。手元の `build`、`audit`、`implement`、`ablate` は、説明の行を読んだだけで、本文は読み直していない。
- **二次情報だけのもの：** Muse と Grok Bot。Dots の発表日。Dots の運用の解説（持ち主だけが指示できる、測るのは PR の数ではなくレビューの手間と採用された結果である）は、検索結果の要約で読んだだけなので、表に入れていない。

## 照合の手順

1. 記事を取得する：`scout fetch https://claude.com/ja/resources/articles/building-always-on-agents`。同じ方法で、`https://www.anthropic.com/research/building-effective-agents` と `https://www.anthropic.com/engineering/managed-agents` を取得する。
2. Touka の版と一覧を確かめる：`gh api 'repos/L4Ph/touka/commits?per_page=1' --jq '.[0].sha'` と `scout repo-tree L4Ph/touka`
3. Touka の文書を読む：`scout repo-read L4Ph/touka <path>`。path は上の表のとおりである。
4. Pi の文書を読む：`scout repo-read earendil-works/pi README.md`、`packages/coding-agent/docs/skills.md`、`packages/durable/README.md`
5. AWS の資料を取得する：`scout fetch` で、発表、EKS のガイド、EC2 のガイド、`nitro-sandboxes-run-firecracker.html`、見本の入口と `getting-started/coding-agent/index.html` を読む。
6. Dots の発表を取得する：`scout fetch https://openai.com/index/introducing-dots/`
7. 手元の skill の置き場所を確かめる：`ls ~/.agents/skills`

## 見直す条件

- Touka がタイマーかメモリを実装したとき。照合した版は `7d01aac` である。
- Pi Durable の README から Experimental の表記が外れたとき。照合した版は `0377a58` である。
- Muse か Grok Bot の公式の発表を読んだとき。
- 手で1回集めて、件数を数えたとき。結果は `knowledge/` に書く。
- 動かす場所の案のどれかを試したとき。
