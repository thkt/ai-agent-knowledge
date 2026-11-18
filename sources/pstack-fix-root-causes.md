# 「手早い修正」を頼むと応急処置が広がるという投稿と、pstack の根本原因の原則（2026-11）

poteto の投稿は、agent に最小の変更や手早い変更を頼むと、agent が書くコードが悪くなりうると述べる。Dave Kiss の投稿は、それを引用し、応急処置を見分ける信号として pstack の規則を1つ挙げる。投稿2件と、その規則を持つ skill `principle-fix-root-causes` を、原文で照合した。自分たちが採用した判断ではない。

## TL;DR

- 投稿の主張は、応急処置が次の agent に写され、コードベースに広がる、というものである。変更が広くなっても、根本原因を直させたい場合があると述べる。
- 引用された規則は、pstack の skill の本文にあった。回避策を弁護するのに段落ほどのコメントが要るなら、コードが間違っている、という規則である。
- skill は、ほかに、症状を起こした状態が正当かを問う手順と、バグを隠す典型の5つを書く。
- 測定は無い。広がった件数も、規則の効果も示されない。
- 手元の `~/.claude` は、同じ信号を見る reviewer をすでに持つ。検査としては持たない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [poteto の投稿](https://x.com/poteto/status/2121646643124809848)、[Dave Kiss の投稿](https://x.com/davekiss/status/2121952069053456494)、[cursor/plugins の pstack の principle-fix-root-causes](https://github.com/cursor/plugins/blob/main/pstack/skills/principle-fix-root-causes/SKILL.md) |
| 著者・発行元 | poteto（Lauren Tan）は pstack の作者である。Dave Kiss と pstack の関係は確かめていない |
| 公開日 | poteto の投稿は 2026-11-14。Dave Kiss の投稿は 2026-11-15。skill を最後に変更した commit は 2026-10-15（`e5136d8`） |
| 照合した版 | `cursor/plugins` の main `bf0c7cd` |
| 照合日 | 2026-11-16 |
| 照合した範囲 | 投稿2件の本文と、スレッド内の poteto の返信1件。`pstack/skills/principle-fix-root-causes/SKILL.md`（21行）の全文 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**応急処置は、1箇所で終わらず、次の agent に写されて広がる。** poteto の投稿は、理由を計算の変化として述べる。人が応急処置を手で書いたのは、根本から直す時間がほとんど無かったからである。agent が書くなら、その制約は弱くなる。そのため、最小の変更や最も手早い変更を頼むことが、害になりうると書く。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 最小の変更や手早い変更を頼むと、agent が書くコードが悪くなりうる | 害の無さそうな応急処置が、次の agent に写される。気づいたときには、その形が全体に広がっている | poteto の投稿に記述がある。件数や例は示されない |
| 変更が広くなっても、根本原因を直させたい場合がある | agent は、応急処置を選んでいた理由（時間が無い）を変える。投稿は「場合がある」と書き、常にとは書かない | poteto の投稿に記述がある |
| 段落ほどの弁護のコメントは、応急処置の信号である | agent が応急処置をしたかを知る方法の1つは、回避策が問題にならない理由を説明する段落を書いたかを見ることである | Dave Kiss の投稿に記述がある。投稿のリンク先は poteto の投稿である |
| その場合は、コメントではなくコードを直す | 回避策の正当化に段落ほどのコメントが要るなら、コードが間違っている | skill の20行目に記述がある。Dave Kiss の引用は原文と一致した |
| 症状ではなく、最初に間違えた箇所を直す | 症状は、誤った状態が見えるようになった場所である。根本原因は、システムがすべきでないことをした最初の箇所か、扱うべき状態を扱わなかった最初の箇所である | skill に記述がある。理由として、症状の修正は証拠を隠し、名前の付いていないバグへの回避策をコードに残すと書く |
| 先に再現し、行き詰まったら計測する | 再現を先に行う。答えがコードの誤った判断になるまで、理由を問い続ける。行き詰まったら、推測せず、実際の error と状態を読む | skill に記述がある |
| 症状を起こした状態が正当かを問う | システムがその状態になりうるなら、そこで壊れたコードが根本原因で、その場で状態を扱うのが修正である。なってはいけない状態なら、壊れた場所の検査はバグを隠すので、その状態を作った箇所を探す | skill に記述がある。description も、guard が修正になるのは、扱う状態が正当な場合だけだと書く |
| バグを隠す典型は5つある | guard、retry、握りつぶした error、丸めた値、捨てた状態 | skill に記述がある |
| 1箇所ではなく、同じ型をすべて直す | 同じ欠陥を持つ箇所をすべて探し、すべて直す | skill に記述がある |
| 深く掘って分かった事実は、常に見える信号にする | 決め手の事実を見つけるのに深い調査が要ったなら、最初の一目で分かる、安くて常に出ている信号を提案する | skill に記述がある |

## 限界

- **測定は無い。** 応急処置が写された件数、根本原因の修正に変えた後の結果は、投稿にも skill にも無い。
- **根本原因の修正は、変更が広くなる。** 投稿は「より侵襲的でも」と書く。広い変更がレビューと検証に与える負担は論じていない。
- **返信が挙げる論文は確かめていない。** スレッドの返信の1つは、Google DeepMind が思考の記録を分析した論文があり、agent は悪いと分かっている指示を守るためにトークンを費やすと書く。論文名は示されない。poteto はこの返信に同意している。
- **skill は、利用者が名指しして使う設定である。** frontmatter は `disable-model-invocation: true` で、モデルが自分で呼ぶ対象から外している。どの playbook がこの原則を読ませるかは、読んでいない。
- **作者は当事者である。** poteto は pstack の作者である。

## 手元の資料との関係（未採用）

この節は、照合者が手元の設定と資料を読んで突き合わせた結果である。どの設定も変えていない。

| 投稿と skill の考え方 | 手元の資料 |
| --- | --- |
| 症状ではなく、根本原因を直す | `~/.claude/CLAUDE.md` の Completion は、Fix に「Root cause resolved」を求め、「Symptom patches」を不足とする。`resolver-build` の agent は「根本原因を直す最小の差分」と書き、最小と根本原因を両方求める |
| 段落ほどの弁護のコメントは、応急処置の信号である | `~/.claude` の `reviewer-causation` は、同じ信号を「Justification camouflage」と呼ぶ。1行の why は通し、回避策を合理化する段落を指摘する。追加行を `ugrep` で拾ってから、文脈で判定する2段である。agent によるレビューで、検査ではない |
| 同じ信号を、検査で拾う | [code-comments-for-agents.md](../knowledge/code-comments-for-agents.md) から起票した feedback-loop#186 の「弱」は、TODO や HACK などの語で落とす。語を含まない段落の弁護は通る。同じページは、この点を「語で自己申告しない正当化は、弱では通る」と書いている |
| 応急処置は、次の agent に写される | [all-code-is-slop-poteto-talk.md](all-code-is-slop-poteto-talk.md) の講演は、タイマーによる応急処置と正当化のコメントが広がった例を1件挙げる。手元では、写された例を集めていない |
| 深く掘って分かった事実は、常に見える信号にする | 同じ講演の「訂正は、rules の行ではなく決定論的な検査にする」と同じ向きである。[pstack-overnight-and-self-starting-loops.md](pstack-overnight-and-self-starting-loops.md) の、繰り返す失敗を検査に変える条件にもあたる |

## 照合の結果

- **照合した主張：** 「考え方と根拠」の各行は、右の列に挙げた原文に記述がある。Dave Kiss が引用した規則は、skill の本文と一致した。
- **手元の設定を読んだ範囲：** `~/.claude` の `rules/`、`agents/`、`skills/`、`CLAUDE.md` を、smallest、workaround、bandaid などの語で検索し、当たった箇所を読んだ。全文は読んでいない。
- **読んでいないもの：** この skill を読ませる playbook。`harness/pstack/claude-code/` にある同名の skill。返信が挙げる論文。
- **確かめていないもの：** Dave Kiss と pstack の関係。

## 照合の手順

1. 投稿とスレッドを取得する：`xr tweet https://x.com/poteto/status/2121646643124809848 --thread` と `xr tweet https://x.com/davekiss/status/2121952069053456494 --thread`。出力には無関係の投稿が混ざるので、著者で絞って読む。
2. Dave Kiss の投稿内のリンクの行き先を確かめる。`https://t.co/kP1VFUV6AY` は、poteto の投稿に転送される。
3. 引用された規則の場所を探す：`gh api 'search/code?q=%22paragraph-long+comment%22+repo:cursor/plugins' --jq '.items[].path'`
4. skill を読む：`scout repo-read cursor/plugins pstack/skills/principle-fix-root-causes/SKILL.md`
5. 手元の設定を検索する：`ugrep -rn -i -e 'smallest' -e 'workaround' -e 'bandaid' --include='*.md' ~/.claude/rules ~/.claude/agents ~/.claude/skills`

## 見直す条件

- `principle-fix-root-causes` が変更されたとき。照合した版は `bf0c7cd` である。
- 手元で、応急処置か、それを弁護するコメントが写された例を集めたとき。
- feedback-loop#186 の設計で、コメントの長さを基準に入れるかを決めたとき。
- 返信が挙げる論文を特定できたとき。
