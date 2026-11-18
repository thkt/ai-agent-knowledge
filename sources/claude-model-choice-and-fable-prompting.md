# Claude のモデル選びと Fable 5.2 へのプロンプトの考え方と根拠（2026-11）

Anthropic が公開した2つの資料を照合した。1つは、作業ごとにどのモデルと effort を選ぶかの指針である。もう1つは、Claude Fable 5.2 に固有のプロンプトの書き方である。手元のハーネスに関わる指摘は、手元の設定と照らした。どれも自分たちが採用した判断ではない。

## TL;DR

- モデルが作業に足りないときは、モデルを上げる前に検証の手段を与え、次に effort を上げる。
- Fable 5.2 では、以前のモデル向けに書いた指示が逆効果になりうる。ガイドは症状ごとに対処を挙げる。
- ベンチマークの点数、費用、効果の割合は Anthropic の社内の試験の値で、再現していない。
- 手元の設定と照らしたが、どの設定も変えていない。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Which Claude model should you build with?](https://claude.dev/blog/which-claude-model-should-you-build-with/)（以下「選び方」）と、[Claude Fable 5.2へのプロンプト](https://platform.claude.com/docs/ja/build-with-claude/prompt-engineering/prompting-claude-fable-5-2)（以下「ガイド」） |
| 著者・発行元 | 選び方は Addy Osmani（Anthropic のブログ）。ガイドは Anthropic の公式ドキュメント |
| 公開日 | 選び方は 2026-11-09。ガイドは記載が無い |
| 照合した版 | 2026-11-13 に `scout fetch` で取得した版 |
| 照合日 | 2026-11-13。選び方は 2026-11-17 に取り直し、表の数値と公開日が変わっていないことを確かめた |
| 照合した範囲 | 2つの本文。手元の `~/.claude/agents` の `model:` の指定と、`~/.claude/rules/conventions/PROSE.md` の文体のルール |
| この資料を根拠にした Issue | なし |

## 中心の主張

**モデルを上げる前に、まず検証の手段を与え、次に effort を調整する。** 選び方は、モデルが作業に足りないときに試す順番を次のように書く。

1. 自分の作業を確かめる手段を与える。例：テスト、ビルド、エンドポイントを呼ぶスクリプト。
2. effort を1段上げる。
3. 1つ上のモデルに移り、小さいモデルで使ったより低い effort から始める。

ガイドは、Fable 5.2 が以前のモデルと違う振る舞いをする場面を、症状ごとに挙げる。以前のモデル向けに書いた指示が、Fable 5.2 では逆効果になりうるとする。

## 選び方：考え方と根拠

| 考え方 | 根拠 | 照合の結果 |
| --- | --- | --- |
| 多くの作業は Opus 5.5 の `medium` から始める | Claude Code の既定のモデルであり、token の単価が Fable 5.2 の半分である | 原文に記述がある。価格表では、Opus 5.5 は入力 $4、出力 $20、Fable 5.2 は入力 $8、出力 $40（いずれも100万 token あたり） |
| 範囲がはっきりした作業は Sonnet 5.5、大量の分類や抽出は Haiku 5.5、最も難しい作業と長時間の無人の実行は Fable 5.2 にする | 選び方の作業ごとの表 | 原文に記述がある。spec が明確でも、移行や性能の最適化は Opus 5.5 以上が要りうると注記する |
| `high` より上に effort を上げる前に、1つ上のモデルを低い effort で試す | FrontierCode 2.0 で、Fable 5.2 の `high` は Opus 5.5 の `max` とほぼ同じ点数（45.3% と 45.5%）で、1件あたりの費用は約半分だった | 原文に記述がある。数値は Anthropic の社内の実行で、再現していない |
| 安く回して、失敗だけを上の effort で回し直す | SWE-bench Pro の一部で、Opus 5.5 の `low` で回して失敗だけを `high` で回し直すと、約97%が1件あたり約 $0.17 で通った。すべて `high` だと 95.3% で $0.29 だった | 原文に記述がある。結果を自動で確かめられる場合に限る |
| メインの context を小さく保つ | 1ターンの cache の読み出しは、context が 150K token で約 $0.03、20K で約 $0.004 | 原文に記述がある。量の多い読み取りは、安い subagent に回すとする |
| subagent は小さいモデルに回す | 間違いに気づきやすい作業（ファイル探し、テストの実行、ログ読み）が向く。コードの編集と判断はメインのモデルに残す | 原文に記述がある。`opusplan` は、計画を Opus、編集を Sonnet に回すため、この指針と逆向きだと書く |
| 組み込みの Explore を安くする | 現在の Claude Code では、組み込みの Explore subagent はセッションのモデルで動く。安く回すには、`model: haiku` を付けた `Explore` という名前の subagent を自分で定義する | 原文に記述がある |
| 繰り返し走らせる agent の作業は Sonnet 5.5 の `medium` から始める | 作業ごとの表は、調査、レビュー、下書きを例に挙げ、範囲の決まった繰り返しの作業に大きいモデルはほとんど要らないと書く | 原文に記述がある。測定は示されない |
| モデルを比べるときは、難しい課題の費用も見る | 20問の検索のベンチマークで、2問が費用の43%を占めた | 原文に記述がある。ベンチマークの名前は書かれていない |
| モデルの切り替えは、作業の区切りで行う | prompt cache はモデルごとである。切り替えた最初のターンは、会話全体に cache の書き込みの料金がかかる。Claude Code では、先に `/compact` するか、短い計画を書いて新しいセッションを始める | 原文に記述がある |
| 下の階層のモデルへ移ると、それまでの思考は引き継がれない | モデルは、自分より上のモデルの思考の block を読まない。読めない block は API が落とし、要求は成功する。Fable 5.2 からほかの一般提供のモデルへ移ると、思考は落ちる | 原文に記述がある。読める組み合わせの一覧は、リンク先の Preserved thinking の文書にあり、読んでいない |
| 1つの context に収まらない作業は、上位のモデルが分け、安い worker に並列で渡す | 約2,200万 token のソースコードから欠陥を探す社内のベンチマークで、Opus 5.5 が Sonnet 5.5 の worker を最大25個まで率いる構成は、Opus 5.5 単独より約3分の1安く、点数はほぼ同じだった | 原文に記述がある。依存する手順が1本に連なる作業と、1つの context に収まる作業には使わないと書く。計画と統合の費用が、節約を上回るためである |
| advisor は効果が出る場面が限られる | 公式のコストのガイドが挙げる4つの組み合わせのどれも、実行役単独を明確には上回らなかった。Opus 5.5 の `high` に Fable 5.1 の advisor を付けた例は、1.7点の上積みに約2.1倍の費用がかかった | 原文に記述がある。上積みは実行ごとのぶれの範囲に近いと書く |
| 複数のモデルを混ぜるなら、最も厳しいモデルに合わせて要求を組む | 履歴を追記だけに保つ、`tool_choice: auto` と strict なツールを使う、content block を `type` で読む | 原文に記述がある。Fable 5.2 は、思考の block より前を変えた要求を 400 で拒否する |
| 自分の作業で測る | 実際の課題を抜き出し、結果の確認を書き、候補ごとに2〜3の effort で回し、費用と点数をグラフにする | 原文に記述がある。複数のモデルの構成は、どの単独のモデルの点よりも良いときだけ残すとする |

## ガイド：症状と対処

| 症状 | 対処 | ガイドが書く効果と代償 |
| --- | --- | --- |
| 作業が終わる前にターンを終える。`low` と `medium` で多い | システムプロンプトに「Finishing work」の文を足す。通常の障害は診断して乗り越え、意図的な障害（触らないと印を付けたファイルなど）はそのままにして報告させる | 続けられたのに未完了で終わったセッションが約半分になった。許可を求める終わり方の大部分は消えたが、「残りの作業を並べてから止まる」終わり方は残った。出力の token は約30%増えた |
| 既存のコードに足すだけで、重複を整理しない | 「You are the owner and long-term maintainer of this codebase, and its design is up to you.」と伝える | 重複を統合した実行が10%未満から約75%になった。出力の token は約70〜80%増え、頼んでいない変更（多くは新しいテストと文書）もした。「No drive-by refactors」と書いても、半分以上の実行で統合した |
| 長いターンの間、何も言わない | 読める出力の無いツール呼び出しが数回続いたら、"The user hasn't heard from you in a while: say in a few words what you're doing, then continue." をターン限りのシステムメッセージとして挟む | 最長の沈黙がおよそ半分になり、token や時間は目立って増えなかった |
| 必要以上に subagent を起動する | subagent のツールの説明に、費用への影響、委任する価値がある場合とない場合、指示の渡し方を書く | `high` の試験で、長めの説明は subagent の数をおよそ半分にした |
| 不要な承認を求める。日常の作業を控える | 古いメモを削除する。承認が要らない手順を列挙する。「never」の付いた指示を、範囲と例外を付けて書き直す | モデルは「never」を絶対の指示として扱う。「ファイルを決して削除しない」より「削除の前に承認を求める」と書く |
| 以前のモデル向けのハーネス | 常設の委任の指示、モデルに見せる token の予算、思考を減らす指示、厳しすぎる文体のルール、蓄積したルールとメモを見直す | 文体のルールについて、「1文につき1つの指示、20語以内」のような厳しい制限は文章をぎこちなくすると書く。思考を減らす指示は、`medium` 以上では思考を止めなかった |
| 長い作業の後の最終報告が分かりにくい | 「Communicating clearly」の文を足す。文脈を持たない読み手に向けて書き、要点を先に置き、ASD-STE100 に近い文体で書かせる | 最終報告の密度が下がり、セミコロンが減った |

ガイドは、effort の選び方も書く。能力の向上の大部分は `high` で得られ、`high` より上は効果がばらつく。`high` 以上では、挨拶への返答の前にも思考する。

## 手元の設定との照合

この節は、照合者が手元の設定を読んで組み立てたものである。どの設定も変えていない。

| 資料の指摘 | 手元の状態（2026-11-13） |
| --- | --- |
| 組み込みの Explore を Haiku の subagent で上書きする | `~/.claude/agents` に `name: Explore` の定義は無い。agent の `model:` は `opus` が21個、`sonnet` が7個で、`haiku` は無い |
| 厳しすぎる文体のルールを見直す | `PROSE.md` に「One claim per sentence」「No nested clauses」「One action per step line」がある。ガイドの例のような語数の上限は無い。Fable 5.2 で文章がぎこちなくなった場合に、最初に見直す候補になる |
| 「Finishing work」の文でも、残りを並べて止まる終わり方は消えない | [stop-hook-done-command.md](../knowledge/stop-hook-done-command.md) の Stop hook は、完了判定のコマンドが通るまで停止を差し止める。プロンプトで減らしきれない分を止める層になる |
| 所有者の一文で、重複を統合させる | [codebase-sweep-prompts.md](codebase-sweep-prompts.md) を元にした `sweep` の草案の `modules` の観点と組み合わせられる。ただし、出力の token の増加と、頼んでいない変更が起きる |
| 繰り返し走らせる agent の作業は Sonnet 5.5 の `medium` から始める | [thkt/feedback-loop#187](https://github.com/thkt/feedback-loop/issues/187) は、テストと lint の改善を人が始めずに回す状態を求める。agent を動かす場所もモデルも決めていない（2026-11-17）。[measure-tests-before-adding-practices.md](../knowledge/measure-tests-before-adding-practices.md) の、すり抜けた壊し方を埋める作業は、`claude-sonnet-5-5` で行った |
| 沈黙を破るリマインダー | このリポジトリで作業した Claude Code のセッションの harness は、ガイドの例とほぼ同じ文を、黙ったままの区間の後に挟んでいた |

## 照合の結果

- **照合した主張：** 「選び方」と「ガイド」の各行は、本文に記述がある。
- **再現していないもの：** ベンチマークの点数、費用、ガイドの効果の割合。どれも Anthropic の社内の試験の値である。
- **ガイドの公開日：** ページに記載が無い。

## 照合の手順

1. 2つの資料を取得する：`scout fetch https://claude.dev/blog/which-claude-model-should-you-build-with/` と `scout fetch https://platform.claude.com/docs/ja/build-with-claude/prompt-engineering/prompting-claude-fable-5-2`
2. 手元の agent の `model:` を数える：`ugrep -rh "^model:" ~/.claude/agents | sort | uniq -c`
3. Explore の上書きを探す：`ugrep -rl "^name: Explore" ~/.claude/agents`
4. 文体のルールを探す：`ugrep -rhoiE "one claim per sentence|one action per step|no nested clauses" ~/.claude/rules`

## 見直す条件

- 新しいモデルが出たとき。またはモデルの価格や既定の effort が変わったとき。
- Claude Code の組み込みの Explore が使うモデルが変わったとき。
- 手元の agent の `model:` か `PROSE.md` を変えたとき。「手元の設定との照合」が古くなる。
