# MCP の Events 拡張：上流の出来事を購読し、誰も頼んでいないときに agent を起こす仕組み（2026-11）

MCP の公式サイトは、Events 拡張（`io.modelcontextprotocol/events`）の概要を公開している。概要、設計の提案（SEP-3415）、仕様の repo、対応クライアントの表を読み、同じ日に手元で行った「出来事で agent を起こす」試しと比べた。Events そのものは、手元で動かしていない。自分たちが採用した判断ではない。

## TL;DR

- 中心は、MCP に「頼まれていないときに届く出来事」の口を足すことである。client は、出来事の種類と引数を指定して購読する。出来事が起きると、agent の実行基盤がそれを受け取り、新しいターンを始めるかを決める。
- 配送の形は 3 つある（poll、push、webhook）。どれも必須ではなく、出来事の種類ごとに server が選ぶ。途切れた後は、カーソルで続きから受け取る。取りこぼしがあれば、server が `truncated: true` で知らせる。
- 安全の決まりは 2 つに集まる。出来事の中身は、信頼できないデータとして扱う。出来事を受け取ったことは、agent が動く許可にならない。
- 対応クライアントの表で、Events の欄が埋まっていたのは 18 のうち 1 つ（ChatGPT、Partial）だけだった。Claude Code の欄は空である。照合日は 2026-11-23。
- 手元の試しは、Events ではなく、GitHub の webhook で agent を起こした。配送、署名、信頼できないデータの扱いは同じ形になった。カーソル、取りこぼしの通知、購読できる出来事の一覧は、手元の形には無い。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Events - Model Context Protocol](https://modelcontextprotocol.io/extensions/events/overview) |
| 著者・発行元 | Model Context Protocol の Tasks and Events Working Group |
| 公開日 | 概要の頁に日付は無い。SEP-3415 の作成日は 2026-10-05 |
| 照合日 | 2026-11-23 |
| 照合した範囲 | 概要の頁の全文、[SEP-3415](https://modelcontextprotocol.io/seps/3415-events-extension) の冒頭から `events/list` の節まで、[ext-events の repo](https://github.com/modelcontextprotocol/ext-events) の README と Issue・PR の題名、[対応クライアントの表](https://modelcontextprotocol.io/extensions/client-matrix)。仕様の本文（`specification/beta/events/events.mdx`）は、認可、TTL、送り先の確認、取りこぼしの語で引いた行だけを読んだ |
| この資料を根拠にした変更 | ハーネスの決まりへの変更は、なし。読んだ日に、Issue が開かれたら agent を起こす webhook を、手元の 4 つの repo に試しとして置いた |

## 中心の主張

**MCP は、頼まれて答える形だけでは足りない。** 新しいメール、repo への push、障害の発生のように、誰も頼んでいないときに起きた事柄へ agent が応じるには、購読と配送の決まりが要る。概要は、その決まりを Events として定める。

## 考え方と根拠

| 考え方 | 資料の記述 | 照合の結果 |
| --- | --- | --- |
| 今ある通知では足りない | `notifications/resources/updated` と `list_changed` は、引数も中身も持たない。開いた接続が要る。カーソルが無く、切れている間の出来事が失われ、失ったことも分からない | 概要と SEP-3415 が、同じ 3 点を挙げる |
| 購読できる出来事を、server が一覧で示す | client は `events/list` を呼ぶ。出来事の種類ごとに、名前、購読の引数の形（`inputSchema`）、届く中身の形（`payloadSchema`）、使える配送の形が返る | SEP-3415 の例で確かめた。メールの受信と障害の発生の 2 つが例に出る |
| 配送の形は 3 つで、必須のものは無い | poll は、client が間隔を置いて取りに行く。push は、client が長い要求を開いたままにする。webhook は、client が `https` の送り先を登録し、server が署名つきで POST する | 概要の表と SEP-3415 が一致する。SDK は、送り先があれば webhook、次に push、次に poll を選ぶ、と概要が書く |
| 3 つあるのは、置かれる環境が重ならないからである | サーバーレスの server は、購読者ごとの接続を持てない。NAT の内側の client は、送り先を持てない。費用を抑えたい client は、poll を続けられない | 概要と SEP-3415 の記述だけである。実装での測定は、示されていない |
| 途切れた後は、カーソルで続きから受け取る | カーソルは、server が決める位置である。client は、配送のたびに保存する。server がその位置から出せないときは、`truncated: true` を返す | 概要の記述。仕様の本文は、webhook では `gap` という制御用の配送で同じことを知らせる、と書く |
| 届け方の保証は、上流しだいである | 上流が出来事を保存していて、client がカーソルから再開すれば、少なくとも 1 回は届く。履歴を持たない種類では `cursor: null` になり、切れている間の出来事は戻らない | 概要の記述だけである |
| 古い出来事を、まとめて受け取らない | client は `maxAgeMs` を渡せる。1 週間止まっていた client が、直近 5 分ぶんだけを受け取る、という使い方である | 概要の記述。仕様の例は 300000 ミリ秒を使う |
| 同じ出来事は、`eventId` で見分ける | `eventId` は、上流の出来事ごとに変わらない。順序は、poll と push では保たれる。webhook では保証されず、`timestamp` で並べる | 概要の記述だけである |
| 購読の一覧は、client が持つ | server に、購読を一覧する呼び出しは無い。client は、server の側の状態をいつでも作り直せる | 概要と SEP-3415 が一致する |
| 出来事の中身は、信頼できないデータである | 外部の仕組みから来るので、prompt injection を含みうる。tool の結果と同じに扱う | 概要と SEP-3415 が一致する |
| 中身は小さくする | server は、送り主、題名、ID のように、見分けて振り分けるのに足りる分だけを送る。全文は、client が tool で取りに行く | 概要の記述だけである |
| 受け取りは、動く許可にならない | server は、購読のときと配送のときに権限を確かめ、取り消されたら購読を終える。agent がその後に行う tool の呼び出しは、いつもの認可を通る | 概要の記述。仕様の本文は、webhook の購読には認証済みの主体が要る、と書く |
| webhook は、送り先を守る | 署名は Standard Webhooks の HMAC である。送り先は `https` に限る。server は、配送を始める前に送り先を確かめ、宛先の IP を検査する | 仕様の本文で確かめた。送り先の確認は、使い捨ての値を送り返させる方法など、複数の方法から 1 つを満たせばよい。IP の検査は、購読のときだけでなく配送のたびに行う |
| 自分で始めた長い作業の結果待ちには、使わない | その用途は Tasks 拡張である | 概要の記述だけである |

## 普及の状態

| 項目 | 照合の結果 |
| --- | --- |
| 段階 | SEP-3415 は Final で、拡張は Beta である。仕様の対象は、プロトコルの版 `2026-07-28` 以降である |
| 対応クライアント | 表の 18 のうち、Events の欄が埋まっていたのは ChatGPT（Partial）だけだった。Claude（web）、Claude Desktop、Claude Code、Cursor の欄は空である |
| repo の表記 | 概要は ext-events を公式の拡張の repo と書く。repo の README の冒頭は、実験の置き場だと書いたままである。同じ README の表は、Events を Beta と書く |
| 開いている議論 | Issue と PR の題名から読めた範囲で、3 つある。ChatGPT は購読しても知らせるだけで、agent がどう応じるかを運ぶものが無い、という報告。送り先の確認が Standard Webhooks の拡張と違う、という指摘。購読を、読むだけで壊さないものに限る、という提案。本文は読んでいない |

## 手元で確かめたこと

Events を実装した client と server は、手元に無い。代わりに、同じ日に、GitHub の webhook で手元の agent を起こす形を試した。使ったのは、手元の agent の実行環境が持つ、webhook で始まる定期実行である。

### 試した形

1. 実行環境に、webhook で始まる実行を 1 つ登録した。受け口の URL が発行される。
2. 絞り込みを 4 つ置いた。出来事が `issues` である。`action` が `opened` である。repo が、決めた 4 つのどれかである。送り主が自分である。
3. 署名の検査を置いた。GitHub の `x-hub-signature-256`（HMAC-SHA256）を使う。
4. 4 つの repo に、GitHub の webhook を足した。送る出来事は `issues` だけにした。
5. agent への指示に、届いた値のうち repo の名前と Issue の番号だけを写した。Issue の題名と本文は、agent が `gh` で取り直す。

### 結果

| 確かめたこと | 結果 |
| --- | --- |
| 署名の無い配送 | 受け口が 401 で拒んだ |
| 署名のある配送 | 202 で受け取られ、agent が起きた |
| GitHub からの確認の配送（ping） | 4 つとも 202 で届いた。絞り込みに合わないので、agent は起きなかった |
| 届いた値を指示に写さなかった 1 回目 | agent は、どの Issue かを知らず、「対象外の配送」と報告して止まった。指示に、値を差し込む書き方を足して直した |
| 差し込んだ値の届き方 | 信頼できないデータだという注記と、専用のタグに包まれて、agent に届いた |
| 模擬の配送で、実在の Issue を 1 件読ませた | agent は本文を取り直して読み、根拠の行について 2 点を指摘した。Issue への書き込みは、行わなかった |
| 本物の Issue が開かれての実行 | 2026-11-23 の時点で、0 回である |

### Events の仕組みとの対応

| Events の仕組み | 手元の試し |
| --- | --- |
| 購読できる出来事の一覧（`events/list`） | 無い。GitHub の webhook の文書を読み、絞り込みを手で書いた |
| 購読の引数で、server が出来事を絞る | 絞るのは受け側である。GitHub は `issues` の出来事を全部送り、受け口が 4 つの条件で落とす |
| webhook の配送と、HMAC の署名 | 同じ形である。署名の形式は、Standard Webhooks ではなく GitHub のものである |
| 配送を始める前の、送り先の確認 | 無い。GitHub の ping は、届くことを確かめるだけである |
| カーソルと `truncated: true` | 無い。手元のマシンが止まっている間の配送は、60 分を超えたら実行しない設定にした。取りこぼしたことを知る手段は、無い |
| `eventId` で重複を見分ける | 使っていない |
| 中身は小さくし、全文は tool で取りに行く | 同じ形にした。指示に写すのは、repo の名前と番号だけである。agent は、2 つの値の形を確かめてから、本文を取り直す |
| 受け取りは、動く許可にならない | 指示に「読むだけにする。報告は 1 つのスレッドだけに書く」と書いた。指示の文で縛っただけで、agent の権限を絞ってはいない |
| TTL と更新 | 無い。GitHub の webhook は、消すまで残る |

## 限界

- **Events を動かしていない。** 手元の試しは GitHub の webhook で、Events の client と server ではない。対応の表は、仕組みの形を比べただけである。
- **仕様の本文を、通して読んでいない。** 語で引いた行だけを読んだ。エラーの扱い、購読の同一性、SDK 向けの手引きは読んでいない。
- **対応クライアントの表は、申告に基づく。** 表は、各 client の側が PR で更新する。欄が空でも、対応していないとは限らない。ChatGPT の Partial の中身は、リンク先を読んでいない。
- **手元の試しは、本物の出来事で 1 回も起きていない。** 確かめたのは、手元から送った模擬の配送と、GitHub の ping である。
- **「読むだけ」は、指示の文である。** agent が指示に従わなかったときに、書き込みを止める仕組みは置いていない。

## 照合の結果

- **照合した主張：** 配送の 3 つの形、カーソルと `truncated`、購読の一覧を client が持つこと、中身を信頼できないデータとして扱うことは、概要と SEP-3415 で一致した。送り先の確認と IP の検査は、仕様の本文で確かめた。
- **読んでいないもの：** 仕様の本文の全体。SEP-3415 の設計の理由の節。ChatGPT の対応の文書。Issue と PR の本文。
- **範囲外：** Tasks 拡張。Standard Webhooks の仕様。SDK の実装。

## 照合の手順

1. 概要を取得する：`scout fetch https://modelcontextprotocol.io/extensions/events/overview`。
2. SEP を取得する：`scout fetch https://modelcontextprotocol.io/seps/3415-events-extension`。
3. 対応クライアントの表を取得し、Events の列を数える：`scout fetch https://modelcontextprotocol.io/extensions/client-matrix`。
4. repo の README と、Issue・PR の題名を読む：`scout repo-overview modelcontextprotocol/ext-events`。
5. 仕様の本文を、語で引く：`scout repo-read modelcontextprotocol/ext-events specification/beta/events/events.mdx` の出力を、`authoriz`、`TTL`、`verification`、`truncated`、`maxAgeMs` で絞る。
6. 手元の試しは、「試した形」の 5 つの手順で行う。署名の無い配送と署名のある配送は、受け口の URL へ、同じ本文を HMAC の有無だけを変えて POST する。

## 見直す条件

- Claude Code か、手元で使う agent の実行環境が、Events に対応したとき。GitHub の webhook の形を、Events の購読に置き換えるかを決める。
- 手元の webhook が、本物の Issue で 3 回起きたとき。指摘が役に立ったかと、取りこぼしがあったかを見て、続けるかを `knowledge/` に書く。
- agent が、指示に反して書き込みを行ったとき。指示の文ではなく、権限で止める形に変える。
- 拡張が Beta から次の段階に進んだとき。仕様の本文を通して読み直す。
