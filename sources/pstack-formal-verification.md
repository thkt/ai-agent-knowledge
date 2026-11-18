# pstack の形式検証 skill と statechart skill の考え方と根拠（2026-11）

pstack は、Cursor の公式プラグインのリポジトリで公開されている agent 用の skill 集である。作者の投稿が勧める2つの skill を、原文で照合した。状態機械を形式検証の言語でモデル化して検査する skill と、その結果を状態遷移図にする skill である。自分たちが採用した判断ではなく、手元では動かしていない。

## TL;DR

- pstack の2つの skill は、状態機械を TLA+ でモデル化して検査し、結果を状態遷移図にする。
- 検査は、次の agent が使える project-local の skill として残す。失敗を見たことのない検査は信じない。
- TLC と Java が要る。効果の件数や時間は書かれていない。
- 手元では動かしていない。Java を入れない形は [state-machine-check-in-app-language.md](../knowledge/state-machine-check-in-app-language.md) で確かめた。

| 項目 | 内容 |
| --- | --- |
| 出典 | [poteto の投稿](https://x.com/poteto/status/2119521036660314339)と、[cursor/plugins の pstack](https://github.com/cursor/plugins/tree/main/pstack) |
| 著者・発行元 | poteto（Lauren Tan）。pstack は Cursor のプラグインのリポジトリにある |
| 公開日 | 投稿は 2026-11-08。pstack を最後に変更した commit は 2026-11-15 |
| 照合した版 | `cursor/plugins` の main `bf0c7cd` |
| 照合日 | 2026-11-16 |
| 照合した範囲 | 投稿と、スレッド内の作者の返信。`pstack/skills/create-formal-verification-skill/SKILL.md`（43行）と `pstack/skills/statechart/SKILL.md`（82行）の全文。`pstack/README.md` の冒頭と skill の一覧 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**テストは誰かが思いついた場合だけを確かめ、形式検証は宣言した範囲のすべての場合を確かめる。** `create-formal-verification-skill` は、この前提から始まる。対象は、壊れると利用者が困り、テストが届きにくい約束である。検証しやすい箇所ではなく、repo の中で最も重要な箇所を選ぶと書く。

投稿は、Steve Ruiz の「アプリの中の暗黙の状態機械を全部見つけて」という prompt を引用し、次の一手として2つの skill を勧める。見つかった状態機械を、サブシステムごとに1つの skill としてモデル化する。重要なユーザーフローを、状態遷移図で可視化する。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 検査を、次の agent が使える skill として残す | 生成するのは project-local の skill である。次の agent は、その skill だけから、変更が検証済みのコードに触れるかを判定し、検査を走らせ、本物のバグとモデルの誤りを見分ける | 原文に記述がある。作者の返信は、例として `/verify-notifications` を挙げる |
| 問いが道具を選ぶ | 事象が悪い順序で起こりうるかは、設計の小さなモデルに対するモデル検査が合い、どの言語でも TLA+ を既定とする。全入力でのコードの振る舞いは、コード上で動く検証器が合う。その言語に検証器が無ければ、そう述べたうえで、生成した入力で実コードをテストする | 原文に記述がある |
| 失敗できる検査だけを信じる | 失敗を見たことのない検査は、何も証明しない。各性質が、守る対象を壊した版（検査の削除、手順の入れ替えなど）で落ちることを示す | 原文に記述がある |
| 発見は発見のまま報告する | 現行のコードが性質を破るなら、コードの言葉で書いた反例を付けて報告する。通るまで性質やモデルを弱めない。利用者が頼まない限り、製品のコードを変えない | 原文に記述がある |
| 範囲を正直に書く | 何を、どの上限と前提のもとで検証し、何を外したかを書く。合格が覆うのは、モデルか上限の内側であり、システム全体ではない | 原文に記述がある |
| 道具を知らなくても読める | 性質と害を、平文で1文ずつ書く。反例は、コードの言葉で短い手順として書く | 原文に記述がある |
| 安く再実行できる | 1つのコマンド、再現できる道具の版、次の agent が関係する変更のたびに使える長さ | 原文に記述がある |
| コードと歩調を合わせる | 各検査が覆うファイルと関数、検証がコードを表していると最後に確かめた commit を記録する | 原文に記述がある。ずれた後に合わせ直す skill として `maintain-formal-verification-skill` を挙げる。その skill は読んでいない |
| 引き渡す前に、実コードを壊して確かめる | 生成した skill を、次の agent と同じ手順で一度たどる。性質を破る仮の変更を実コードに入れ、skill が検出することを確かめる。一度も走らせていない verify skill は下書きである | 原文に記述がある |
| 図は、判断する人の言葉で描く | `statechart` は、TLA+ のモデルを TLC で検査し、Mermaid の状態遷移図にする。状態と矢印のラベルは、変数名やコードの名前ではなく、読み手が見るものの言葉で書く | 原文に記述がある。矢印は20本以下を目安とする |
| 破る状態と、そこへの最短の経路を示す | 規則を破る状態に印を付け、各規則について最短の経路を報告する。規則が破れたら、その規則を外して TLC を走らせ直す | 原文に記述がある |
| モデルを疑う | モデルがコードを表す場合、見つかった経路を実コードで、テストか実際の再現として1手ずつたどる。バグが起きなければ、モデルが間違っていると考える | 原文に記述がある |

## 限界

- **TLC と Java が要る。** `statechart` は、初回の実行で TLC と Java を mise で入れると書く。TLC は TLA+ のモデル検査器で、Java で動く。
- **検査するのはモデルである。** 合格は、モデルと上限の内側だけを覆う。モデルとコードのずれは、対象の commit の記録と、経路の再現で詰める。
- **効果の測定は無い。** 作者の返信は、複数のクライアント（デスクトップ、iOS、Android）にまたがる通知で、重複を防ぎ既読を合わせる用途に効いたと書く。件数や時間は書いていない。
- **pstack 全体は Cursor 向けである。** README は Cursor での利用を前提に書かれている。Claude Code 向けの文書（`docs/harness/claude-code.md`）は読んでいない。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた案である。どの設定にも入れていない。

| pstack の考え方 | 手元の資料 |
| --- | --- |
| 暗黙の状態機械を見つけてから、モデル化する | `~/.claude` の `sweep` skill の `state` lens は、フラグや真偽値に散った暗黙の状態機械を探し、明示的な状態と遷移を持つ型にする。投稿が引用する prompt と同じ問いで、モデル化はその次の段にあたる |
| 失敗できる検査だけを信じる | [measure-tests-before-adding-practices.md](../knowledge/measure-tests-before-adding-practices.md) は、壊した実装45種でテストを測った。[formal-methods-playbook-mizchi.md](formal-methods-playbook-mizchi.md) の broken-variant と同じ手法である |
| 事象の順序は TLA+ で確かめる | [state-machine-check-in-app-language.md](../knowledge/state-machine-check-in-app-language.md) は、Java を入れずに、アプリの言語で状態を全列挙する形を題材で確かめた。「いつか必ず起こる」性質は、その形の対象外である |
| モデルを疑い、経路を実コードで再現する | 同じページで、モデルから作った経路で手書きの実装を動かした。状態ごとの最短経路だけでは実装の欠陥を拾えず、遷移ごとの経路で拾えた |

## 照合の結果

- **照合した主張：** 「考え方と根拠」と「限界」の各行は、右の列に挙げた原文に記述がある。
- **動かしていないもの：** pstack、TLC、`statechart` のスクリプト。
- **読んでいないもの：** `maintain-formal-verification-skill`、`create-verification-skill`、`statechart/references/render.md`、`docs/guide/` の各章、README の残りの節。第7章は [pstack-overnight-and-self-starting-loops.md](pstack-overnight-and-self-starting-loops.md) で照合した。
- **読めなかったもの：** スレッド内で作者がリンクした `@mattyp` の投稿2件。ログインを求められ、本文を取得できなかった。

## 照合の手順

1. 投稿とスレッドを取得する：`xr tweet https://x.com/poteto/status/2119521036660314339 --thread`。作者の投稿だけを読む。
2. 投稿内のリンクの行き先を確かめる：`scout fetch https://t.co/SxilBe4Diu`。`cursor/plugins` の `pstack` に着く。
3. 版を確かめる：`gh api 'repos/cursor/plugins/commits?path=pstack&per_page=1' --jq '.[0].sha'`
4. ファイルの一覧を取得する：`scout repo-tree cursor/plugins -p pstack`
5. 2つの skill を読む：`scout repo-read cursor/plugins pstack/skills/create-formal-verification-skill/SKILL.md` と `scout repo-read cursor/plugins pstack/skills/statechart/SKILL.md`

## 見直す条件

- pstack の2つの skill が変更されたとき。照合した版は `bf0c7cd` である。
- `statechart` が TLC 以外の検査器に対応したとき。
- 手元で TLC か FizzBee を動かし、アプリの言語での全列挙と結果を比べたとき。
