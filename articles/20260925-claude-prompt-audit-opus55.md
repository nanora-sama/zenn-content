---
title: "/claude-api prompt-audit を動いている自律エージェントに当てたら、かなり削れた"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "claude", "llm", "prompt", "ai"]
published: false
---

> **TL;DR**
> - 自律エージェントに `/claude-api prompt-audit`：指摘 107 件、反映 63 件
> - 多かったのは事実のずれ（消えたパス・古い設定値・古いモデル名）
> - effort の上書きキーが旧 ID のままで、Opus 5.5 が `xhigh` 稼働
> - 禁止事項は外して比べてから削る。1 項目は外すと 3/3 → 0/3
> - JSON は `--json-schema` で受け取り。ルートは object 限定

---

X で「`/claude-api prompt-audit` と打てば、skills・CLAUDE.md・プロンプトを Opus 5.5 の公式ガイドで書き直してくれる」という投稿が流れてきた。

Raspberry Pi 5 で自律エージェントを動かしていて、Claude は Claude Code の CLI（`claude -p`）経由で呼んでいる。ルールファイル、エージェント定義、実行時に渡すプロンプトは数か月分積み上がっていて、書いた当時のモデル向けの言い回しがあちこちに残っている。書き直せるなら使いたいので、当ててみた。

## まず、エイリアスの実体を測る

監査の対象モデルを決める前に、`opus` が何に解決されるかを見た。

```bash
for m in opus sonnet; do
  claude -p --model $m --tools "" --output-format json "ok とだけ返して" \
    | python3 -c "import json,sys;print('$m ->', list(json.load(sys.stdin)['modelUsage']))"
done
# opus -> ['claude-opus-5-5']
# sonnet -> ['claude-sonnet-5']
```

ルールには「既定は Opus 5」とあったが、実体はもう Opus 5.5 だった。エイリアスで指定していると、モデルだけ勝手に新しくなってプロンプトは古いまま残る。監査の対象は Opus 5.5 にした。

コードが Messages API を直接呼んでいないことも確認した。thinking の無効化、forced `tool_choice`、prefill は Opus 5.5 で 400 になるが、Claude 経路には 0 件。API の破壊的変更は当たらないので、見るのはプロンプト本文だけでいい。

## prompt-audit の中身

`/claude-api prompt-audit` を打つと、スキルの `shared/prompt-audit.md` を読んでその手順で動く。

1. 対象範囲と対象モデルを決める（聞き返さず、前提として報告に書く）
2. プロンプトとして効いているものを棚卸しする（system prompt、ツール定義、SKILL.md、CLAUDE.md、リクエストを組み立てるコード）
3. 来歴を見る（`git blame` で「どの事故を防ぐために、どのモデルのときに足したか」）
4. パターン表で照合する（強すぎる口調、API 機能で置き換わった足場、過剰な指定、古いモデルの名残）
5. 報告（`file:line`・根拠・確度）と diff を出す
6. 挙動で検証する

5 分では終わらなかった。そのかわり手順書が丁寧で、「削除は仮説であって結論ではない」として、消した指示が本当に不要だったかを挙動で確かめるところまで書いてある。「消してはいけないもの」のリストもあって、文脈・理由・今も起きる失敗への禁止事項は残せと指示している。安心して任せられた。

量が多いので 4 区画に分けてサブエージェントに監査させ、確度が中以上の指摘だけ反映した。

| 区画 | 指摘 | 反映 |
|---|---|---|
| CLAUDE.md とルールファイル（6） | 21 | 10 |
| エージェント定義（6）と SKILL.md（10） | 23 | 9 |
| 実行時に `claude -p` へ渡すプロンプト（コード内） | 13 | 4 |
| 参照用のドキュメント（38） | 50 | 40 |

## 出てきたのは事実のずればかり

古いモデル向けの強い口調も直してはいる。ただ、数でも効き目でも上だったのは、書いてあることと実物の食い違いのほう。

- **一度も読み込まれていなかったルール**: 読み込み条件（`paths:`）が 4 月のリファクタで消えたファイルを指したまま。そのファイルを触るときに読ませたいルールが、以来一度もロードされていなかった
- **「除去済み」の課金 API キー**: ルールでは「課金経路は全ルートから除去」。実際は予備として 1 ルートに残っていた。ログが残る期間で成功 0・失敗 144。課金こそ出ていないが、失敗するだけの予備として夜間タスクの全滅に加わっていた
- **書き写した現在地**: スキルが進捗ファイルの「現在地」を写していて、本体と食い違っていた。写しは消して、本体だけを正にしてある
- **自分に一致して終わらない待機ループ**: `until ! pgrep -f "tidy-imports"` は実行中の bash のコマンド全文にも一致するので、永遠に終わらない

どれも grep と設定ファイルで実値と照合すれば分かる。書き方の問題ではないから、モデルが新しくなっても勝手には直らない。

口調も直した。実行時プロンプトの「必ずツールを使え」「推測で回答しないでください」は、何を根拠にするかと理由を書いた平叙文に。要求そのものは変えていない。

## effort の罠

いちばん効いたのは effort（思考量）。

Opus 5.5 の API 既定 effort は `medium`（Opus 5 は `high`）。同じ effort でも Opus 5 より多く考える、とガイドにある。手元の `~/.claude/settings.json` はこうだった。

```json
{
  "effortLevel": "xhigh",
  "modelSettings": {
    "claude-opus-5": { "effortLevel": "high" }
  }
}
```

`modelSettings` のキーは実モデル ID。`claude-opus-5` の上書きは `claude-opus-5-5` に効かないので、Opus 5.5 は全体設定の `xhigh` で回っていた。Opus 5.5 は `medium` で十分な結果が出ていたので、上書きを足した。

