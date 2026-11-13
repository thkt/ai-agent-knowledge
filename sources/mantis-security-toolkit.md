# Google「Mantis」の考え方と根拠（2026-11）

AI coding agent に脆弱性を探させ、再現させ、修正させる Mantis の考え方を、根拠と照合した結果をまとめる。自分たちが採用した判断ではない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [google/mantis](https://github.com/google/mantis) |
| 著者 | Google。README は "This is not an officially supported Google product." と書く |
| 公開日 | リポジトリの作成は 2026-06-15。release と tag は無い |
| 照合日 | 2026-11-13 |
| 照合した版 | `main` の `729d316`（2026-11-12） |
| 照合した範囲 | README と README_AGENTS.md の主張、skill の定義、深刻度の較正規則、`reference/evals/` の README とファイル一覧、依存パッケージ、手元のハーネスとの対応 |
| この資料を根拠にした Issue | なし |

## 中心の主張

Mantis の中心の主張は次のとおり。**LLM が出した脆弱性の指摘は、本番での成立性の確認、再現、深刻度の較正を通すまで信用しない。**

README は、較正の目的を、LLM による深刻度の水増しを抑えることだとする（"combat LLM inflation of severity"）。人に届けるのは最も重大なリスクに絞り、数千件の "critical" を並べないことを目指す。

README は、すべての指摘を security の専門家が手で確かめるよう求める。自動で再現できなかったことは偽陽性の証明にならない。再現できたことも、あらゆる文脈で悪用できる証明にならない。

## 段の構成

README の Overview が挙げる段は次の順である。各段は `skills/` の skill に当たる。

1. git 履歴から過去の脆弱性を抽出する（`mantis-history`）
2. コードの索引と要約を作る（`mantis-structural-index`、`mantis-summarize`）
3. threat model を作る（`mantis-threat-model`）
4. 調べる仮説を立てる（`mantis-plan`）
5. 仮説ごとにコードを調べ、指摘を出す（`mantis-researcher`）
6. 重複を除く（`mantis-dedupe`）
7. 偽陽性を除き、本番で成立するかを確かめる（`mantis-review`、`mantis-critic`）
8. 再現する（`mantis-reproduce`）
9. 複数の指摘をつなげた攻撃経路を組む（`mantis-chain`）
10. 修正し、修正が効いたかを敵対的なループで確かめる（`mantis-patch`）
11. 規則に沿って深刻度を較正する（`mantis-calibrate`）
12. 1周で学んだことを抽出する（`mantis-reflect`）

開発中は、蓄積した知識をもとに `mantis-advise` が助言する。

## 考え方と根拠

| 考え方 | 資料が挙げる根拠 | 照合の結果 |
| --- | --- | --- |
| 深刻度の上限は、攻撃者が元の立場から新たに得る能力（marginal capability）で決める | 較正規則の Core Principle（[calibration_rules.md:13-20](https://github.com/google/mantis/blob/729d316/skills/mantis-calibrate/references/calibration_rules.md?plain=1#L13-L20)）と 27 の規則 | 規則は 27 あり、内訳は LOW 上限が 11、HIGH 上限が 8、MEDIUM 上限が 8 で一致する。規則で水増しがどれだけ減るかは未照合。`reference/evals/` には calibrate の段の benchmark（`run_eval.py --stage calibrate`）と 3 件の dataset（`calibrate_dataset.json`）がある。しかし実行結果は、README、README_AGENTS.md、`reference/evals/` のどこにも載っていない |
| 再現に失敗した指摘と、再現を試していない指摘は LOW を上限にする | 規則 01 `repro_failure`。対象のコードが変わって証拠が古くなった場合は除く（STALE_EVIDENCE） | 規則の本文に記述がある |
| 影響が、攻撃者が元から権限を持つ範囲（自分の account、tenant、container など）に収まるなら MEDIUM を上限にする | Category C の規則 2 `self_contained_blast` | 規則の本文に記述がある |
| 指摘が本番の release build（assertion を無効にした状態）でも成立するかを確かめ、debug 専用の機能や assertion で止まる経路を除く | `mantis-critic` の定義 | 定義に記述がある。`reference/evals/` に 4 件の dataset（`critic_dataset.json`）があるが、実行結果は載っていない |
| researcher に関数単位の検索 tool（`find_callers` など）を渡すと、精度が上がり token が減る | README_AGENTS.md は、arm ごとに 6 run の A/B で精度 75% 対 22%、token は約半分と書く（[README_AGENTS.md:520-524](https://github.com/google/mantis/blob/729d316/README_AGENTS.md?plain=1#L520-L524)） | 数値は evals README の round 2 の集計と一致する（[evals/README.md:193-195](https://github.com/google/mantis/blob/729d316/reference/evals/README.md?plain=1#L193-L195)。3/4 = 75%、2/9 = 22%、5.0M 対 10.7M token）。ただし round ごとに arm あたり 3 run で、75% と 22% は round 2 の 3 run だけの値である。"six-run-per-arm" は 2 round の合計を指す。対象は上流で修正が確かめられるまで伏せられており、照合できない。evals README 自身が、片方の arm だけが見つけた bug が両方の arm にあり、2つの toolset は補い合う関係だと書く（[同:195-199](https://github.com/google/mantis/blob/729d316/reference/evals/README.md?plain=1#L195-L199)） |
| 解説が公開されている脆弱なアプリ（Juice Shop など）は、LLM の段の評価に使わない。学習データに解説が含まれ、成績が水増しされる | [evals/README.md:51-62](https://github.com/google/mantis/blob/729d316/reference/evals/README.md?plain=1#L51-L62) | 記述がある。Juice Shop は、LLM を使わない surveyor の順位付けの評価にだけ使う。その評価も、閾値を Juice Shop で決めたため汎化の証拠にならないと evals README 自身が書く |
| 自動で生成したコードを実行するため、本番や社内網から切り離した環境でだけ使う | README の CAUTION（[README.md:3](https://github.com/google/mantis/blob/729d316/README.md?plain=1#L3)）。README_AGENTS.md は、Docker と、network を切った gVisor を使うよう勧める | 記述がある。README は "intended for demonstration purposes only" と書き、本番での利用を想定していない（[README.md:159](https://github.com/google/mantis/blob/729d316/README.md?plain=1#L159)） |
| PR を走査する GitHub Action を用意する | README の WARNING（[README.md:17](https://github.com/google/mantis/blob/729d316/README.md?plain=1#L17)） | end-to-end では実行されていないと README 自身が書き、merge の gate に使わないよう求めている |
| 特定の agent の基盤に依存しない | README_AGENTS.md は Gemini CLI、Antigravity CLI、Google ADK などで動くと書く | reference の harness は `google-adk`、`litellm`、`anthropic` に依存する（[requirements.in:1-7](https://github.com/google/mantis/blob/729d316/reference/requirements.in#L1-L7)）。`reference/README.md` の例の model は `gemini-3.8-flash` である。Claude のモデルで最後まで動くかは未照合 |

## 照合の結果

- **載っている計測**：README、README_AGENTS.md、`reference/evals/` に載っている計測は、次の2つだけである。
  - 構造 index の A/B。対象は1つで、round ごとに arm あたり 3 run
  - LLM を使わない surveyor の順位付けの benchmark。閾値を決めた Juice Shop で測っている
- **中心の主張に計測が無い**：`reference/evals/` には、reproduce（2 件）、critic（4 件）、calibrate（3 件）の dataset がある。しかし実行結果は上の3か所に載っておらず、再現と較正で指摘の質がどれだけ上がるかは分からない。
- **README_AGENTS.md と evals README の書き方の差**：
  - run 数：README_AGENTS.md は "six-run-per-arm" と書き、75% と 22% が 6 run の値に読める。evals README では round 2 の 3 run の値である
  - 対象の開示：evals README は分類（"widely shipped ~58k-line C driver subsystem"）だけを書き、具体的な対象を伏せる。README_AGENTS.md は "Linux kernel driver" まで書く
- **利害関係**：Google が公開しているが、公式の製品ではない。

## 手元のハーネスでの観測

[thkt/dotclaude](https://github.com/thkt/dotclaude) の `4b653ca`（`main`、2026-11-12）の security の reviewer と critic を、Mantis の較正規則と比べた。

| 観測 | 結果 | 対応する Mantis の要素 |
| --- | --- | --- |
| `reviewer-security` の報告の基準を読んだ | 悪用できるか不確実でも、具体的な修正案がある指摘は報告する（[reviewer-security.md:46](https://github.com/thkt/dotclaude/blob/4b653ca/agents/reviewers/reviewer-security.md?plain=1#L46)）。深刻度に low を付けない（[同:73](https://github.com/thkt/dotclaude/blob/4b653ca/agents/reviewers/reviewer-security.md?plain=1#L73)） | 較正規則は、根拠の弱い指摘の深刻度に上限をかける。再現していない指摘は LOW を上限にする（規則 01）。手元の基準とは向きが逆になる |
| reviewer が読み込む深刻度の表を読んだ | 指摘の確かさで深刻度を決める3段の表である（[use-context-reviewer-security/SKILL.md:41-50](https://github.com/thkt/dotclaude/blob/4b653ca/skills/use-context-reviewer-security/SKILL.md?plain=1#L41-L50)）。悪用が確実なら critical、脆弱性が明らかなら high、可能性があるなら medium とし、推測だけなら報告しない | Mantis は、再現できたかと、攻撃者が元の立場から得る能力で上限を決める。手元の表は再現を条件にしない。再現していない「可能性がある」指摘は、手元では medium、Mantis では LOW が上限になる |
| `critic-audit` の深刻度の確認を読んだ | 呼び出し元と、失敗を目にする人を追い、主張された影響範囲と比べる（[critic-audit.md:37](https://github.com/thkt/dotclaude/blob/4b653ca/agents/critics/critic-audit.md?plain=1#L37)）。狭ければ 1 段以上下げる（[同:80](https://github.com/thkt/dotclaude/blob/4b653ca/agents/critics/critic-audit.md?plain=1#L80)） | `self_contained_blast` など、影響範囲で上限を決める規則に近い。ただし、攻撃者の元の立場を基準にした上限の規則は手元に無い。どこまで下げるかは critic が影響範囲を追って判断する |
| 上の3つと `agents/_lib/finding-schema.md` を `reproduc` で検索した | 当たったのは [finding-schema.md:26](https://github.com/thkt/dotclaude/blob/4b653ca/agents/_lib/finding-schema.md?plain=1#L26) の1件だけである。この行は、Trigger を検証者が再現できる観測可能な条件として書くよう求める。再現を実行する段ではない | 再現の段（`mantis-reproduce`）と規則 01 に当たる確認は、検索した範囲に無い |

## 照合の手順

1. `scout repo-read google/mantis <path> --ref 729d316` で次のファイルを読む：`README.md`、`README_AGENTS.md`、`reference/README.md`、`reference/evals/README.md`、`reference/requirements.in`、`skills/mantis-calibrate/references/calibration_rules.md`、`skills/mantis-critic/SKILL.md`。
2. 規則の数を、`calibration_rules.md` で `` **`<名前>` ( `` の形の行を数えて確かめる。
3. 75% と 22% を、evals README の round 2 の件数から割り算で確かめる。
4. `reference/evals/` のファイルの一覧を `scout repo-tree google/mantis -p reference/evals --ref 729d316` で取り、実行結果のファイルが無いことを確かめる。
5. dataset の件数を、各 `*_dataset.json` の `"case_id"` の数で数える。
6. release と tag の有無を、`gh api repos/google/mantis/releases` と `gh api repos/google/mantis/tags` で確かめる。
7. 手元の定義を dotclaude の `4b653ca` で読む。
8. `ugrep -rn -i reproduc agents/reviewers/reviewer-security.md agents/critics/critic-audit.md skills/use-context-reviewer-security agents/_lib/finding-schema.md` で、再現への言及を探す。

## 見直す条件

- calibrate か critic の段の benchmark の結果が公開されたとき
- 構造 index の A/B の対象が公開され、追試できるようになったとき
- release が出たとき、または README から "demonstration purposes only" や PR scan の WARNING が外れたとき
- `reviewer-security` か `critic-audit` の深刻度の基準が変わったとき
