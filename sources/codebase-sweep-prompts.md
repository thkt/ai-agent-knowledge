# X で共有されたコードベースの総点検のプロンプトと tips（2026-11）

2026-10 から 11 に X で共有された、coding agent 向けのプロンプトと AGENTS.md の tips を集めた。14件の投稿の要点をまとめ、引用元がある tip は原典と照合した。投稿の原文は載せていない。表のリンクから読める。どれも自分たちが採用した判断ではなく、効果も測っていない。これを元にした skill の草案は、照合者が別に作った（「手元の資料との関係」）。

| 項目 | 内容 |
| --- | --- |
| 出典 | X の投稿14件（下の表）と、2件が引用した [anomalyco/opencode の AGENTS.md](https://github.com/anomalyco/opencode/blob/ba77ff90d34efc0bc792169698c760dd892ad1e2/AGENTS.md) |
| 公開日 | 投稿は 2026-10-03 から 2026-11-11 |
| 照合した版 | OpenCode の AGENTS.md は `dev` の `ba77ff9`（2026-11-13） |
| 照合日 | 2026-11-13 |
| 照合した範囲 | 投稿の本文と投稿者自身の返信（`xr tweet <url> --thread`）。OpenCode の AGENTS.md の Style Guide の節 |
| この資料を根拠にした Issue | なし |

| 投稿 | 投稿日 | いいね | 系統 |
| --- | --- | --- | --- |
| [@alexalbert__](https://x.com/alexalbert__/status/2119884951550386671) | 2026-11-09 | 10051 | 総点検 |
| [@_dylanga](https://x.com/_dylanga/status/2117099613022732412) | 2026-11-02 | 1874 | 総点検 |
| [@steveruizok](https://x.com/steveruizok/status/2116552016218710389) | 2026-10-31 | 8546 | 総点検 |
| [@veliorix](https://x.com/veliorix/status/2119152590625861873) | 2026-11-07 | 512 | 総点検 |
| [@_khrus](https://x.com/_khrus/status/2118971805885239777) | 2026-11-07 | 2112 | 総点検 |
| [@Rakel076](https://x.com/Rakel076/status/2114220474381025739) | 2026-10-25 | 1500 | 過剰な防御 |
| [@Stargridcompass](https://x.com/Stargridcompass/status/2114739186477441482) | 2026-10-26 | 1424 | 過剰な防御 |
| [@Voxyz_ai](https://x.com/Voxyz_ai/status/2115518965443076573) | 2026-10-28 | 2071 | 過剰な防御 |
| [@Av1dlive](https://x.com/Av1dlive/status/2114420226725204453) | 2026-10-25 | 699 | 過剰な防御 |
| [@bkrem_](https://x.com/bkrem_/status/2119446181403357393) | 2026-11-08 | 305 | 過剰な防御（逆向き） |
| [@dmokafa](https://x.com/dmokafa/status/2114977298122244116) | 2026-10-27 | 538 | テスト |
| [@sesere115](https://x.com/sesere115/status/2106265918145978753) | 2026-10-03 | 4075 | 進め方 |
| [@jpschroeder](https://x.com/jpschroeder/status/2120551435788222485) | 2026-11-11 | 471 | 総点検の外 |
| [@sub_827](https://x.com/sub_827/status/2119493058526085471) | 2026-11-08 | 476 | 総点検の外 |

## 中心の主張

**agent には、原則の名前か、探す対象の名前を1つ渡して、コードベース全体を点検させる。** 共有されたプロンプトの多くは1行で、何を探すかだけを名指しする。@veliorix は「モデルはもう用語を知っている。原則の名前を言えばいい」と書く。

もう1つの共通点は、agent が足しすぎるものを止める指示である。足しすぎるものの例は、起きそうにないケースへの防御、try/catch、後方互換のコードである。

## 系統ごとの要点

### 総点検のプロンプト

| 投稿 | 投稿の内容 | 要点と照合の結果 |
| --- | --- | --- |
| @alexalbert__ | 新しいモデルが出るたびに、全プロジェクトで同じ1つのプロンプトを回すと書く。プロンプトは、プロジェクト全体を読ませ、壊れているもの、不整合、非効率、不要になったものを探させる。確実なものは直させ、残りはレビュー用の一覧にさせる | 新しいモデルが出るたびに、全プロジェクトで回す。「確実なものは直し、残りは一覧にする」という出力の決め方は、どの観点にも使える |
| @_dylanga | プロンプトは `do a deep scan of all hot paths and fix any unnecessary sequential async functions calls` である。差分がレビューしやすく、ほぼ毎回成果が出ると書く | 差分が読みやすいことを利点に挙げる。投稿の画像は読んでいない |
| @steveruizok | プロンプトは `find all of the implicit state machines in my app` である | 返信で目的を「明示的にするため。矛盾が見つかりやすい（Yeah so that I can make them explicit, tends to find conflicts）」と書く |
| @veliorix | `deepen modules and simplify interfaces` は、John Ousterhout の『A Philosophy of Software Design』の4章にあたると書く。同じ本から、agent に使える指示をさらに3つ挙げる。`pull complexity downwards` は、関数が面倒を引き受け、呼び出し側を単純に保つ。`define errors out of existence` は、try/catch を足す代わりに、エッジケースが起きない API に変える。`design it twice` は、書く前に、根本から違う2案を描かせる。モデルは用語を知っているので、原則の名前を言えばよいと結ぶ | 『A Philosophy of Software Design』の章との対応は、本を照合していない。"design it twice" は書く前の設計の段階の指示で、総点検には当たらない |
| @_khrus | 最も成果が出たプロンプトとして `deepen modules and simplify interfaces` を挙げる。重複の除去と中核機能の集約が進み、一貫性の改善、バグ修正、行数の削減につながったと書く。PR ごとに deepening の工程を回し、夜間の自動化で溜まった保守を片付ける。返信に、本人の版の中核を載せる（Claude による要約と本人が書く）。中身は3点である。各モジュールを、単純なインターフェースの裏に多くの振る舞いを隠す形にする。消しても複雑さが呼び出し側に戻るだけなら残し、何も失われないなら統合する。モジュールが自分で決められる引数、どの呼び出し側も同じ値を渡す引数、誰も読まない引数を削り、全ての呼び出し側を新しい形に移して古い経路を消す | @veliorix が挙げた一文を、残すかの判断（消しても複雑さが呼び出し側に戻るだけなら残し、何も失われないなら統合する）と、引数を削る条件にまで具体化している。PR ごとに deepening の工程を回し、夜間の自動化で溜まった保守を片付けるという。別の返信で「レビューとは別の工程にした方が焦点がぶれない。重要なモジュールは2回回す」と書く。効果は自己申告で、Matt Pocock の skills の repo へのリンク（t.co）は開いていない |

### 過剰な防御を止める

| 投稿 | 投稿の内容 | 要点と照合の結果 |
| --- | --- | --- |
| @Rakel076 | Codex が小さな変更にも空判定と try/catch を重ね、ほぼ起きない並行性の問題まで防ぎ、コード量が倍になると書く（中国語）。OpenCode の AGENTS.md に、これを止める行があると紹介する。Style Guide の4条は、自分の AGENTS.md や CLAUDE.md にそのまま写せると書く | 4条は OpenCode の AGENTS.md の Style Guide（L51-55）に原文がある。下の表に英語の原文を載せる |
| @Stargridcompass | Codex の過剰な防御を抑える方法として、OpenCode の AGENTS.md の4条を紹介し、自分の AGENTS.md に写すよう勧める（中国語） | @Rakel076 と同じ4条を紹介している |
| @Voxyz_ai | Opus 5.5 に書かせ、Codex Astra にレビューさせる構成で、Astra が「技術的には起こりうるが、実際にはほぼ起きない」バグを指摘しすぎると書く。レビュー側の指示に1行を足す。各バグを、起こる頻度と深刻さで1から10に採点させ、積が5未満なら Issue にするだけでコードを触らせない。閾値はプロジェクトに合わせる。直す側にも1行を足す。誰も頼んでいない、まれなケースや将来のケースのために設計させない | レビューする側には、頻度と深刻度の積で直すかを決めさせる。直す側には、頼まれていないケースを設計させない |
| @Av1dlive | AGENTS.md に貼る6行で、Opus 5.5 の使用量が50%減ったと書く | 6行は画像だけで、本文を読めなかった。投稿者の返信によれば、無駄の多くは「起きそうにないエッジケース、それ用の巨大なテスト、try/catch、大きすぎる関数」で、4つ目の「説明の文とコメント」を止めると token が47〜50%減ったという。数値は自己申告である |
| @bkrem_ | コードは DRY でなければならない、という指示を載せる。同じ定数、型、検証、ロジックが2か所以上に現れたら、共有の helper に抽出して再利用させる。コピーされたロジックと、繰り返されるリテラルや定義を指摘させる | 2か所以上に出たら共通化させる。OpenCode の「1回しか使わない helper を先に切り出さない」とは向きが逆で、似ただけのコードまで共通化させるおそれがある |

OpenCode の Style Guide の該当行（`ba77ff9` の L51-55）の原文は次のとおり。OpenCode は MIT License で公開されている。

| 行 | 原文 |
| --- | --- |
| L51 | Keep things in one function unless composable or reusable |
| L52 | Validate unknown values once at the boundary that owns them. Pass typed values inward instead of repeating `typeof value === "object"` and property-existence checks. Do not defensively revalidate values already guaranteed by a schema, constructor, or internal type. |
| L53 | Do not extract single-use helpers preemptively. Inline the logic at the call site unless the helper is reused, hides a genuinely complex boundary, or has a clear independent name that improves the caller. |
| L54 | Before adding complexity for a speculative or vanishingly unlikely race or security edge case, explain the concrete failure mode, likelihood, and complexity cost to the user and get their buy-in. Do not silently expand scope for theoretical robustness. |
| L55 | Avoid `try`/`catch` where possible |

### テスト

| 投稿 | 投稿の内容 | 要点と照合の結果 |
| --- | --- | --- |
| @dmokafa | AI が書くテストが悪いのは、開発者から学んだからだと書き、agent に良いテストを書かせる8項目を挙げる。モックを避ける。結合テストに Testcontainers を使う。テストをコードの構造ではなく公開 API に結合させる。直す前に、失敗するテストでバグを再現する。テストにロジックを入れない。mutation テストと property-based テストで、足りないテストを探す。Playwright MCP で実際の利用者の流れを試す。利用者の視点で振る舞いを確かめる。AI が書いたテストの数ではなく、拾えるバグの数を測れと結ぶ | 投稿には有料講座の宣伝が付いている。8項目は一般的な助言で、根拠は示していない。返信で、Playwright で試す流れは「失敗したときの影響が大きい流れを優先する」と書く |

### 進め方

| 投稿 | 投稿の内容 | 要点と照合の結果 |
| --- | --- | --- |
| @sesere115 | 利用者が組織の内側に限られる開発中のサービスを前提に、後方互換とデータ移行を行わず、常に理想的で KISS なコードに書き換えさせる指示を載せる。返信で、開発の順番を4段で示す。資料を先に更新し、矛盾や古い記述が無いことを確かめる。TDD でテストを書く。最小の実装をする。不要なコードと資料の矛盾が無いかを最後に確かめる | 防ぎたいものとして、後方互換やデータ移行のコードの量産、不要になったコードや資料を消さないこと、細かすぎるテスト（犬小屋を作るのに、釘1本ずつにテストを書くようなもの、とたとえる）を挙げる。後方互換を捨てる指示は、利用者のいない開発中のサービスに限られる |

### 総点検の外

| 投稿 | 投稿の内容 | 扱い |
| --- | --- | --- |
| @jpschroeder | monorepo の多くを、親の「coordinator」repo の中に個別の repo を並べる構成に置き換えたと書く。親は `.gitignore` と AGENTS.md だけを持つ。agent を親の root から動かし、複数のプロジェクトにまたがる変更をさせる | repo の構成の話で、Home Repo とほぼ同じ形。総点検のプロンプトではない |
| @sub_827 | Claude や ChatGPT に細かく教えても直らなかった文章が、「助詞・助動詞・接続詞の使い方を見直して」と言うだけで直ったと書く | 日本語の文章の推敲の話。原則の名前を渡すと効く、という点は @veliorix と同じ |

## 資料の間で食い違う点

1. **共通化の閾値。** @bkrem_ は「2か所以上に出たら共通化」とする。OpenCode は「再利用されるか、複雑な境界を隠すか、呼び出し側を読みやすくする名前があるときだけ切り出す」とする。手元のルールは、DRY を「同じ知識か、似ただけのコードか」で判断する。helper は、呼び出し元が2つになってから作る。手元のルールは OpenCode の側に近い。
2. **防御を足す判断を誰がするか。** OpenCode は、足す前にユーザーに説明して同意を得させる。@Voxyz_ai は、頻度と深刻度の積が閾値未満なら、agent に Issue にさせるだけでコードを触らせない。前者は人に、後者は数値の閾値に判断を移す。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた案である。どの設定にも入れていない。

依頼者が手元で使っていた総点検のプロンプトは、次のとおり。

```text
- do a deep scan of all hot paths and fix any unnecessary sequential async functions calls
- find all of the implicit state machines in my app
- deepen modules and simplify interfaces
あと不要なテストを整理する
- 可能な限りモックを避ける
- テストをコード構造に結合させない。パブリックAPIに結合させる
- 修正前に、失敗するテストでバグを再現する
- テストからロジックを排除する
- ユーザーの視点から振る舞いをアサートする
```

これを skill にする草案を、照合者が scratchpad に作った。草案は次の3点に絞った。

- 1回の実行で見る観点を1つにする。@_dylanga が挙げる「差分が読みやすい」を保つため。
- 出力を @alexalbert__ の「確実なものは直し、残りは一覧にする」にそろえる。
- 直すかの判断に @Voxyz_ai の頻度と深刻度の積を使う。

`modules` の観点は @_khrus の版で具体化した。「消しても複雑さが呼び出し側に戻るだけなら残す」は、手元の `reviewer-design` の deletion test と同じ判断である。

似た agent は手元にある。`reviewer-efficiency`、`reviewer-design`、`enhancer-code`、`reviewer-coverage` である。ただし、どれも差分を対象にする。`enhancer-code` だけは、`explicit_files` でファイルを指定できる。草案は、コードベース全体を対象にする点で、これらと分ける。

## 照合の結果

- **投稿の取得：** 14件とも `xr tweet <url> --thread` で取得した。`--thread` の出力には別人の投稿や広告が混ざるため、投稿者自身のものだけを抜き出した。
- **OpenCode の4条：** @Rakel076 と @Stargridcompass が紹介した4条は、OpenCode の AGENTS.md の Style Guide（`ba77ff9` の L51-55）に原文があった。紹介文の「4条」は L51 と L55 をまとめ、L52 から L54 を1条ずつにしている。
- **読めなかったもの：** @Av1dlive の6行（画像）と、@_dylanga の投稿の画像。
- **照合していないもの：** @veliorix の『A Philosophy of Software Design』の章との対応。@Av1dlive と @dmokafa の効果の主張。

## 照合の手順

1. 投稿を取得する：`xr tweet <url> --thread > <id>.md`
2. 投稿者自身のものだけを抜き出す。`author: '@<投稿者>'` の行を持つ項目だけを残す。投稿者の表記は大文字と小文字が URL と違うことがある（例：`@Stargridcompass`）。
3. OpenCode の版を固定する：`gh api repos/anomalyco/opencode/commits/dev --jq .sha`
4. Style Guide の行を探す：`scout repo-read anomalyco/opencode AGENTS.md | grep -nE "## Style Guide|Validate unknown values once|single-use helpers|vanishingly unlikely|try./.catch"`

## 見直す条件

- OpenCode の AGENTS.md の Style Guide が変わったとき。
- 草案の skill を手元で使い、観点ごとの効果を測ったとき。結果は `knowledge/` に書く。
- 手元の DRY と helper のルールを変えたとき。「資料の間で食い違う点」の1が変わる。
