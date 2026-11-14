# Thoughtworks「Sensible Defaults」の考え方と根拠（2026-11）

Sensible Default は、それを覆す文脈が無い限り採用するプラクティスを指す、Thoughtworks の用語である。Martin Fowler の用語解説と、そこから辿れる Thoughtworks の公開資料5本を照合した。個別の defaults の一覧は公開ページから取得できず、資料の中で名前が挙がったものだけを拾った。自分たちが採用した判断ではない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Sensible Default](https://martinfowler.com/bliki/SensibleDefault.html)（以下「bliki」）、[Sensible defaults](https://www.thoughtworks.com/insights/topic/sensible-defaults)（以下「playbook ページ」）、[Starting out with sensible default practices](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/sensible-defaults)（以下「podcast 2022」）、[Sensible defaults: A way to think about our technology practices](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/sensible-defaults-way-think-technology-practices)（以下「podcast 2024」）、[Better technology decision-making is underpinned by 'sensible defaults'](https://www.thoughtworks.com/insights/blog/technology-strategy/better-tech-decision-making-underpinned-by-sensible-defaults)（以下「blog」）、[Engineering practices to accelerate your data product delivery](https://www.thoughtworks.com/insights/e-books/modern-data-engineering-playbook/engineering-practices-to-accelerate-your-data-product-delivery)（以下「e-book」） |
| 著者・発行元 | bliki は Martin Fowler。playbook ページは Thoughtworks で、著者の記載が無い。podcast 2022 は Mike Mason、Neal Ford、Kief Morris、Birgitta Böckeler。podcast 2024 は Ken Mugrage、Rebecca Parsons、Kief Morris、Brandon Cook。blog は Yewande Ige。e-book は David Tan と Mitchell Lisle |
| 公開日 | bliki は 2026-09-29。podcast 2022 は 2022-07-14。blog は 2024-09-05。playbook ページ、podcast 2024、e-book は、取得した本文から日付を読み取れなかった |
| 照合した版 | 2026-11-14 に `scout fetch` で取得した版。版の識別子は無い |
| 照合日 | 2026-11-14 |
| 照合した範囲 | bliki の本文。playbook ページの本文（静的 HTML と、ブラウザ描画後の両方）。podcast 2本の transcript の全文。blog の本文。e-book の当該の章 |
| この資料を根拠にした Issue | なし |

## 中心の主張

**default は出発点であり、外すことはできるが、外す側が理由を説明する。** bliki は Evan Bottcher の言葉を引く。「環境に厳しい制約が無ければ期待される行動が sensible default である。これらを実践するか、より良いやり方をせよ。別のやり方を選んだなら、理由を説明できるようにしておけ」。

bliki は、この語を「ベストプラクティス」と意図して対置したと書く。「ベストプラクティス」は、常に行うものという前提を含む。sensible default は、新しい文脈では再評価し、状況が変われば上書きするものである。

## 考え方と根拠

| 考え方 | 資料の記述 | 照合の結果 |
| --- | --- | --- |
| 外すなら、理由と補い方を説明する | Rebecca Parsons は、外す側に2つを求める。その状況に合わない理由と、default で得ていた利点をどう補うかである。「好きではない」は理由にならないとする | podcast 2024 に記述がある |
| 外す判断は、そのプラクティスを知ったうえで下す | Kief Morris は、経験の無い人の「時間の無駄だ」という拒否を、知識に基づく判断と区別する。外す前に、経験のある人を呼ぶことを勧める | podcast 2022 に記述がある |
| 自明なものは載せない | 「良い品質のソフトウェアを書く」のような、誰も反対しない文言は載せない。バージョン管理のように、業界で当然になったものも載せない。客先で議論になるものを選ぶ | podcast 2022 と podcast 2024 の両方に記述がある |
| 実際にやっていることを載せる | Kief は、IaC に理想を持つが、まだ誰もそう働いていないので default にしないと述べる。一覧は、文書になる前からチームにあった規範を書き起こして作った | podcast 2024 に記述がある。ほかの資料と食い違う点を「限界」に書いた |
| ツールではなくプラクティスを載せる | Neal Ford は、Technology Radar の作成者が技術の default 一覧を試みてやめたと述べる。技術は古くなるのが早く、粒度の議論が終わらなかったためである | podcast 2022 に記述がある。playbook ページも、ツールではなく原則に焦点を当てると書く |
| 名前ではなく、中身で書く | 「continuous integration」と「trunk-based development」の名前を使わず、「integrate frequently and continuously」と書いた。CI が「ブランチでビルドサーバーを回すこと」の意味に薄まったためである。PR やブランチを使っても、頻繁に統合していればよいとする | podcast 2024 に記述がある。podcast 2022 と e-book は、まだ「trunk-based development」の名前を使う |
| defaults は組で機能する | テストの安全網が無ければ continuous delivery はできず、テストか pair programming が無ければ毎日の統合は難しい。一覧は、互いに矛盾しない1組として選ぶ。Radar の Adopt は一覧であり、全部を同時に行うものではない | podcast 2022 と podcast 2024 に記述がある。blog は、職種をまたぐ例を挙げる |
| 議論するのは、外すときだけにする | Birgitta Böckeler は、既定を入れ替える例を挙げる。「1人が担当するのを既定にし、2人で組むたびに議論する」を、「2人で組むのを既定にし、外したいときだけ議論する」に変える。粒度はチーム単位とする | podcast 2022 に記述がある |
| 統制ではなく、事前の案内として使う | Neal は、統制の多くが逸脱を後から取り締まるのに対し、default は開始時に選ぶものだと述べる。統制の代わりではなく、その一部とする | podcast 2022 に記述がある |
| 適用の状況を調べ、使われないものは外す | 適用の有無をアンケートで集める。未適用のチームを赤く表示する用途には使わない。使われていない default は、理由を調べ、一覧から外すことも考える | podcast 2022 に記述がある |
| 当然になったものは一覧から外す | Birgitta は、原則を5、6個に絞り、当然になったものを外す運用を挙げる。議論を起こさなくなった default も同じ扱いを考えるとする | podcast 2022 に記述がある |
| 外したら、結果を見る | 外した後に何が起きるかを観察する。チームに、理由を理解して外す判断ができる人が1、2人要る | podcast 2022 に記述がある |
| 一覧を配るだけでは足りない | Birgitta は、1段落の説明を渡して「やれ」と言うだけでは足りないと述べる。Kief は、trunk-based development に要る feature toggle を例に、道具と習熟が要ると述べる | podcast 2022 に記述がある |
| 定期的に見直す | bliki は、アジャイル原則の12番目を引き、defaults 自体も定期的に再評価すると書く。playbook ページは、必要なら変更するか廃止すると書く | bliki と playbook ページに記述がある |
| 領域ごとの community が管理する | defaults は、global community of practice が合意で決める。Mural のボードと文書へのコメントで非同期に議論した。全会一致は求めない | podcast 2024 と blog に記述がある |
| GenAI には defaults を定めない | playbook ページは、GenAI の変化が速く、定めてもすぐ古くなると書く。代わりに、安全の境界、倫理面の考慮、透明な報告の仕組みを定める「sensible guardrails」を挙げる。Rebecca は、多数の事例でうまくいく方法がまだ無いと述べる | playbook ページと podcast 2024 に記述がある。Brandon Cook は、coding assistant を TDD や pairing を補強するものと見る |

## 名前が挙がった defaults

公開された一覧そのものではない。資料の中で default として名前が挙がったものを、領域ごとに並べた。

| 領域 | default | 出典 |
| --- | --- | --- |
| ソフトウェア開発 | pair programming、test-driven development | podcast 2022、podcast 2024、e-book |
| ソフトウェア開発 | integrate frequently and continuously。以前の名前は continuous integration と trunk-based development | podcast 2024。以前の名前は podcast 2022 と e-book |
| ソフトウェア開発 | fast automated build。podcast 2022 は、手元のビルドが最長でも1分ほどとする | podcast 2022、e-book |
| ソフトウェア開発 | automated deployment pipeline | podcast 2022、e-book |
| ソフトウェア開発 | continuous deployment。2022年の時点では、continuous delivery ではなくこちらが一覧にあった | podcast 2022 |
| ソフトウェア開発 | build security in、quality and debt effectively managed、build for production | e-book |
| インフラ | 最初からすべてをコードで定義する。先に作って後からコードに起こす進め方をしない | podcast 2022 |
| インフラ | インフラのコードの変更を、パイプラインとテストを通して届ける | podcast 2022 |
| インフラ | チームが、必要なインフラを自分で操作できる。待ち時間を作らないための権限であり、全責任を負う意味ではない | podcast 2022 |
| インフラ | インフラのコード、ツールのコード、ビルドスクリプトを含むすべてのコードで、品質を扱う | podcast 2022 |
| プロダクト | continuous customer research | blog |
| デザイン | regular user testing | blog |
| QA | value-based testing | blog |
| 計画 | risk register。書いて放置する一覧ではなく、運用の手順を伴うもの | podcast 2024 |

bliki が sensible default の例に挙げる3つは、「バージョン管理を使う」「UI ロジックとドメインロジックを分離する」「デプロイパイプラインを自動化する」である。podcast 2本は、バージョン管理を、当然になったので載せないものの例に挙げる。bliki の例は用語の説明であり、Thoughtworks の一覧からの引用とは書いていない。

e-book は、defaults の背後にある価値を3つ挙げる。fast feedback、simplicity、repeatability である。podcast 2022 で Birgitta も同じ3つを挙げ、価値は抽象的なので、出発点として具体的なプラクティスが要ると述べる。

## 限界

- **個別の defaults の一覧を取得できなかった。** podcast 2024 は一覧をサイトで公開したと述べるが、playbook ページの本文には、考え方の説明しか無かった。静的 HTML とブラウザ描画後の両方で確かめ、ページ内のリンクにも一覧への導線は無かった。
- **効果の測定値は無い。** どの資料も、根拠を Thoughtworks の現場経験に置く。blog の「市場投入が早まる」「競争で優位に立つ」は、数値を伴わない。e-book は、continuous delivery のプラクティスと業績の相関を外部の資料へのリンクで示すが、リンク先は照合していない。
- **因果は示されていない。** Kief は、pair programming をせずに成功している会社が多いと認める。「これらのプラクティスが開発者を上達させる」という Kief の主張に、Birgitta は「相関か因果か」と問い、答えは出ていない。
- **願望を載せるかどうかが、資料の間で食い違う。** playbook ページは「ときに願望を含む」と書く。podcast 2022 は、continuous deployment が一覧にあることを、願望を載せるべきかという未解決の議論として扱う。podcast 2024 の Kief は、願望ではなく実態だと述べる。
- **blog は営業資料を兼ねる。** 最後の節は、Thoughtworks が提供する支援の一覧である。
- **導入には前提がある。** blog は、技術的な成熟が要ると書く。podcast 2022 は、インフラでは TDD と CI の道具が業界に揃っていないと述べる。

## 手元の資料との関係（未採用）

この節は、照合者が手元の資料と突き合わせて組み立てた案である。どの設定にも入れていない。

| Sensible Defaults の考え方 | 手元の資料 |
| --- | --- |
| 外すなら、理由を説明する | [hadome.md](hadome.md) の `gate:allow <rule> reason: ...` は、例外に理由を書かせる。このリポジトリの `knowledge/_template.md` は「却下した案」と理由の欄を持つ。default で得ていた利点をどう補うかを書く欄は、`knowledge/_template.md` に無い |
| 自明なものは載せない | `~/.claude/rules/conventions/MARKDOWN.md` の「Removing duplicate instructions」は、システムプロンプトと重複する指示を削除できるとする。モデルが指示無しで行うことを rules に書かない、という同じ選び方である |
| 当然になったものは一覧から外す | 同じ節は、削除の可否をモデルの更新のたびに確かめ直すとする。README の「既存ページを見直す」と各ページの「見直す条件」は、知見の側の見直しを担う。`~/.claude/rules/core/BOUNDARIES.md` の Overeagerness は、既存の構造を外す前に DR の Reassessment Triggers を読むとする |
| GenAI には defaults ではなく guardrails を置く | hadome は、守らせる境界を機械で検査する。rules は、外せる既定を文章で渡す。playbook ページの区別に当てはめると、前者が guardrail、後者が default にあたる |

Sensible Defaults は、人のチームのプラクティスを対象にする。agent に渡す rules に当てはめる根拠は、どの資料にも無い。

## 照合の結果

- **照合した主張：** 「考え方と根拠」「名前が挙がった defaults」「限界」の各行は、右の列に挙げた資料の本文に記述がある。
- **読めなかったもの：** 個別の defaults の一覧。playbook ページのスライド表示の部品は、静的に取得できた文言だけを読んだ。
- **未照合：** playbook ページ、podcast 2024、e-book の公開日。bliki が触れる [Steve Bennett の投稿](https://stevebennett.co/posts/the-power-of-sensible-defaults)。podcast 2022 が触れる「Enduring techniques from 10 years at the Radar」。e-book がリンクする外部の資料。
- **音声との照合：** していない。podcast 2本は、ページに載った transcript だけを読んだ。

## 照合の手順

1. bliki を取得する：`scout fetch https://martinfowler.com/bliki/SensibleDefault.html`。公開日は `--raw` を付けた出力にある。
2. playbook ページを取得する：`scout fetch --raw https://www.thoughtworks.com/insights/topic/sensible-defaults`。`--raw` を外すと、「The sensible defaults playbook」「Not a monolith, but a map」「A note on GenAI」の節が落ちる。
3. playbook ページをブラウザ描画でも取得する：`scout fetch --js <同じ URL>`。sandbox の中では Chromium が起動しないため、sandbox を外して実行した。一覧が無いことを確かめる。
4. 関連する資料を探す：`scout search` は `BRAVE_SEARCH_API_KEY` が無く使えなかったため、Claude Code の WebSearch で「Thoughtworks sensible defaults」を検索した。
5. podcast 2本、blog、e-book の章を `scout fetch` で取得し、transcript と本文を読む。podcast 2022 の公開日は `--raw` を付けた出力にある。

## 見直す条件

- Thoughtworks が個別の defaults の一覧を、取得できる形で公開したとき。
- Thoughtworks が GenAI の defaults か guardrails を公開したとき。
- 手元の rules か `knowledge/_template.md` に、「外すときは理由と補い方を書く」を試したとき。
