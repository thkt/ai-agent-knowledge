# poteto-mode の24原則を、5本の軸として読む（2026-11）

pstack の入口の skill `poteto-mode` は、24の原則を Core、Architecture、Verification、Delegation、Meta の5つのグループに分けて並べる。原則の一覧とその外の規則を原文で照合し、5本の軸として要約した。グループの名前と所属は原文のもので、軸ごとの一言と問いは照合者の要約である。依頼者はこの5本の軸に同意しており、重要な参照先としてこのページを残す。手元で採用の判断として検証はしていないので、`knowledge/` ではなく `sources/` に置く。

## TL;DR

- poteto-mode は、24の原則を5つのグループに分け、原則ごとに使う場面を添える。原則の名前と所属は、pstack の README の一覧と一致する。
- 5本を一文にすると、最小に作り、正しさは構造に持たせ、実物で確かめ、文脈と人の時間を守って任せ、繰り返す失敗は仕組みにする、となる（照合者の要約）。
- 返答では、どの原則がどの判断を変えたかを名指しさせる。引いてよいのは、そのセッションで本文を読んだ原則だけである。
- 手元の `~/.claude` の規則と重なるのは、Attack the Premise（Strong Inference）、Boundary Discipline、Fix Root Causes、Explain the Number などである。照合者が読んだ範囲に対応が無いのは、Architecture の大半と、Experience First、Exhaust the Design Space、Build the Lever などである。
- 原則ごとの本文は、Fix Root Causes 以外読んでいない。効果の測定は無い。