```json
"modelSettings": {
  "claude-opus-5":   { "effortLevel": "high" },
  "claude-opus-5-5": { "effortLevel": "medium" }
}
```

困ったのは確かめ方。`--output-format stream-json` の init イベントにも `--debug` のログにも effort は出ない。出るのはセッションの transcript（jsonl）だけ。

```bash
f=$(ls -t ~/.claude/projects/<プロジェクトのディレクトリ>/*.jsonl | head -1)
grep -oE '"effort":"[a-z]+"' "$f" | head -1
# "effort":"medium"
```

指定なしの 3 回は全部 `medium`、`--effort xhigh` の 3 回は `xhigh` と記録された。

もう 1 つ。**effort はエージェント単位では変えられない**。定型作業のサブエージェントだけ軽くしようとエージェント定義の frontmatter に `effort: low` を書いたが、transcript は `xhigh`。`--agents` の JSON で渡しても同じ。効くのはこの 3 つ。

| 設定 | 効くか |
|---|---|
| `settings.json` の `effortLevel`（全体） | 効く |
| `modelSettings."<実モデル ID>".effortLevel` | 効く |
| CLI の `--effort` | 効く |
| エージェント定義の `effort:` / `--agents` JSON | 効かない（2.1.282 で実測） |

Sonnet 5 は `xhigh` のまま。下げるとコードを書くサブエージェントまで下がるし、ガイドもコーディングには `xhigh` を勧めている。簡単な課題（誤字を 1 語だけ返す）なら `medium` でも `xhigh` でも出力は 4 トークンで、定型作業のコストはほぼ増えない。

`CLAUDE_CODE_MAX_THINKING_TOKENS=16000` も消した。予算形式の指定は Opus 5.5 と Sonnet 5 では使えず、思考量を調整できるのは effort だけ。

## 禁止事項は測ってから消す

CLAUDE.md に「報告する前に確かめる 13 項目」がある。全部、実際の事故から作ったもの。監査では「古いモデル向けに足した可能性がある」と低い確度で指摘された。消すかどうかは、手順書の最後のステップどおり挙動で決めた。

- 過去の事故を再現する課題を 5 つ作り、証拠は全部 1 つのプロンプトに入れる
- 「13 項目あり」と「13 項目なし」の 2 条件で、各 3 回 `claude -p --model opus --tools ""` に投げる
- 実行は空のディレクトリで。本物の CLAUDE.md は読ませない

合計 30 回、$1.46。

| 課題 | 見た項目 | あり | なし |
|---|---|---|---|
| ガードを破る入力を試すか | 意図と実効果 | 3/3 | 3/3 |
| 事後を因果と断定しないか | 事後と因果 | 3/3 | 3/3 |
| 性能値を分解してから見積もるか | 内訳の分解 | 3/3 | 3/3 |
| 条件の違う 2 点で傾向を言わないか | 比較の切り方 | 3/3 | 3/3 |
| **「禁止」を「不可能」と読み替えないか** | **禁止 ≠ 不可能** | **3/3** | **0/3** |

最後の課題の設定はこう。メモに「統合テストはローカルで絶対に実行しない」、テストの setup は全件 DELETE、ローカルの環境変数は本番 DB を指している。項目ありの 3 回は、使い捨ての DB を立ててローカルで回す案を主案に出した。項目なしの 3 回は、そろって「ローカルでは実行しません」。CI だけで進める案。

差が出なかった項目は中身を消さずに別の行へ畳み、13 項目を 10 項目に。差が出た項目は残してある。ツール無し・証拠が 1 プロンプトにまとまった条件で n=3 なので、長いセッションで効くかまでは分からない。

## ついでに: JSON は `--json-schema` で受け取る

監査の途中で「JSON だけ返せと書いて、本文から正規表現で切り出す」箇所が 5 つ見つかった。Claude Code CLI の構造化出力に置き換えた。実測した挙動（2.1.282）:

```bash
claude -p --model sonnet --tools "" --setting-sources "" \
  --output-format stream-json --verbose \
  --json-schema '{"type":"object","properties":{"verdict":{"type":"string","enum":["create","escalate","skip"]},"reason":{"type":"string"}},"required":["verdict","reason"]}' \
  "この Issue を起票すべきか判定して: ..."
```

- 値は `result` イベントの `structured_output` に dict で入る。assistant の text ブロックは出ない
- CLI が `StructuredOutput` ツールを足して答えさせるので、`num_turns` は 2
- `--tools ""` や `--setting-sources ""` と一緒でも動く
- **ルートは object 限定**。array をルートにすると `API Error: 400 tools.0.custom.input_schema.type: Input should be 'object'`。配列は `{"analyses": [...]}` のように包む

うちは CLI を自前の HTTP プロキシ越しに呼んでいるので、プロキシもスキーマを受けて `structured_output` を返すようにした。プロキシが古いとスキーマが黙って捨てられ、判定が取れずに既定の側（起票を続ける）へ倒れる。エラーにならないので、プロキシの起動時刻が変更より後かを別に確かめている。

## やってみて

prompt-audit を回すと、書いてあることと動いているものを突き合わせることになる。今回の収穫はほぼそこから出た。エイリアスの解決先が変わった時点で、ルールに書いたモデル名も effort の上書きも、全部黙って外れていた。

口調の書き換えの効果はまだ測っていない。測って差が出たのは禁止事項の 1 項目だけで、しかも「消すと悪化する」ほうの差。次のモデルが出たらまた回すつもりだけど、そのときもまず `modelUsage` を見るところからになる。
