# Makarevych「Evals: how to know whether an AI system actually works」の考え方と根拠（2026-10）

この記事が示す LLM の評価（eval）の考え方と、その根拠を照合した結果をまとめる。自分たちが採用した判断ではない。

## TL;DR

- 採点器も検証の対象である。合格率を報告するときは、採点器と人の判定の一致度を並べる。
- eval は、入力、システム、採点器、数値の4つで定義する。
- arXiv の引用46件はすべて実在した。題が記事と違うものが2件あった。
- 本文の実例の数値は出どころを確かめられなかった。記事が紹介する製品との関係は、開示されていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Evals: how to know whether an AI system actually works](https://x.com/sermakarevich/status/2106453816757354947)（X の Article） |
| 著者 | Sergii Makarevych（@sermakarevich） |
| 公開日 | 2026-10-03 |
| 照合日 | 2026-11-13 |
| 照合した範囲 | 引用の実在、arXiv の abstract に載る数値、記事の公開後に出た関連製品 |
| この記事を根拠にした Issue | [thkt/cantle#169](https://github.com/thkt/cantle/issues/169)（Level 5 の考え方で比較の判定規則を決める） |

## 中心の主張

記事の中心の主張は次のとおり。**採点器（grader）も検証の対象である。製品の合格率を報告するときは、採点器と人の判定の一致度を必ず並べる。**

記事は eval を「入力、システム、採点器、数値」の4つで定義する。LLM は流暢で自信のある誤答を返すため、少数の出力を読んだだけでは品質を判断できない。そこで、正解つきの入力で繰り返し採点し、誤差の幅つきの数値で前の版と比べる。

記事によれば、AI チームが自分を欺く最も多い経路は、検証していない採点器が出す確信的な数値である。

## 考え方と根拠

| 考え方 | 記事が挙げる根拠 | 照合の結果 |
| --- | --- | --- |
| 正解つきの入力を最大の資産として扱い、調整用と報告用に分ける | Anthropic の「Demystifying evals for AI agents」が、実際の失敗から集めた 20〜50 件で始めるよう勧める | Anthropic の記事は実在する。20〜50 件の記述は未照合 |
| 採点器を作る前に出力を 100〜200 件読み、失敗の種類を数える | Husain の 2024 年・2025 年の記事。Shankar et al. 2024（採点しながら基準が変わる criteria drift） | Shankar et al. は arXiv:2404.12272 で実在する。Husain の記事は未照合 |
| 答えが決まるもの（ラベル、形式、必須の事実）は code の採点器で採点する。ただし code の採点器は意味を読めない | 著者のチュートリアルの例。全 code 検査に合格した返信は 11.7% で、捏造は1件も検出できなかった | チュートリアルの公開先が分からず、照合できない |
| LLM 採点器には、失敗の種類ごとに yes/no で問い、根拠の文を引用させ、判断材料の文脈を渡す | Cho et al. 2026（BINEVAL）。人との相関が G-Eval の 0.514 に対して 0.563 | arXiv:2606.27226 で実在する。abstract は G-Eval 以上の結果と書くが、数値は未照合 |
| 採点器は人のラベルで測る。素の一致率ではなく、kappa か「悪い出力を検出した率」で報告する | チュートリアルの例。一致率は 60% だが kappa は 0.15 | 例は照合できない。補強として、Norman et al. 2026（arXiv:2606.19544）の abstract が、素の一致率と kappa の差を MT-Bench で 33〜41pt と報告する。この論文は記事の別の行で引用されている |
| 採点器の偏り5種（提示順、長さ、自分の系統への好み、満足と成功の混同、採点器の交代）に対策を取る | Wang 2024、Dubois 2024、Panickssery 2024、Bodhwani 2026（GAUGE）、Yang 2026 | 5本とも実在する。GAUGE の「接戦の組で劣る側を選ぶ率 31%」は abstract と一致する。GAUGE の AUC 0.49 と 0.73 は abstract に無く、未照合 |
| 採点器を3つ程度並べ、平均ではなく幾何中央値で集約する | Verga 2024（PoLL）、Acharya 2026（RoPoLL） | 2本とも実在する。RoPoLL の abstract は、偏った採点器の混入に対して平均より強いことと、3台構成の結果を報告する。記事の「最悪の場合 540 倍正確」は未照合 |
| 「この主張は資料に支えられているか」の判定は、小さな検出器か型付きの採点器に任せる | HHEM、LettuceDetect、MiniCheck、SelfCheckGPT。JEV-as-a-Judge 2026 | 実在する。JEV の「費用は生成型の採点器の 0.36%」は abstract と一致する。カスケードの記述は食い違う。記事は「92.5% を保ち費用は約半分」とし、abstract（v6）は「GPT-6 より 0.9pt 正確で費用 41%」とする |
| 率には信頼区間を付け、2版の比較は同じ入力で対にする。検出できる最小の差を、比較の前に計算する | Miller 2024 | arXiv:2411.00640 で実在する。例（60 件で検出できる最小の差は約 12pt）はチュートリアル由来で照合できない |
| 検索と生成のように、段ごとに分けて採点する | RAGAS、ARES、SWE-Explore 2026 | 3本とも実在する |
| エージェントは最終状態と経路の両方で採点する。複数回実行し、全回成功の率（pass^k）と成功あたりの費用を測る | τ-bench、AgentLens 2026、Rabanser 2026、Bai 2026 | 4本とも実在する。Rabanser の abstract は「能力の向上に比べて信頼性の向上は小さい」と書き、記事の主張と一致する。記事の「6分の1」は未照合 |
| 最適化に使った採点器はシステムに攻略される。システムが一度も見ない採点器を残しておく | Wang et al.（Qwen）2026、Lu et al. 2026 | 2本とも実在する。「攻略された解が 28.6% から 0.6% に減った」は未照合 |
| ベンチマークのスコアは、モデル・プロンプト・評価ハーネス・版の組として記録する | Biderman 2024、Schaeffer 2023、GSM1k、Akhtar 2026、Zeng & Papailiopoulos 2026 | 実在する。「ベンチマークの約半数が飽和」と「2因子で変動の 90% 超を説明」は abstract と一致する。記事の「5ベンチマークで残りを約4点以内で予測」は未照合（abstract は、隠したスコアを 4.6 点以内で復元すると書く） |
| commit ごと、変更ごと、リリース後の3つの頻度で回す。一度だけ正解データで校正し、その範囲でだけ安い採点器を信頼する | Husain 2024、Bodhwani 2026（GAUGE） | GAUGE の abstract が、校正してから信頼する運用（calibrate-then-trust）を提案している。記事の主張と一致する |

## リーダーが求める4つの数値

記事は、次の4つが1枚に揃うまで、どの数値も意味を持たないとする。

1. 製品の合格率と信頼区間
2. 採点器と人の判定の一致度
3. 検出できる最小の差
4. 直前の変更で悪化したケースの割合

## 照合の結果

- **arXiv の引用 46 件**：すべて実在し、第一著者も一致した。記事とタイトルが違うのは次の2件である。
  - arXiv:2606.16576：実際の題は "Can Agents Infer Environment from Interaction? Evidence from Agentic Automata Learning"
  - arXiv:2605.22681：実際の題は "Scientific reasoning does not reliably translate into scientific forecasting in frontier AI"（CUSP を導入する論文）
- **arXiv 以外**：次の3件の実在を確かめた。
  - Anthropic の「Demystifying evals for AI agents」
  - Fiddler の TCO ページ（ベンダーの主張で実験は示されていない。記事自身もそう断っている）
  - OpenAI による promptfoo の買収（2026-03-09 発表）
- **照合していないもの**：HKUDS の ClawWork、Husain と Yan のブログ。
- **照合できないもの**：本文の実例の数値は、著者のチュートリアルから取られている。kappa 0.15、合格率 0.70、検出できる最小の差 12pt がその例である。チュートリアルの公開先が分からないため、照合できない。
- **利害関係**：記事は TypeSafe の Jev を好意的に紹介し、著者の知識ベースに Jev のチュートリアルがあると書く。TypeSafe との関係は開示されていない。

## 記事の公開後に変わった点

Google が 2026-11-11 に Gemini Decision（`gemini-decision-1`）と Decisions API を公開した。Jev と同じく、文章を生成せず選択肢の確率だけを返すモデルである。公開の事実は Google AI Studio の changelog で確かめた。

Jev との比較の数値は DataCamp の記事から取った。DataCamp は Google の発表としているが、Google のモデルページと Decisions API ガイドに比較表は無く、一次資料では確かめていない。

| 項目 | Gemini Decision 1 | Jev 1.14 |
| --- | --- | --- |
| 共通の7ベンチマークの勝ち数 | 5（多くは 1〜2pt 差） | 2（Mind2Web Multimodal、When2Call） |
| 校正誤差（Top-1 ECE、小さいほど良い） | 0.0250 | 0.0494 |
| 7評価の合計費用 | $0.88 | $0.68 |

Google の Decisions API ガイドは、確信度の閾値を自分のトラフィックのラベル付き例で決めるよう書いている。これは、ベンダーの数値は自分の課題に当てはまらないという記事の主張と一致する。

## 照合の手順

1. 記事の本文を、ブラウザで `https://x.com/sermakarevich/status/2106453816757354947` を開いて読む。CLI の `xr article <URL>` でも取得できる。
2. 引用の arXiv ID をカンマで区切り、`https://export.arxiv.org/api/query?id_list=<ID>&max_results=100` をブラウザか `curl` で開く。
3. 各 entry の title・著者・summary を、記事の引用と数値に照らす。
4. arXiv 以外の引用は、ウェブ検索で発行元のページを探す。
5. Gemini Decision は、ブラウザで `https://aistudio.google.com/docs/changelog` と `https://aistudio.google.com/docs/models/gemini-decision-1` を開いて読む。

## 見直す条件

- arXiv の新しい版で abstract の数値が変わったとき。JEV-as-a-Judge は v6 まで改訂されている。
- Google が Gemini Decision と Jev の比較を一次資料で公開したとき。
- 著者のチュートリアルが公開され、実例の数値を照合できるようになったとき。