| 項目 | 内容 |
| --- | --- |
| 出典 | [pstack/skills/poteto-mode/SKILL.md](https://github.com/cursor/plugins/blob/2f87c85/pstack/skills/poteto-mode/SKILL.md) と、[pstack/README.md](https://github.com/cursor/plugins/blob/2f87c85/pstack/README.md) の principles の節（cursor/plugins） |
| 著者・発行元 | poteto（Lauren Tan）。pstack は Cursor のプラグインの repo にある |
| ライセンス | MIT（README の license の節と `pstack/LICENSE`） |
| 公開日 | 2つのファイルを最後に変更した commit は 2026-11-16（`2f87c85`）。最初に公開された日は確かめていない |
| 照合した版 | `2f87c85` |
| 照合日 | 2026-11-19 |
| 照合した範囲 | `poteto-mode/SKILL.md` の全文（110行）。README の principles の節（24原則の表） |
| この資料を根拠にした Issue | なし |

## 中心の主張

**すべての判断を、名前の付いた原則に結び付ける。** poteto-mode は、Non-negotiables の各 trigger を Principles の節の原則が支えると書く。返答では、判断を変えた原則と、それが変えた具体の選択を名指しさせる。引用してよいのは、そのセッションで原則の本文（leaf の SKILL.md）を読んだものだけである。

## 5本の軸

軸の名前は原文のグループの名前である。「一言で」と「自分に問うこと」は照合者の要約である。

| 軸 | 一言で | 自分に問うこと | 原則の数 |
| --- | --- | --- | ---: |
| 1. Core（作り方） | 足す前に削り、最初からそうだったように作る | もっと小さい変更で済まないか。後付けになっていないか | 10 |
| 2. Architecture（構造） | 正しさを、条件分岐ではなく型と構造に持たせる | この誤りは、型や構造で起こらなくできないか | 6 |
| 3. Verification（確かめ方） | 代理の指標ではなく実物で、症状ではなく原因で確かめる | 「通った」は実物で見たか。その数字は何を測っているか | 5 |
| 4. Delegation（任せ方） | 文脈を汚さず、人を待たない | これは subagent に回せないか。取り消せる作業で確認を求めていないか | 2 |
| 5. Meta（学び方） | 同じ指示を2回書いたら、文章ではなく仕組みにする | この教訓を、lint や検査や script にできないか | 1 |

原則の数は Core に偏るが、軸の重さは数と別である。照合者の読みでは、poteto の講演（[all-code-is-slop-poteto-talk.md](all-code-is-slop-poteto-talk.md)）が強調するのは、Architecture、Verification、Meta にあたる部分である。たとえば、訂正を lint にすること、`useEffect` を禁止して判断の余地を無くすこと、である。

## 原則と使う場面

「使う場面」と「規則」は、poteto-mode と README の一覧の要約である。

| 軸 | 原則 | 使う場面 | 規則 |
| --- | --- | --- | --- |
| Core | Laziness Protocol | リファクタリング、diff の大きさを決めるとき、抽象や層を足したくなったとき | 削る側に寄せ、問題を解く最小の変更にする |
| Core | Foundational Thinking | ロジックを書く前 | 中心の型とデータ構造、土台と機能の順番、並行する主体が何を共有するかを先に決める |
| Core | Redesign from First Principles | 新しい要件を既存の設計に入れるとき | 最初からその要件が前提だったように設計し直す |
| Core | Attack the Premise | 同じ前提を共有する2つ以上の修正が、同じ検査で落ちたとき | 前提を書き出し、次の修正を書く前に前提そのものを試す |
| Core | Subtract Before You Add | 追加、リファクタリング、書き直しの順番を決めるとき | 不要なものを先に消し、単純になった土台の上に作る |
| Core | Minimize Reader Load | 追いにくいコードを読むか、形を整えるとき | 問いから答えまでの層と、読み手が抱える隠れた状態を数え、呼び出し元が1つの wrapper を畳む |
| Core | Outcome-Oriented Execution | 段階の境目がはっきりした書き直しや移行 | 目標の構造に収束させ、使い捨ての互換の状態を保たない |
| Core | Experience First | 製品、UX、機能の範囲の取捨 | 実装の都合より利用者の満足を選び、荒い機能を多く出すより、磨いた機能を少なく出す |
| Core | Exhaust the Design Space | 前例の無い操作か構造の決定 | 競う試作を2〜3作り、並べて比べてから決める |
| Core | Build the Lever | 自明でない作業すべて | 手作業ではなく、作業をする道具か、作業を証明する道具（codemod、script、生成器）を作る。道具はレビュアーが再実行できる成果物である |
| Architecture | Model the Domain | 状態を持つロジック、分岐の多いコード、同じ形の前提が複数のファイルに繰り返すとき | ドメインを、散らばった条件分岐ではなく構造（状態機械、型付きのモデル、表、reducer）で表す |
| Architecture | Boundary Discipline | 検証、エラー処理、framework の adapter をつなぐとき | 検査はシステムの境界に集め、内側は型を信頼し、業務ロジックを純粋に保つ |
| Architecture | Type System Discipline | 型か関数の形を設計するとき | 不正な状態を型で表せなくし、意味のある基本型に印を付け、外部のデータは境界で解析する |
| Architecture | Make Operations Idempotent | 落ちたり再試行されたりする中で動くコマンド、手順、ループ | 途中までの実行があっても、同じ終わりの状態に収束させる |
| Architecture | Migrate Callers Then Delete Legacy APIs | 古い呼び出し元が残る中で、新しい内部 API を入れるとき | 呼び出し元を移し、古い API を同じ回で消す |
| Architecture | Separate Before Serializing Shared State | 並行する主体が、同じファイル、branch、key、object に書くかもしれないとき | 直列化する前に、共有そのものを無くす |
| Verification | Prove It Works | 作業の後、完了を宣言する前 | 代理の指標や自己申告や「コンパイルが通る」ではなく、実物（機能を動かす、実際の値を読む、diff を見る）で確かめる |
| Verification | Fix Root Causes | デバッグ | 各症状を、システムが最初に誤った場所までたどって直す。guard が修正になるのは、それが扱う状態が正当なときだけである |
| Verification | Sequence Work into Verifiable Units | 複数の段の作業と、commit と PR の積み方 | 作業を、それぞれ確かめられる状態で終わる小さな単位に分け、次の前に確かめ、並びそのものが証明になる順に出す |
| Verification | Test Behavior, Not Implementation | テストを書く、変える、残すとき | 利用者から見える振る舞いを確かめ、内部だけが変わったときは通り続けるようにする |
| Verification | Explain the Number | 測る前と、測った数字を信じる、報告する、使う前 | その数字の上限を決めているものを突き止め、思っている作業とは別のものを測っていないことを確かめる |
| Delegation | Guard the Context Window | 大きな出力、長いファイル、繰り返しの読み込みで context が埋まるとき | 量の多い作業は subagent に回し、本筋には要約だけを残す |
| Delegation | Never Block on the Human | 取り消せる作業で「X をしてよいか」と聞きたくなったとき | 進めて結果を見せ、人に後から直してもらう |
| Meta | Encode Lessons in Structure | 同じ指示を2回目に書いていると気づいたとき | 文章を足す代わりに、lint、メタデータの印、実行時の検査、script にする |

## 原則の外の規則

poteto-mode は、原則の一覧のほかに次の規則を持つ。

- **最初の返答：** 利用者の目標と、それを満たす最小の変更を言い直してから進む。subagent への指示、PR の Why、最後の報告も、目標から始める。
- **実行で決められる分かれ道：** 質問せず、試作で決める。
- **コードを書くとき：** 先にデータの形を名指しする。
- **自律：** 取り消せる作業と外部への操作は、確認せずに進める。止まるのは、取り消せない書き込み（共有 branch への force-push、deploy、データの削除、顧客へのメッセージ）のときである。
- **No も正しい答え：** 同意を既定にしない。価値が無ければ、そう言う。
- **返答の書き方：** 各主張に証拠か、測った、推論した、推測した、のラベルを付ける。自分で走らせられる確認を、人に渡さない。

## 限界

- **軸の一言と問いは、照合者の要約である。** 原文の文ではない。
- **原則ごとの本文は、ほぼ読んでいない。** 表の「使う場面」と「規則」は、poteto-mode と README の一覧の要約である。本文を照合したのは Fix Root Causes だけである（[pstack-fix-root-causes.md](pstack-fix-root-causes.md)）。
- **効果の測定は無い。** 原文は原則を規則として述べ、守った場合と守らない場合の差を示さない。
- **作者は当事者で、Cursor の機能を前提にする部分がある。** Task tool、Project の worker、`~/.cursor/rules` の設定、モデルの割り当ては Cursor 向けである。原則の一覧そのものは、Cursor に依存しない。
- **「5本の軸」という呼び方の出どころは推測である。** 呼び方は、2026-11-18 の [mpyw の投稿](https://x.com/realmpyw/status/2122973409080758660)による。投稿は資料を引用しておらず、poteto-mode の5つのグループを指すというのは、依頼者と照合者の推測である。

## 手元の資料との関係（未採用）

この節は、照合者が手元の `~/.claude` の規則と突き合わせた結果である。照合者が読んだのは、`CLAUDE.md`、`rules/PRINCIPLES.md`、`rules/core/` の4ファイル（PREFLIGHT、BOUNDARIES、OUTCOME、OPERATION）、`rules/development/TOOLS.md`、`rules/conventions/` の PROSE と MIRROR である。ほかの規則にあるかは確かめていない。どの設定も変えていない。

| poteto-mode の原則 | 手元の規則 | 違い |
| --- | --- | --- |
| Laziness Protocol、Subtract Before You Add | PRINCIPLES.md の Occam's Razor と、BOUNDARIES.md の Reuse Ordering の1（要らないものは作らない）（[BOUNDARIES.md:44](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L44)） | 同じ向きである |
| Minimize Reader Load | BOUNDARIES.md の Readable Code（[:16](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L16)）と、PRINCIPLES.md の Miller's Law | 同じ向きである。poteto は、層と隠れた状態を数えるという測り方を示す |
| Attack the Premise | BOUNDARIES.md の Strong Inference（[:51-53](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L51-L53)） | ほぼ同じ規則である。どちらも、同じ前提の修正が2回失敗したら前提を試す |
| Outcome-Oriented Execution | CLAUDE.md の Outcome-driven | 語が同じで意味が違う。poteto は、移行で目標の構造に収束し、互換のコードを残さないことを指す。手元は、done の状態での振る舞いを outcome と呼ぶ |
| Boundary Discipline | BOUNDARIES.md の Overeagerness の「validation はシステムの境界に置く」（[:73](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L73)） | 同じである |
| Prove It Works | OPERATION.md の Output Verifiability（[:13](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/OPERATION.md?plain=1#L13)） | 同じ向きである |
| Fix Root Causes | CLAUDE.md の Completion の Fix の行（[:34](https://github.com/thkt/dotclaude/blob/8a30604/CLAUDE.md?plain=1#L34)） | 同じである。どちらも、症状の手当てを修正と数えない |
| Sequence Work into Verifiable Units | PRINCIPLES.md の TDD / Baby Steps と、PREFLIGHT.md の Task Decomposition（[:26](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/PREFLIGHT.md?plain=1#L26)） | 近い。手元は分ける閾値（ファイル数、行数）を持ち、poteto は並びが証明になる順番を求める |
| Explain the Number | BOUNDARIES.md の Measurement（[:55-60](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L55-L60)） | 近い。手元は、数字が良くなっても outcome が動かなければ指標が誤りだとする。poteto は、数字の上限と測った対象を確かめる手順を求める |
| Never Block on the Human | BOUNDARIES.md の Overeagerness の「分かれる判断を、それに依存しない部分の次の行動と同じ返答で利用者に返す」（[:75](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L75)）。OPERATION.md の Knowledge gaps の「人に聞き、答えに依存しない手順を続ける」（[:18](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/OPERATION.md?plain=1#L18)）。PREFLIGHT.md の Interpretation Clarity（[:20](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/PREFLIGHT.md?plain=1#L20)） | 一部が違う。手元は、答えの要る判断を利用者に返しながら、それに依存しない部分を進める。計画の承認を待つのは、Issue や外部の仕様を受けた対話中のセッションに限る。poteto は、取り消せる作業なら聞かずに進め、後から直してもらう |
| Encode Lessons in Structure | BOUNDARIES.md の Systems Thinking の Shifting the Burden（[:14](https://github.com/thkt/dotclaude/blob/8a30604/rules/core/BOUNDARIES.md?plain=1#L14)）と、PRINCIPLES.md の trigger「同じ症状の修正が繰り返す」（[:39](https://github.com/thkt/dotclaude/blob/8a30604/rules/PRINCIPLES.md?plain=1#L39)） | 近い。poteto は、置き場（lint、メタデータ、実行時の検査、script）を名指しする |
| Foundational Thinking、Redesign from First Principles、Experience First、Exhaust the Design Space、Build the Lever、Model the Domain、Type System Discipline、Make Operations Idempotent、Migrate Callers Then Delete Legacy APIs、Separate Before Serializing Shared State、Test Behavior Not Implementation、Guard the Context Window | 読んだ範囲に対応する規則は無い | ほかの規則にあるかは確かめていない |

## 照合の結果

- **照合した主張：** poteto-mode の Principles の節にある24原則の名前と5つのグループは、README の principles の表と一致する。グループごとの数は、Core 10、Architecture 6、Verification 5、Delegation 2、Meta 1 である。
- **動かしていないもの：** poteto-mode、pstack。
- **読んでいないもの：** 原則ごとの SKILL.md（Fix Root Causes を除く）、`playbooks/`、`worker.md`。

## 照合の手順

1. 版を確かめる：`gh api 'repos/cursor/plugins/commits?path=pstack/skills/poteto-mode&per_page=1' --jq '.[0].sha'`
2. poteto-mode を読む：`scout repo-read cursor/plugins pstack/skills/poteto-mode/SKILL.md`
3. README の principles の節を読む：`scout repo-read cursor/plugins pstack/README.md` の出力のうち、「## principles」から「## not shipped here」まで。
4. グループごとの数を数える：README の表の group の列を数え、poteto-mode の Principles の節の見出しごとの数と比べる。

## 見直す条件

- poteto-mode の Principles の節か、README の principles の表が変わったとき。照合した版は `2f87c85` である。
- 依頼者が、5本の軸のどれかを `~/.claude` の規則に入れると決めたとき。そのときは `knowledge/` にページを書き、このページを根拠として引く。
- 原則ごとの SKILL.md を読んだとき。表の「使う場面」と「規則」を、本文で照合し直す。
