# Makarevych「How to write and test agent skills with evals」の考え方と根拠（2026-10）

この記事が示す agent skill の作り方と検証の考え方を、根拠と照合した結果とあわせてまとめる。自分たちが採用した判断ではない。

## TL;DR

- skill の失敗には2種類ある。agent が指示どおりに動かない場合と、動いても成果が良くならない場合である。
- skill ごとに adherence と effect の2つを、自分のモデルと環境で測る。
- 引用はすべて実在した。abstract と合わない数値が2件、原文より強く書かれた箇所が4点あった。
- 手元では、自作の31 skill のうち3つが、90日のあいだ一度も読み込まれていなかった。

| 項目 | 内容 |
| --- | --- |
| 出典 | [How to write and test agent skills with evals](https://x.com/sermakarevich/status/2112198653469241856)（X の Article） |
| 著者 | Sergii Makarevych（@sermakarevich） |
| 公開日 | 2026-10-19 |
| 照合日 | 2026-11-13 |
| 照合した範囲 | 引用の実在、arXiv の abstract に載る数値、arXiv 以外の出典の本文、手元のハーネスでの再現 |
| この記事を根拠にした Issue | [thkt/dotclaude#781](https://github.com/thkt/dotclaude/issues/781)（rules の要素の effect を測れるかを 1 要素の pilot で確かめる） |

## 中心の主張

記事の中心の主張は次のとおり。**skill の失敗には2種類あり、成果を見るだけでは区別できない。そのため skill ごとに adherence と effect の2つのスコアを、自分のモデルと環境で測る。**

- adherence: agent が skill の指示どおりに動いたか
- effect: 指示どおりに動いた結果、成果物が良くなったか

記事は skill を「agent の振る舞いについての仮説を文章で書いたもの」と定義する。skill を入れてもモデルは変わらず、context に文章が増えるだけである。

## 判定の規則

記事は、2つのスコアの組み合わせで次の手を決める。effect は、skill なしの arm と、長さを揃えた無関係な文章の arm（length control）の両方と比べる。

| adherence | effect | 次の手 |
| --- | --- | --- |
| 低い | 問わない | 文面を書き直す。idea の良し悪しはまだ分からない |
| 高い | 変わらない | その idea（point）を捨てる |
| 高い | 上がった | 増えた費用と並べて、受け入れるかを決める |

## 考え方と根拠

| 考え方 | 記事が挙げる根拠 | 照合の結果 |
| --- | --- | --- |
| skill の費用は3か所で発生する。listing は毎ターン、本文は読み込んだ後ずっと、hook はセッション開始ごとにかかる | Claude Code の skills ドキュメントの "every line is a recurring token cost"。著者の開発機では 89 skill 中 81 が一度も呼ばれていなかった | 引用は実在する。ただし原文で主語になっているのは、読み込んだ後の本文である。listing の費用は同じページの "Find unused skills" 節が "Every skill in the skill listing adds to your context on every turn, whether or not Claude ever uses it." と書いている。89 と 81 は著者の手元の値で、照合できない |
| モデルが自分で skill を選ぶのは半分前後で、候補が増えるほど正しく選べなくなる | Vercel のブログ（56% の case で一度も呼ばれない。skill ありも docs なしも 53%。明示の指示で 79%。常時読む docs index で 100%）。Spence の再試験（22 prompt、Sonnet 4.5、強制 hook なしで 50〜55%、ありで 100%）。SRA-Bench（arXiv:2604.24594）。Demystifying Agent Skills（arXiv:2608.14036、候補 5 個で 29.6%、100 個で 3.3%） | Vercel の4つの数値は一致する。ただし記事が引用符で書く "invoke this skill" は原文の文言ではない。100% は skill の強制ロードではなく、AGENTS.md に埋め込んだ docs index の結果である。Spence の数値も一致する。ただし 100% に達したのは forced-eval と llm-eval の2種類の hook だけである。SRA-Bench の 5,400 件と「必要かどうかに関わらず、似た率で skill を読み込む」は abstract と一致する。記事の 11.1〜72.1% は abstract に無く、未照合である。29.6% から 3.3% への低下は abstract と一致する |
| 公開されている skill の多くは効果がなく、token だけが増える | Signal or Noise（arXiv:2608.23067）。SWE-Skills-Bench（arXiv:2603.15401、49 skill 中 39 で改善ゼロ、平均 +1.2pt、token 最大 451% 増、3 skill が最大 10pt 悪化）。Agent Skills Can Be Harmful（arXiv:2608.11888、307 件中 182 件が費用だけの後退） | Signal or Noise の 31 skill、50 project、4 モデル、Pass@2 の 1.3〜4.2% 低下、token 72〜394% 増は abstract と一致する。Sonnet 4 で 4.2pt という帰属と、「4 モデル全てで改善した組は 1%」は未照合である（abstract は改善した組を 17〜36% と書く）。SWE-Skills-Bench の数値は v1（2026-03-16）の abstract と全て一致する。Harmful の 307 件中 182 件は一致する。記事 Level 4 の「62.6% が費用の後退」は、abstract の件数から計算した 59.3%（182/307）と合わない |
| 公開 skill の4分の1に脆弱性のパターンがあり、検査ツールは回避される | Agent Skills in the Wild（arXiv:2601.10338、26.1%、script 同梱で 2.12 倍）。Do Not Mention This to the User（arXiv:2602.06547、98,380 件中 157 件が悪意あり）。Trail of Bits の調査を紹介した CSA の note（5 製品を回避） | 3つとも数値は一致する。USENIX Security 2026 への採録は abstract に無く、未照合である。Trail of Bits の「1時間未満」は、4つの手法のうち3つの開発時間を指す。記事の Takeaway 5「scanner は1時間未満で回避された」は、原文より強い書き方である |
| skill の効果はモデルと環境ごとに変わり、他の環境へ持ち込めない | Signal or Noise（同じ skill が Sonnet 4 で +33pt、DeepSeek と Qwen で −22pt）。SkillsBench（arXiv:2602.12670、skill ありの Haiku 4.5 が 27.7%、skill なしの Opus 4.5 が 22.0%） | Signal or Noise の abstract は「skill の順位はモデル間で弱くしか移らない」と書き、skill を「skill・project・model の組についての仮説」と位置づける。この主張と一致する。+33pt と −22pt は未照合である。SkillsBench の「skill を持つ小さいモデルが、持たない大きいモデルに並ぶ」は v1 と v4 の abstract と一致する。27.7% と 22.0% は未照合である |
| 「自分の手元で効いた」は根拠にならない。効く skill は agent を手順に固定することで効く | Demystifying Agent Skills（2.8pt の改善の信頼区間が 0 をまたぐ。効果の 65.7% が手順への固定、4.5% が知識の追加）。Anthropic のデータ分析のブログ（21% から 95% 超へ） | 65.7% と 4.5% は abstract と一致する。2.8pt と信頼区間は abstract に無く、未照合である（abstract の主要な数値は Workflow Memory に対する +6.06pt）。Anthropic の 21% と 95% は一致する。ただし原文で 21% は上限（"didn't exceed"）である。記事が添える「1トピックあたり数十件の eval」は原文では収穫逓減の目安として書かれ、95% を達成した条件としては書かれていない |
| 目的を測定できる1文で書き、核になる point を仮説つきで短く並べる。point は skill なしで走らせたときの失敗から取るのが最も信頼できる | Agentic Context Engineering（arXiv:2510.04618、失敗から作った playbook で 12.3% の改善） | abstract は v1 も v3 も agent で +10.6%、finance で +8.6% と書く。12.3% は abstract に無い |
| effect はテストの数ではなく質で測る。テストを1本ずつ、変更前のコードと仕込んだ欠陥に対して実行してから label を付ける | Rethinking the Value of Agent-Generated Tests（arXiv:2602.07900）。Coverage と mutation の再現研究（arXiv:2607.22880）。LLM 生成コードの欠陥 6,000 件超（arXiv:2609.09315）。GTDD（arXiv:2610.02952）。Radzymiński の mutation testing の記事（line coverage 100% で mutation score 61%） | 5つとも主張と一致する。Rethinking の abstract は、解けた task と解けなかった task でテストを書く頻度の差が「小さい」と書き、記事の「同じ」より弱い。Radzymiński の原文の値は 61.29% で、改善前の値である |
| eval は skill なし・skill・length control の arm を置き、各3 run、モデルごとに回す。タスクはモデルの学習の締め切り後に集めた実リポジトリのものを使う | Signal or Noise（同じ長さの無関係な skill で 3.3pt と 3.5pt の低下）。SWE-bench-Live（arXiv:2505.23419）。SWE-rebench（arXiv:2505.20411）。Claude Code の plugin evals ドキュメント | Signal or Noise の abstract は、一部のモデルでは同じ長さの無関係な skill が低下の大半を再現したと書く。3.3pt と 3.5pt は未照合である。記事の Takeaway 4「同じ長さの無関係な文章が skill と同じくらい結果を動かした」は、abstract では一部のモデル（length-distracted）に限られる。SWE-bench-Live と SWE-rebench は一致する。plugin evals の、fresh session、既定3回の実行、with/without の比較、regex・tool_used・file_exists と LLM の grader、`--model`、費用の表示は一致する |
| skill は文面・point・eval・結果の4点を揃えて公開し、モデルや依存が変わるたびに eval をやり直す | Anthropic のデータ分析のブログ（1か月で 95% から 65% へ低下）。Memento-Skills（arXiv:2603.18743、test gate ありで 52.3% から 66.0%）。arXiv:2607.16621（gate なしで 6.2〜11.5pt 低下） | 95% と 65% は一致する。ただし原文が挙げる原因は「データモデルが日々変わる」ことである。記事 Takeaway 14 の「モデル更新後に再実行しなかった」とは違う。Memento-Skills の abstract は GAIA で相対 +26.2% と書き、52.3% から 66.0% と整合する。改善を test gate に帰す記述は abstract に無い。arXiv:2607.16621 は記事の題と実際の題が違い、6.2〜11.5pt も未照合である |
| skill の仕様は形式だけを定め、テストや「どのモデルで試したか」の欄を持たない | Agent Skills specification | test、eval、試したモデルの欄が無いことは一致する。ただし環境の要件を書く `compatibility` 欄はある（"Indicates environment requirements (intended product, system packages, network access, etc.)"）。記事 Level 3 の「environment の欄が無い」とは食い違う |
| 例に使った TDD skill は、毎セッション注入される hook と長い本文を持つ | obra/superpowers（skill 15 個、hook が注入する 492 語、TDD skill 1,475 語、付属 1,310 語、コード例は TypeScript と npm） | 記事の公開前で最後の main の commit（`bb92a77`、v7.0.0）で数え、全て一致した。v7.0.2 でも同じだった。hook は startup に加えて clear と compact でも発火する。TDD skill の本文はテストの実行コマンドを "bare `pytest`, `npm test`, `cargo test` — whatever the repo uses" と書いている。TypeScript と npm に限られるのはコード例だけである |

## 照合の結果

- **arXiv の引用 17 件**：すべて実在した。記事とタイトルが違うのは次の2件である。
  - arXiv:2604.24594：実際の題は "Skill Retrieval Augmentation for Agentic AI"。記事の副題 "lost in implicit routing" は無い
  - arXiv:2607.16621：実際の題は "From Memory to Skills: Evidence-Grounded Co-Evolution Governance for Long-Horizon LLM Agents"（第一著者 Bo Tang）。記事は "Meta-Context Engineering via agentic skill evolution" と書く
- **abstract と合わない数値**：次の2件である。
  - arXiv:2510.04618：記事は 12.3%、abstract は v1 も v3 も +10.6%
  - arXiv:2608.11888：記事は 62.6%、abstract の件数から計算すると 59.3%
- **arXiv 以外の引用 9 件**：すべて実在した。原文より強く書かれているのは次の4点である。
  - Claude Code ドキュメントの引用は、読み込んだ後の本文の費用についての文である
  - Vercel の 100% は AGENTS.md の docs index の結果で、skill の強制ロードの結果ではない
  - Trail of Bits の「1時間未満」は、4つの手法のうち3つの開発時間である
  - Anthropic の精度低下の原因は、記事の Level 10 では正しく「データの変化」と書かれている。Takeaway 14 だけが、モデル更新後に再実行しなかったことに結びつけている
- **事実と食い違うもの**：Agent Skills specification には環境の要件を書く `compatibility` 欄がある。記事 Level 3 の「environment の欄が無い」と食い違う。
- **照合できないもの**：著者の手元の値（89 skill 中 81 が未使用、最後の使用が 37 日前）。
- **利害関係**：記事は製品を勧めていない。ツールの節は "as observed rather than endorsed" と断っている。

## 記事の公開後に変わった点

- SWE-Skills-Bench（arXiv:2603.15401）の v2 が 2026-10-27 に出た。対象は 98 skill に増え、92 skill で有意な効果が出なかった。平均の合格率は 76.6% から 76.9% だった。害の原因は、v1 の「古い版向けの指示」から「skill の雛形が task の指示を上書きする」に変わった。記事 Level 2 と Level 10 の「古い指示が害の主な原因」は、v2 では支えられない
- Rethinking the Value of Agent-Generated Tests（arXiv:2602.07900）の v3 が 2026-11-09 に出た。照合は v3 の abstract で行った

## 手元のハーネスでの観測

記事は「他人の環境での試験は自分の環境の試験にならない」と主張する。そこで、手元の `~/.claude` で同じ問いを2つ確かめた。

| 観測 | 結果 | 対応する記事の主張 |
| --- | --- | --- |
| 2026-11-13 に、`~/.claude/projects` の transcript のうち更新日が 90 日以内のもの（7,528 ファイル）から、skill の呼び出しを数えた | 自作の 31 skill のうち、どの経路でも読み込まれなかったのは 3 つ（`census`、`transcribe`、`use-cli-ast-grep`）だった。Skill tool の呼び出しだけで数えると 7 つに見える。差の 4 つ（`use-context-reviewer-*`）は agent の frontmatter の `skills:` で読み込まれ、Skill tool を通らない。この 4 つは 9〜23 件の transcript で読み込まれていた | Level 1 の「skill がどう読み込まれたかを書かない測定は、2つの条件を混ぜる」 |
| [thkt/dotclaude#748](https://github.com/thkt/dotclaude/pull/748)（2026-09-24、opus、reference ごとに 10 run） | SKILL.md から参照される reference を、reviewer agent が自分から Read した run は 0/30 だった | Level 7 の「読み込まれなかった skill は全ての point で 0 点になる。この 0 を平均に埋もれさせない」 |
| 同上 | reference を空にした arm でも、元のままの arm でも、仕込んだ欠陥を全て検出した（空にした arm で 15/15）。arm の間で effect に差が出なかった | 判定の規則の「effect が変わらない」。ただし両方の arm が全て検出しており、天井に達した課題では effect の差を測れない。#781 はこれを最初の前提として確かめる |

## 照合の手順

1. 記事の本文を `xr article https://x.com/sermakarevich/status/2112198653469241856` で取得する。
2. 引用の arXiv ID をカンマで区切り、`https://export.arxiv.org/api/query?id_list=<ID>&max_results=100` を開く。版を指定するときは ID の後ろに `v1` などを付ける。
3. 各 entry の title・著者・summary を、記事の引用と数値に照らす。
4. arXiv 以外の引用は `scout fetch <URL>` で本文を取得して照らす。Vercel のページは既定の抽出で表が欠けるので、`--raw` を付ける。
5. superpowers の語数は、`gh api repos/obra/superpowers/contents/<path>?ref=<sha>` で raw の内容を取得し、`wc -w` で数える。
6. model による skill の呼び出しは、transcript の JSONL から `"name":"Skill","input":{"skill":"<name>"` を探し、tool_use の id で重複を除いて数える。
7. 人が打った呼び出しは、`<command-name>/<name></command-name>` で別に数える。
8. agent の frontmatter による読み込みは、`Base directory for this skill: <~/.claude の絶対パス>/skills/<name>` を含む transcript で数える。6〜8 のどれにも現れない skill を未使用とする。

## 見直す条件

- arXiv の新しい版で abstract の数値が変わったとき。SWE-Skills-Bench と Rethinking は記事の公開後に改訂されている
- thkt/dotclaude#781 の pilot で、rules の要素の effect を測った結果が出たとき
- Claude Code の skill の読み込み方（listing の常駐、自動選択、agent の preload）が変わったとき
