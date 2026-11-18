# Google「Elephant-Goldfish Model（EGM）」の考え方と根拠（2026-11）

設計文書を書くセッションと、会話の履歴を持たないセッションを分けてレビューする EGM の考え方を、根拠と照合した結果をまとめる。自分たちが採用した判断ではない。

## TL;DR

- 設計文書は、会話の履歴を持たない新しいセッションが、文書だけを読んで設計を説明できるまで直す。
- 書くセッション（Elephant）と読むセッション（Goldfish）を分け、文書が溜まった文脈に頼っていないかを確かめる。
- エッセイの数値はすべて著者の経験の値で、計測方法は書かれていない。
- plugin の試験が確かめるのは手順の実行で、レビューの見落としが減ったかは測っていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [google/elephant-goldfish-model](https://github.com/google/elephant-goldfish-model)（plugin）と、その元になったエッセイ [Elephants, Goldfish and the New Golden Age of Software Engineering](https://medium.com/@drensin/elephants-goldfish-and-the-new-golden-age-of-software-engineering-c33641a48874) |
| 著者 | Dave Rensin とそのチーム（Google）。README は "This is not an officially supported Google product." と書く |
| 公開日 | エッセイは 2026-04-27（2026-10-21 に plugin 公開の追記）。plugin は v0.1.0 が 2026-10-27、v0.1.1 が 2026-11-06 |
| 照合日 | 2026-11-13 |
| 照合した版 | plugin は `main` の `96e9d6f`（2026-11-12） |
| 照合した範囲 | エッセイの本文、plugin の skill と reviewer の定義、TESTING.md の実行記録、Issue の状態、手元のハーネスとの対応 |
| この資料を根拠にした Issue | なし |

## 中心の主張

EGM の中心の主張は次のとおり。**設計文書は、会話の履歴を持たない新しいセッション（Goldfish）が、文書だけを読んで設計を説明できるまで直す。**

- Elephant：要件の議論、却下した案、細かな決定をすべて持つセッション。設計文書を書く
- Goldfish：履歴を持たない新しいセッション。目の前に置かれたものしか知らない

エッセイは、設計文書の出来が Elephant のセッションに溜まった文脈に頼っていないかを、Goldfish で確かめるとする。手順は "Feed the Elephant; test it against the Goldfish." の一文にまとめられている。

## レビューの段

plugin の `egm-review` は 5 つの stage を順に回す（[egm-review/SKILL.md:287-303](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/skills/egm-review/SKILL.md?plain=1#L287-L303)）。stage の番号は 5 から始まる。1〜4 が何を指すかは照合していない。

| Stage | 名前 | reviewer | 読ませる範囲 |
| --- | --- | --- | --- |
| 5 | Initial Zoom Out | `egm-zoom-out`、`egm-inventive-catalyst` | 文書の全体と参照先 |
| 6 | Comprehension Test | `egm-comprehension` | 文書だけ |
| 7 | Critic Review | `egm-design`、`egm-structural`、`egm-style` | 機能の節と参照先 |
| 8 | Intermediate Zoom Out | `egm-zoom-out` | 文書の全体と参照先 |
| 9 | Implementation Readiness Test | `egm-implementation-readiness`、`egm-codebase-details`、`egm-style` | 詳細の節と参照先 |

## 考え方と根拠

| 考え方 | 資料が挙げる根拠 | 照合の結果 |
| --- | --- | --- |
| コードを書く前に、AI と議論して設計を固める。最初の案は AI に出させ、理解の誤りを見つける | エッセイ Phase 1 の Step 2〜4。根拠は著者のチームの経験 | エッセイに記述がある。人が先に案を出した場合との比較は示されていない |
| 文書だけで設計を説明できるかを、新しいセッションで確かめる（Comprehension Test） | エッセイ Phase 3 の Step 5 | 読ませる範囲がエッセイと plugin で違う。エッセイは "this document and the files it references" を読ませる。plugin の Stage 6 は参照先を渡さず、文書だけを読ませる（[egm-review/SKILL.md:118-119](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/skills/egm-review/SKILL.md?plain=1#L118-L119)）。plugin の reviewer は最初に文書を自分の言葉で言い換え、言い換えられない節を Blocker とする（[comprehension.md:37](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/agents/comprehension.md?plain=1#L37)） |
| 新しいセッションの reviewer に、見落とし、誤った前提、端のケースを挙げさせる（Critic Review） | エッセイ Step 6。提案の約 30% が "highly valuable" | 30% は、著者が "Anecdotally" と断った経験の値である。計測ではない |
| 実装に必要な情報が文書に揃っているかを、新しいセッションに問う（Implementation Readiness） | エッセイ Step 7 | エッセイに記述がある。効果の計測は示されていない |
| Goldfish を、会話の履歴に加えて、ホストが注入する指示ファイルからも切り離す | TESTING.md の Claude Code の probe。reviewer が自分の persona だけを読み、親と project の `CLAUDE.md` を見ないことを確かめる（[TESTING.md:97-101](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/TESTING.md?plain=1#L97-L101)） | Claude Code 2.1.308 では isolation probe が通った（[TESTING.md:146](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/TESTING.md?plain=1#L146)）。Antigravity CLI と Codex では、指示ファイルの除外を試験していない（[TESTING.md:145-148](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/TESTING.md?plain=1#L145-L148)）。[#237](https://github.com/google/elephant-goldfish-model/issues/237)（2026-11-13 時点で OPEN）は、Codex と Antigravity の reviewer に `AGENTS.md` などが注入されると報告している |
| 既存のコードベースでは、ディレクトリごとの README を先に作る。AI が書いた末端のディレクトリの README は約 50% 誤るので、担当者が 5〜10 分かけて直す | エッセイの既存コードベースの節 | 50% と 5〜10 分は、計測方法が書かれていない。plugin では `egm-bootstrap` がこの手順に当たる。`egm-bootstrap` の本文（23.2 KB）は照合していない |
| 設計文書と現在のコードの食い違い（drift）を報告する | `egm-drift-report` の定義 | 文書にあってコードに無い機能やフィールド、計画と違う設計を drift とする（[egm-drift-report/SKILL.md:34-40](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/skills/egm-drift-report/SKILL.md?plain=1#L34-L40)）。文書から参照されないロジックも drift とする（[同:60-63](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/skills/egm-drift-report/SKILL.md?plain=1#L60-L63)） |

## 照合の結果

- **数値**：エッセイの数値はすべて著者の経験の値で、計測方法は書かれていない。対象は次の3つである。
  - Critic Review の提案のうち約 30% が有益（著者が "Anecdotally" と書く）
  - AI が書いた末端の README は約 50% 誤る
  - 新しいメンバーが書いた設計文書は、たいてい 70〜80% 正しい
- **plugin の試験**：TESTING.md が確かめるのは、インストール、reviewer の隔離、stage の順序、必要なファイルを読んだかである。レビューで見落としが減ったかは測っていない。TESTING.md は、試験用の設計への指摘を "design findings, not workflow execution failures" として、実行の失敗と分けて扱う。
- **エッセイと plugin の差**：
  - Comprehension Test で読ませる範囲（上の表の2行目）
  - plugin は、エッセイに無い Zoom Out の stage（5 と 8）を足している
- **利害関係**：エッセイの著者のチームが plugin を公開している。エッセイは 2026-10-21 に plugin の紹介を追記した。

## 手元のハーネスでの観測

[thkt/dotclaude](https://github.com/thkt/dotclaude) の `4b653ca`（`main`、2026-11-12）で、EGM の要素に対応するものを探した。

| 観測 | 結果 | 対応する EGM の要素 |
| --- | --- | --- |
| `critic-design` の frontmatter を読んだ | `omitClaudeMd: true`（[critic-design.md:6](https://github.com/thkt/dotclaude/blob/4b653ca/agents/critics/critic-design.md?plain=1#L6)）で、`CLAUDE.md` を読まずに起動する。subagent なので親の会話も受け取らない。一方で `memory: project`（[同:8](https://github.com/thkt/dotclaude/blob/4b653ca/agents/critics/critic-design.md?plain=1#L8)）があり、run をまたぐ記憶を `.claude/agent-memory/critic-design/` に持つ | Goldfish の隔離。会話と `CLAUDE.md` からの切り離しは TESTING.md の Claude Code の probe と同じ条件になる。run をまたぐ記憶を持つ点は、"zero memory" の Goldfish と違う |
| `agents/critics/` と、`skills/` の `think`、`qualify`、`issue` を、`paraphrase`、`comprehen`、`restate`、`explain back`、`cold read`、`in your own words`、`言い換え` で検索した | 当たったのは `skills/issue/references/prose-review.md:21` とそのテストだけである。この行は、リンク先の文書を本文に書き写さないよう求めるもので、言い換えによる理解の確認ではない。`critic-design` の観点表（[critic-design.md:27-40](https://github.com/thkt/dotclaude/blob/4b653ca/agents/critics/critic-design.md?plain=1#L27-L40)）は、隠れた前提、隠れたコスト、失敗の仕方などを攻撃する | 検索した範囲に、Stage 6 の Comprehension Test に当たる手順は無い |
| `/adrift` の定義を読んだ | DR の Decision Outcome と現在のコードの drift を調べる（[adrift.js:3-4](https://github.com/thkt/dotclaude/blob/4b653ca/workflows/adrift.js#L3-L4)） | `egm-drift-report` は設計文書と README を対象にする。比べる文書の種類が違う |
| 質問の出し方の規則を比べた | 手元の `CLAUDE.md` は "Recommend first" と定める（[CLAUDE.md:26](https://github.com/thkt/dotclaude/blob/4b653ca/CLAUDE.md?plain=1#L26)）。Claude Code の AskUserQuestion の説明は、推奨する選択肢を先頭に置き "(Recommended)" を付けるよう書く | `egm-review` は、1回に1問だけ聞き、選択肢に "(Recommended)" を付けないと定める（[egm-review/SKILL.md:65-69](https://github.com/google/elephant-goldfish-model/blob/96e9d6f/skills/egm-review/SKILL.md?plain=1#L65-L69)）。手元の規則と逆になる |

## 照合の手順

1. エッセイの本文を `scout fetch <エッセイの URL>` で取得し、"Goldfish" と "Anecdotally" で検索する。
2. plugin の定義を `scout repo-read google/elephant-goldfish-model <path> --ref 96e9d6f` で読む。読んだのは `skills/egm-review/SKILL.md`、`agents/comprehension.md`、`skills/egm-drift-report/SKILL.md`、`TESTING.md` である。
3. Issue の状態を `gh issue view 237 -R google/elephant-goldfish-model` で確かめる。
4. 手元の定義を dotclaude の `4b653ca` で読む。
5. `ugrep -rl -i -e paraphrase -e comprehen -e restate -e 'explain back' -e 'cold read' -e 'in your own words' -e 言い換え agents/critics skills/think skills/qualify skills/issue` で、言い換えの手順を探す。

## 見直す条件

- EGM が、レビューで見落としが減ったかを測った結果を公開したとき
- #237 が閉じ、Codex と Antigravity で指示ファイルの除外が試験されたとき
- `critic-design` か `/think` に、文書を言い換えさせる手順が入ったとき
