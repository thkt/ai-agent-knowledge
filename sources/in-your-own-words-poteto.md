# 「意図の理解がボトルネック」という発言と、pstack の /in-your-own-words（2026-11）

Ray Fernando が X に投稿した 26 秒のインタビュー動画で、Lauren Tan（poteto）は、エージェントに自分の言葉で言い直させる skill を紹介する。動画の音声と画面、pstack の skill の本文と履歴を照合した。手元の `~/.claude` に同じ形の skill を足したが、効果は測っていない。

## TL;DR

- 発言の中心は、エージェントは十分に有能になり、今のボトルネックは利用者の意図を理解することだ、というものである。そのため、作業を頼んで文脈を渡したら、まず自分の言葉で言い直させ、意図が伝わったかを確かめる。
- skill の実物は、`cursor/plugins` の `pstack/skills/in-your-own-words/SKILL.md` にあった。本文は「利用者の目的と、解こうとしている問題を、自分の言葉で言い直せ」の 1 文だけで、`disable-model-invocation: true` のため、人が `/in-your-own-words` と打ったときだけ動く。
- 測定は無い。言い直しで読み違いがどれだけ減るかの数字は、動画にも skill にも無い。
- 手元の `~/.claude` の決まりは、解釈があいまいなときにエージェントが自分で気づき、仮説を添えて聞く作りである。この skill は、エージェントが「わかったつもり」のときにも、人の側から言い直しを求められる点が違う。照合日は 2026-11-22。

| 項目 | 内容 |
| --- | --- |
| 出典 | [Ray Fernando の X の投稿](https://x.com/RayFernando1337/status/2123899583071662314)（動画つき）と、[cursor/plugins の pstack](https://github.com/cursor/plugins/tree/7a58391/pstack) |
| 著者・発行元 | 投稿とインタビューは Ray Fernando。話し手は Lauren Tan（poteto）で、pstack の作者である |
| 公開日 | 投稿は 2026-11-20。skill の追加は 2026-10-13 |
| 照合した版 | `cursor/plugins` の main `7a58391`。skill の履歴は commit `7d8d909`（追加）と `dbf4981`（書き換え） |
| 照合日 | 2026-11-22 |
| 照合した範囲 | 投稿の本文、動画（25.8 秒）の音声と画面、`pstack/skills/in-your-own-words/SKILL.md` の全文と、そのファイルの commit の履歴 |
| この資料を根拠にした PR | [thkt/dotclaude#938](https://github.com/thkt/dotclaude/pull/938) |

## 中心の主張

**エージェントが十分に有能になった今、詰まるのは能力ではなく、利用者の意図の理解である。** 動画の中で、Lauren Tan は pstack に入った skill のうち好きなものの 1 つとして「In Your Own Words」を挙げる。作業を頼み、文脈を渡したら、エージェントにそれを自分の言葉で言い直させ、意図を理解したかを確かめる、と話す。

## 考え方と根拠

| 考え方 | 原文の記述 | 照合の結果 |
| --- | --- | --- |
| 作業の前に、エージェントに意図を言い直させる | 動画の発言：作業を頼んで文脈を渡したら、自分の言葉で言い直させ、意図を理解したかを確かめる。画面には「Your agent / In my own words: … you want this done. Is that your intent?」という例の字幕が出る | skill がある。本文は 1 文で、言い直す対象を「利用者の目的と、解こうとしている問題」とする（[SKILL.md L7](https://github.com/cursor/plugins/blob/7a58391/pstack/skills/in-your-own-words/SKILL.md?plain=1#L7)） |
| 言い直すのは指示ではなく、目的と問題 | 動画では「意図」と言う | 追加の 17 分後の commit `dbf4981` で、「目的と問題を中心に言い直す」形に書き換えられた（commit の題は「reword /in-your-own-words around goals and the problem」） |
| 人が呼ぶ | 動画では、利用者がエージェントに言い直させると話す | `disable-model-invocation: true` で、モデルの側からは呼ばれない（[SKILL.md L4](https://github.com/cursor/plugins/blob/7a58391/pstack/skills/in-your-own-words/SKILL.md?plain=1#L4)） |
| 意図の理解がボトルネック | 動画の発言：エージェントはとても有能になり、意図の理解が今のボトルネックだと思う | 未照合。根拠は話し手の見立てで、数字は示されない |

## 手元の設定との差

| 観点 | pstack の /in-your-own-words | 手元の `~/.claude` |
| --- | --- | --- |
| 誰が始めるか | 人が `/in-your-own-words` と打つ | エージェントが、解釈があいまいだと気づいたときに聞く（`rules/core/PREFLIGHT.md` の Interpretation Clarity） |
| 何を返すか | 目的と問題の言い直し | 仮説を添えた質問 |
| 防げる読み違い | エージェントが「わかったつもり」の読み違いも、人が気づいて確かめられる | エージェントが気づいた読み違いだけ |

2026-11-22 の会話で、利用者の「じゃあ意味ないもんね。直そう」を、エージェントは「#924 の conflict を直す」と読んだ。利用者の意図は「`globalStore` を有効にして install を速くする」だった。このときはエージェントが聞き返したので食い違いが見つかった。エージェントが自信を持っていれば、聞き返さずに違う作業へ進んだと考えられる（照合者の推測）。手元に足した skill は、この読み違いを Bad と Good の例に使った。

## 限界

- **測定が無い。** 言い直しで読み違いがどれだけ減るか、手戻りがどれだけ減るかの数字は無い。
- **短い切り抜きである。** 動画は会場の廊下での 26 秒のインタビューで、ほかの 2 つの「好きな skill」は動画に入っていない。
- **話し手は当事者である。** Lauren Tan は pstack の作者である。
- **文字起こしは機械による。** whisper.cpp は pstack を「T-Stacks」と聞き取った。skill の名前は、リポジトリで確かめた。

## 照合の結果

- **照合した主張：** 「考え方と根拠」の 1〜3 行目は、skill の本文と commit の履歴で確かめた。4 行目は話し手の見立てで、照合できない。
- **読んでいないもの：** pstack のほかの skill と手引き。動画の全編（元のインタビューが長いかは分からない）。
- **範囲外：** pstack の Cursor 向けの導入方法。

## 照合の手順

1. 投稿を取得する：`xr tweet https://x.com/RayFernando1337/status/2123899583071662314`。本文と動画の URL が出る。
2. 動画を取得し、音声を 16 kHz の mono の WAV にする：`ffmpeg -i video.mp4 -ar 16000 -ac 1 audio.wav`。
3. 文字起こしする：`whisper-cli -ng -m ggml-large-v3-turbo-q5_0.bin -f audio.wav -l en -otxt`。sandbox の中では GPU の初期化に失敗したので、`-ng` で CPU を使った。
4. 画面を見る：`ffmpeg -i video.mp4 -vf "fps=1/4,scale=360:-1,tile=4x2" -frames:v 1 contact.png` で 4 秒ごとの画面を 1 枚に並べる。
5. skill を読む：`scout repo-read cursor/plugins pstack/skills/in-your-own-words/SKILL.md`。
6. 履歴を確かめる：`gh api 'repos/cursor/plugins/commits?path=pstack/skills/in-your-own-words/SKILL.md'`。

## 見直す条件

- pstack の skill の本文が変わったとき。
- 手元に足した `/in-your-own-words` を使い、読み違いが減ったかを判断できたとき。採用を決めたら `knowledge/` にページを書く。
