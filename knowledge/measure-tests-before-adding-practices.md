# 検証手法は、先に壊した実装でテストを測り、生き残りから必要になった順に入れる

## 結論

プロジェクトに検証手法を入れるときは、決まった順番で積まず、次の順で進める。

1. 1つのモジュールを1箇所ずつ壊し、今あるテストが拾うかを測る。
2. 拾われなかった壊し方を、コードが決めている規則として読み、テストで埋める。agent に任せ、完了判定を付ける。
3. 測るのに時間がかかりすぎるとき、またはテストが書けないときに、判定を I/O から切り離す。
4. 入力の組み合わせが多い規則に PBT を、状態遷移と並行性の箇所に形式手法を当てる。

手順2の完了判定は、コードを実行して確かめる。agent や reviewer が「確かめた」と報告したときは、実行した結果なのか、コードを読んだ結果なのかを区別して読む。自分の仕組みについても、確認の段が実行と読解のどちらで行われているかを点検する。

確かめたのは1と2だけである。3と4は、1と2の結果から置いた順番で、試していない。この知見は検証だけで、`~/.claude` の設定には入れていない。

## 採用した理由

測ることを先頭に置くと、後の段で何を入れるかが、測った結果から決まる。

| できるようになったこと | 観察 |
| --- | --- |
| テストの不足を、具体的な一覧にする | 既存のテスト380件は、壊した実装45種のうち33種を拾い、12種を拾わなかった |
| 不変条件の一覧を、測った結果から得る | 拾われなかった12種は、コードが決めているのにテストが固定していない規則だった。例は、依存対応が集合外のテストを指すと止まること、収集コマンドの指定時に `.git` の下を数えないこと、出力がちょうど200文字のとき末尾を示さないこと |
| 切り離しが要るかを、費用から判断する | 12種は、既存と同じ CLI 経由の統合テストで全部拾えた。切り離さなくても検出はできた。費用は、全テストが4分41秒、関係する6ファイルに絞って73秒だった |
| agent が確かめられなかったことを、外から確かめる | agent は壊した実装を当てて走らせるコマンドを権限で拒否され、「実行して確認できていない」と報告した。完了判定が12種を走らせ、全部落ちることを確かめた |
| 「確かめた」が実行か読解かを、仕組みの側で見分ける | 手元の audit の検証役 critic-evidence は、コードを実行するツールを持たない。それでも判定の名前は `verified`、チェックの名前は `execution_trace` である。assert の enhancer-evidence は、build と test の結果を dynamic evidence と呼び、static findings と分けて扱う。区別は assert の語彙にあり、audit の語彙に無い |

## 却下した案

| 案 | 却下した理由 |
| --- | --- |
| 決まった順番で積む。切り離し、不変条件の一覧、テスト、確認の順 | 約30行の題材では、この順で書かせたテストの検出数は、いきなり書かせた場合を上回らず、費用は約4倍だった（[staged-test-writing-toy-trial.md](staged-test-writing-toy-trial.md)）。実プロジェクトでも、切り離しをせずに12種を全部埋められた |
| 全テストで測る | 全テストは4分41秒かかり、45種を直列で測ると3時間半になる。関係する6ファイルで測り、生き残った12種だけを残りの17ファイルで確かめ直した |
| 拾ったかの確認を agent に任せる | agent の確認用のコマンドは、権限で2回拒否された。agent は、コードを読んだ設計上の対応だけを報告した |
| 完了判定を exit 2 と stderr で返す | Claude Code は、stderr に `No such file or directory` があると差し止めない（[stop-hook-return-channels.md](stop-hook-return-channels.md)）。テストの出力はこの文字列を含みうるため、exit 0 と `decision: "block"` で返した |

## 検証

きっかけは、[形式手法の優先度についての投稿](https://x.com/k1_c_/status/2119279300805632276)である。投稿は、形式手法の前に、責務と境界を意識したアーキテクチャ、ふるまい駆動のテスト、不変条件と PBT を整えることを挙げる。その順番を、手元の repo の1モジュールで試した。

| 項目 | 内容 |
| --- | --- |
| 検証日 | 2026-11-14 |
| モデル | `claude-sonnet-5-5`（`--model sonnet`）。同じセッションの利用記録に `claude-haiku-5-5` も現れた |
| ツール | Claude Code 2.1.325、bun 1.4.4、Node.js v26.13.1、git、jq、rsync |
| 対象 | 手元の repo feedback-loop の `src/classification.ts`（204行）。commit は `4cc7995` |

### 再現手順

壊した実装は、次のスクリプトで作る。1行が1種で、`from` を `to` に置き換える。`from` は対象のファイルにちょうど1回だけ現れる。

```js
// make-mutants.mjs  usage: node make-mutants.mjs <classification.ts> <out dir>
import fs from "node:fs";
import path from "node:path";

export const MUTATIONS = [
  ["F01", "walk enters .git", "entry.name === '.git' ||", ""],
  ["F02", "walk enters node_modules", "entry.name === 'node_modules' ||", ""],
  ["F03", "walk enters hidden directories", "(excludeHidden && entry.isDirectory() && entry.name.startsWith('.'))", "false"],
  ["F04", "walk also drops hidden files", "excludeHidden && entry.isDirectory() && entry.name", "excludeHidden && entry.name"],
  ["F05", "walk returns non-regular entries", ": entry.isFile()", ": true"],
  ["E01", "untraversable path counts as existing", "    return false;", "    return true;"],
  ["E02", "dangling symlink counts as missing", "lstatSync(path, { throwIfNoEntry: false }) !== undefined", "statSync(path, { throwIfNoEntry: false }) !== undefined"],
  ["D01", "missing and existing swapped", "if (!entryExists(join(root, path))) {", "if (entryExists(join(root, path))) {"],
  ["D02", "unlisted and unmatched swapped", "} else if (isTestFile(path, config)) {", "} else if (!isTestFile(path, config)) {"],
  ["D03", "guidance for collect and builtin swapped", "(config.collect === undefined", "(config.collect !== undefined"],
  ["D04", "globbed-only side inverted", "paths: globbed.filter((path) => !inCollected.has(path)),", "paths: globbed.filter((path) => inCollected.has(path)),"],
  ["D05", "empty sides are printed", ".filter(({ paths }) => paths.length > 0)", ".filter(({ paths }) => paths.length >= 0)"],
  ["D06", "diagnostic paths unsorted", "JSON.stringify(paths.sort())", "JSON.stringify(paths)"],
  ["D07", "guidance dropped", "return guidance === undefined ? [line] : [line, guidance];", "return [line];"],
  ["D08", "paths on both sides are printed", "collected.filter((candidate) => !inGlobbed.has(candidate))", "collected"],
  ["O01", "shown output length halved", "const shownOutput = 200;", "const shownOutput = 100;"],
  ["O02", "tail shown at exactly the limit", "return stdout.length > shownOutput", "return stdout.length >= shownOutput"],
  ["O03", "tail shows the head again", "stdout.slice(-shownOutput)", "stdout.slice(0, shownOutput)"],
  ["P01", "control characters kept in reason", ".replaceAll(/\\p{Cc}+/gu, ' ')", ""],
  ["P02", "reason length unbounded", ".slice(0, shownOutput);", ";"],
  ["P03", "non-string-array output accepted", "assertStringArray(collected);", ""],
  ["B01", "builtin list keeps hidden paths", "!path.split('/').some((part) => part.startsWith('.')) &&", ""],
  ["B02", "builtin list misses _spec files", "(?:\\.test|_test|\\.spec|_spec)", "(?:\\.test|_test|\\.spec)"],
  ["B03", "builtin list misses .mts files", "js|jsx|ts|tsx|mjs|cjs|mts|cts", "js|jsx|ts|tsx|mjs|cjs|cts"],
  ["C01", "collected paths not checked as repo-relative", "collected.forEach(assertRepoRelativePath);", ""],
  ["C02", "duplicate collected paths accepted", "assert(new Set(collected).size === collected.length, '収集結果が重複しています');", ""],
  ["R01", "search prefix loses its trailing slash", "fixed.slice(0, fixed.lastIndexOf('/') + 1)", "fixed.slice(0, fixed.lastIndexOf('/'))"],
  ["R02", "nested search roots both walked", "!all.some((parent) => parent !== prefix && prefix.startsWith(parent))", "true"],
  ["R03", ".git search root allowed", "part === '.git' ||", ""],
  ["R04", "node_modules search root allowed", "part === 'node_modules' ||", ""],
  ["R05", "hidden search root allowed for builtin", "(collection.kind === 'builtin' && part.startsWith('.'))", "false"],
  ["R06", "missing search root accepted for builtin", "assert(found || collection.kind === 'command', ", "assert(true || collection.kind === 'command', "],
  ["R07", "hidden directories walked for builtin", "filesUnder(root, prefix, collection.kind === 'builtin')", "filesUnder(root, prefix, false)"],
  ["R08", "hidden directories skipped for collect", "filesUnder(root, prefix, collection.kind === 'builtin')", "filesUnder(root, prefix, true)"],
  ["X01", "expected set ignores testGlobs", "files.filter((path) => isTestFile(path, config)).sort()", "files.sort()"],
  ["X02", "set mismatch ignored", "if (JSON.stringify([...actual].sort()) !== JSON.stringify(expected)) {", "if (false) {"],
  ["X03", "set comparison depends on order", "JSON.stringify([...actual].sort())", "JSON.stringify(actual)"],
  ["A01", "duplicate assurance accepted", "assert(assurance.size === config.assurance.length, 'assurance指定が重複しています');", ""],
  ["A02", "assurance outside the set accepted", "[...assurance].every((path) => expected.includes(path)),", "true,"],
  ["A03", "one valid assurance is enough", "[...assurance].every(", "[...assurance].some("],
  ["A04", "dependency tests outside the set accepted", "config.dependencies.every((item) => item.tests.every((path) => expected.includes(path))),", "true,"],
  ["A05", "one valid dependency test is enough", "item.tests.every(", "item.tests.some("],
  ["Q01", "quality includes assurance tests", "quality: expected.filter((path) => !assurance.has(path)),", "quality: expected,"],
  ["Q02", "assurance list empty", "assurance: [...assurance],", "assurance: [],"],
  ["Q03", "all excludes assurance tests", "all: expected,", "all: expected.filter((path) => !assurance.has(path)),"],
];

if (process.argv[1] === new URL(import.meta.url).pathname) {
  const [srcPath, outDir] = process.argv.slice(2);
  const src = fs.readFileSync(srcPath, "utf8");
  fs.mkdirSync(outDir, { recursive: true });
  for (const [id, , from, to] of MUTATIONS) {
    if (src.split(from).length !== 2) throw new Error(`${id}: expected exactly one match`);
    fs.writeFileSync(path.join(outDir, `${id}.ts`), src.replace(from, to));
  }
  fs.writeFileSync(path.join(outDir, "M00.ts"), src); // unchanged control
}
```

1種を走らせるスクリプトは、次のとおり。repo を `.git` ごと複製し、対象のファイルを差し替え、バンドルを作り直してからテストを実行する。テストは生成したバンドル経由でコードに届くため、作り直しを省くと差し替えが反映されない。

```bash
#!/bin/bash
# run-mutant.sh  usage: run-mutant.sh <repo> <mutants dir> <results dir> <id> [test files...]
R="$(cd "$(dirname "$0")/.." && pwd)"
repo="$1"; mutants="$2"; results="$3"; id="$4"; shift 4
work="$R/work/$(basename "$results")-$id"
mkdir -p "$work" "$results"
rsync -a --delete "$repo/" "$work/"
cp "$mutants/$id.ts" "$work/src/classification.ts"
cd "$work" || exit 1
bun scripts/build.ts > "$results/$id.build" 2>&1 || { echo "build-failed" > "$results/$id.exit"; exit 0; }
start=$(date +%s)
bun test ${BAIL---bail=1} --timeout 60000 "$@" > "$results/$id.out" 2>&1
echo "$?" > "$results/$id.exit"
echo "$(( $(date +%s) - start ))" > "$results/$id.secs"
```

1. repo を作業用の場所に `git clone` し、`4cc7995` を checkout する。元の repo の `node_modules` へ symlink を張る。
2. `node make-mutants.mjs <repo>/src/classification.ts mutants` を実行する。
3. 45種と対照 `M00` を、関係する6ファイルで走らせる。`ls mutants | sed 's/\.ts$//' | xargs -P 4 -I{} bash run-mutant.sh <repo> mutants results-a {} ./tests/config.test.ts ./tests/standalone.test.ts ./tests/batch.test.ts ./tests/results.test.ts ./tests/cli.test.ts ./tests/entrypoints.test.ts`
4. `results-a/<id>.exit` を読む。`0` は拾われなかったことを、`1` は拾われたことを指す。`M00` が `0` であることを確かめる。
5. 拾われなかった種と `M00` を、残りの17ファイルで `BAIL=""` を付けて走らせる。各出力の `(fail)` の行を `M00` のものと比べ、`M00` に無い行があれば拾われたとする。
6. agent 用に repo をもう1つ `git clone` し、`node_modules` へ symlink を張る。
7. Stop hook を書く。hook は次の表の順に判定し、差し止めるときは exit 0 で `{"decision":"block","reason":"..."}` を出力する。

| 条件 | hook の扱い |
| --- | --- |
| `src/`、`plugins/`、`scripts/` に `git diff` がある | 差し止め、元に戻すコマンドを理由に書く |
| `tests/classification-gaps.test.ts` が無い | 差し止める |
| `M00` で `tests/classification-gaps.test.ts` が落ちる | 差し止め、落ちたテスト名を理由に書く |
| 拾われなかった12種のどれかで、そのファイルが通る | 差し止め、通った種の ID、説明、置き換えの前後を理由に書く |
| 12種すべてでそのファイルが落ちる | 止まるのを許す |
| 差し止めが6回を超えた | 止まるのを許し、残った種を記録する |

8. agent を実行する。`claude -p --model sonnet --settings <settings> --permission-mode acceptEdits --allowedTools "Bash(bun test:*),Bash(bun:*),Bash(git status:*),Bash(git diff:*)" --output-format stream-json --verbose "<prompt>" < /dev/null`
9. hook の記録、agent が足したファイル、出力の `result` を読む。
10. `~/.claude/agents/critics/critic-evidence.md` の `tools:` の行と Verdicts の表を読む。`~/.claude/agents/enhancers/enhancer-evidence.md` の入力の表から Dynamic evidence の行を読む。

手順8の prompt は、次の文の前に、拾われなかった12種を1行ずつ並べたものである。各行は、ID、説明、置き換えの前後からなる。

```text
Add tests to a new file tests/classification-gaps.test.ts so that each variant makes at least one test fail, while all tests pass on the current code. Follow the style of the existing tests (see tests/config.test.ts, tests/cli.test.ts, tests/entrypoints.test.ts and the helpers under tests/support). Assert on what a user of the tool observes (exit status, diagnostics, classification output), not on implementation details. Do not change anything under src/, plugins/ or scripts/. Run `bun test ./tests/classification-gaps.test.ts` to check your tests. If you conclude that a variant cannot be told apart from the current code by any test, say which one and why. A completion check runs when you finish and reports the variants that still pass.
```

### 結果

- 全テストは23ファイル380件で、280.74秒かかった。複製した環境では、そのうち10件が何も変えない状態で失敗した。10件は関係する6ファイルの外にあった。
- 関係する6ファイルは189件で、どれも通った。1ファイルずつ直列に走らせた合計は73秒だった。
- 手順3で、45種のうち33種が拾われた。
- 拾われなかった12種は、`A04`、`A05`、`B01`、`C01`、`E01`、`F01`、`F04`、`O02`、`P01`、`R03`、`R04`、`R08` である。手順5でも、12種のどれも `M00` に無い失敗を出さなかった。
- 拾われなかった12種は、次の4つにまとまった。

| まとまり | 種 |
| --- | --- |
| 依存対応の検証 | `A04`、`A05` |
| 収集コマンドの指定時の除外と探索 | `F01`、`R03`、`R04`、`R08` |
| 隠しファイルの扱い | `B01`、`F04` |
| 境界と診断の文面 | `O02`、`P01`、`E01`、`C01` |

- 手順8の agent は、21ターン、128.8秒、0.68ドルで終わった。189行のテストファイルを足し、テストは14件だった。`src/`、`plugins/`、`scripts/` は変更しなかった。
- Stop hook は1回だけ走り、差し止めなかった。`M00` ではテストファイルが4秒で通り、12種ではどれも落ちた。落ちたテストの名前は、種ごとの規則を述べていた。例は、`A04` に対する「依存対応が集合外のテストだけを指すと、分類検査は止まる」である。
- agent の Bash は、権限で2回拒否された。agent の最後の報告は、12種で落ちることを「実行して確認できていません」と書いた。
- 手順10で、`~/.claude`（commit `4207ed2e`）の定義を読んだ。
  - critic-evidence の `tools:` は `Read, LS, Bash(git:*), Bash(ugrep:*), Bash(bfs:*)` で、コードを実行するツールが無い。
  - 判定 `verified` の条件は、具体的な実行経路をたどれて、発火の条件を名指しできることである。たどる手段は、ファイルを読むことと検索である。
  - enhancer-evidence は、`build=pass, tests=fail (3 failed)` のような1行を Dynamic evidence として受け取り、static findings と突き合わせる。
- 確かめていないこと：
  - `~/.claude` のほかの agent、skill、workflow で、確認の段が実行と読解のどちらかは点検していない。読んだのは上の2ファイルだけである。
  - 測ったのは1モジュールで、agent の実行は1回だけである。
  - 45種は、判定の分岐、境界、除外の条件から手で選んだ。選び方に網羅性の根拠は無い。
  - agent には、拾われなかった種の置き換えの前後を見せた。足したテストが、その置き換えだけに合わせ込まれていないかは確かめていない。テストの中身は名前だけを読んだ。
  - 何も変えない状態で失敗した10件の原因。
  - 結論の手順3と4。切り離しをした場合に、測る時間がどれだけ縮むかは測っていない。
- 測定の途中で、複製から `.git` を除いたために、対照と19種が同じテストで落ちた。そのテストは、repo から checkout を作る。`.git` を含めて、対照と19種を走らせ直した。上の数字は、走らせ直した後のものである。

## 見直す条件

- 2つ目のモジュールか別の repo で測り、拾われなかった種を既存の形式のテストで埋められなかったとき。結論の手順3を前に出す根拠になる。
- 判定を I/O から切り離して、測る時間を比べたとき。
- mutation のツールを入れ、手で選んだ45種と結果を比べたとき。
- `~/.claude` の全体を、確認の段が実行か読解かで点検したとき。または、audit の判定に実行と読解の区別を入れたとき。
- Claude Code か bun のバージョンが上がり、Stop hook の入出力かテストの実行時間が変わったとき。
