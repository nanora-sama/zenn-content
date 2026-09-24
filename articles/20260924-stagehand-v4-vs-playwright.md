---
title: "Stagehand v4 は Playwright より速い？ — 97% 減の出どころと Pi 5 で起動しない理由"
emoji: "🎭"
type: "tech"
topics: ["stagehand", "playwright", "raspberrypi", "browserautomation", "cdp"]
published: true
---

> **TL;DR**
> - 「Stagehand v4 で LLM 呼び出し 97% 減・act 4.3 倍速」は、**まだマージされていない実験 PR**（#2951〜#2955、2026-09-24 時点で全部 OPEN）の数字でした
> - 比べている相手は Playwright ではなく、**Stagehand 自身の LLM 経路**です
> - 「Playwright の 2 倍速」「キャッシュで LLM コストほぼ 0」は、どちらも **Browserbase（クラウド）上**という条件つきです
> - Raspberry Pi 5（arm64）では標準の `local_browser.launch()` が `Extensions.loadUnpacked: Method not available` で**起動しませんでした**
> - 手元のブラウザ操作は全部固定セレクタで、Stagehand が速くする「LLM にクリック先を決めさせる処理」が 0 本だったので、移行はしないと決めました

---

X で「Stagehand v4 がブラウザ自動化の勢力図をひっくり返した」という投稿が流れてきました。LLM 呼び出し 97% 減、4.3 倍速、Playwright の 2 倍速、トークン 80% 減。

私は Raspberry Pi 5 で自律エージェントを動かしていて、ブラウザ操作もその中でやっています。本当に Playwright より明確に良いなら乗り換えたい。そう思って、GitHub の一次資料を当たり、Pi 5 で実際に動かしてみました。

結論から書くと、移行しませんでした。この記事ではその判断に使った材料を、再現できる形で残します。

## 投稿の主張と一次資料の突き合わせ

| 投稿の主張 | 一次資料で分かったこと |
|---|---|
| Jev 対応で act 4.3 倍速・LLM 呼び出し 97% 減（成功率 98.3%） | PR 本文の値（act p50 1.97s → 0.46s、147 回中 4 回、120 件中 118 件成功）から計算は合う。ただし**未マージ**で、比較相手は Stagehand の gemini-3.8-flash 経路 |
| observe 11.1 倍 / extract 8.7 倍 | observe は 16 件での数字。extract の 8.7 倍は、LLM 無しで終わった **75 件中 37 件だけ**の速度 |
| 出典は stagehand.dev/evals | このページに上の数字は無い。載っているのは Mind2Web のモデル別正答率 |
| Playwright の 2 倍速 | README の条件は「Browserbase 上で」。公式ブログ（2026-08-10）の実測は **1.59 倍** |
| キャッシュで LLM コストほぼ 0 | 公式 docs に "With a local browser … the cache option has no effect" |

### Jev は何か

Jev は TypeSafe 社のホスト型モデルです。文章は生成せず、型の決まった質問（選択・スコア）に確率で答えます。使うには Stagehand とは別に TypeSafe の API キーが要ります。

PR の本文には "thresholds were tuned on these suites (in-sample)" と書かれていて、閾値を決めたのと同じデータで成績を測っています。未知のサイトで同じ数字が出るかは、まだ誰も示していません。

PR の状態は誰でも確認できます。

```bash
for n in 2951 2952 2953 2954 2955; do
  gh pr view $n --repo browserbase/stagehand --json number,state,title \
    -q '"\(.number) \(.state) \(.title)"'
done
```

2026-09-24 に叩いた時点では 5 本とも `OPEN` でした。リリース済みの Python 版 4.1.0 には入っていません。

### 「Playwright の 2 倍速」の実体

v3 で Playwright への依存が外れ、v4 はブラウザ内の拡張機能（MV3）がコマンドをまとめて実行する作りになりました。公式の移行ガイドにも "there is no interop" とあり、Playwright の上に載るライブラリではなく、置き換える別のドライバです。

拡張機能の中で処理すると、SDK とブラウザの往復が減ります。これが効くのはブラウザが遠くにある場合で、公式ブログも "Client-to-remote round trip measured: 42.2ms" の条件で測っています。ブラウザが同じマシンにあれば往復は数 ms なので、減らせる分がそもそも少ないはずです。

## Pi 5 で動かしてみた

環境は Raspberry Pi 5（arm64 / Debian）、Stagehand Python 4.1.0、Playwright 1.63.0 です。

### 標準の launch() が起動しない

```python
import asyncio
from stagehand import local_browser

async def main():
    b = await local_browser.launch(headless=True)
    ...

asyncio.run(main())
```

```text
CDP command failed: Extensions.loadUnpacked: Method not available.
```

Debian の Chromium 143 でも、Playwright 同梱の Chromium 147 でも同じでした。headless でも、Xvfb 上の headed でも変わりません。

### port では失敗し、pipe では通る

同じ Chromium に同じ CDP メソッドを送って切り分けました。

- `--remote-debugging-port`（WebSocket）経由 → `Method not available`
- `--remote-debugging-pipe`（Playwright の `launch()` が使う方式）経由 → 成功して拡張 ID が返る

Stagehand の `launch()` は port 経由でこのメソッドを呼ぶので、この環境では通りません。公式 CI は x86_64 の Google Chrome で動いていて、Google Chrome と Chromium のどちらの差なのかまでは確かめていません。

### 動いた回避策（ただし本末転倒）

Playwright で起動して拡張機能を読み込み、そこへ Stagehand を繋ぐと動きました。

```python
# 1. Playwright（pipe）で起動し、拡張機能を読み込む
b = await p.chromium.launch(
    executable_path="/usr/bin/chromium", headless=True,
    args=["--enable-unsafe-extension-debugging",
          "--remote-debugging-port=9555", "--remote-allow-origins=*"],
    ignore_default_args=["--disable-extensions"],
)
s = await b.new_browser_cdp_session()
r = await s.send("Extensions.loadUnpacked", {"path": "<site-packages>/stagehand/_extension"})

# 2. 別プロセスから Stagehand を繋ぐ
b = await local_browser.connect(cdp_url="http://127.0.0.1:9555", extension_id=r["id"])
```

goto・クリックでの遷移・スクリーンショットまで通りました。ただ、Playwright から移行したいのに、起動を Playwright に頼ることになります。

LLM を設定しないまま `act()` を呼ぶと `An LLM was not configured` で止まります。セレクタ指定の決定的な操作は LLM 無しで動きます。

### ローカルでの速度

同じページで決定的な操作を 50 回ずつ実行した中央値（ms）です。計測中のロードアベレージが 9〜10 と高かったので参考値として見てください。

| 操作 | Stagehand 4.1.0 | Playwright 1.63.0 |
|---|---|---|
| click | 66〜79 | 81〜99 |
| fill | 111 | 18〜44 |
| inner_text | 51 | 6.5〜12.5 |

click はほぼ同じで、fill と読み取りは Playwright の方が速い結果でした。ローカルでは「2 倍速」は再現しませんでした。

## 足りない機能

公式の移行ガイドに、v4 に無いものが並んでいます。

- `route`・リクエストのモック
- `storageState`
- ダウンロードイベント
- 1 ブラウザにつきコンテキスト 1 つ
- Chromium のみ
- テストランナー・`expect`・trace・動画・codegen
- `page.pdf()`
- dialog ハンドラ

Python 4.1.0 では `page.on` が受け付けるイベントも `"console"` だけでした。PDF（#2819）とネットワークイベント（#2832）の PR は OPEN のままです。Issue #2744 では「15 本のスクリプトのうち 13 本が機能不足で移行できない」という報告も出ています。

## 決め手になったのは自分のコードだった

一次資料を読むより先にやるべきだったのが、自分のコードの棚卸しです。

私のエージェントのブラウザ操作を洗い出すと、次のようになっていました。

- 主力は、Playwright を使わず CDP を直接叩く自作のクライアント（arm64 で Playwright のリモート接続がうまく動かず、以前に乗り換えた）
- Playwright を直接 import しているのは 5 ファイルだけ
- **全部が固定セレクタで、LLM にクリック先を決めさせている処理は 0 本**
- LLM が画面を見るのは、撮ったスクリーンショットの合否判定と、テキストの要約だけ

Stagehand の速さの主張は、どれも「LLM が操作を決める」処理についての話です。その処理が 0 本なら、移行しても速くなる所がありません。逆に、決まったセレクタの操作に Stagehand の層を足す分だけ遅くなります。

## まとめ

- バズった倍率は「何と比べたか」を一次資料で確かめる。今回は相手が Playwright ではありませんでした
- 「対応した」は「リリースされた」ではない。PR の状態は `gh pr view` ですぐ見られます
- 条件つきの主張（クラウド上で、キャッシュが効けば）は、自分の環境がその条件に当てはまるか確かめる
- arm64 のように公式 CI と違う環境では、最小の手順を 1 回実際に動かす
- 乗り換えを考える前に、その道具が速くする処理が自分のコードに何本あるかを数える

もし次の 3 つがそろったら、見直すつもりです。

1. Jev と PDF・ネットワークイベントの PR がマージされる
2. arm64 の Chromium で `launch()` が起動する
3. 自分のエージェントに「LLM に画面を読ませて操作させたい」用途ができる

そのときも全面移行ではなく、その用途だけに足す形になると思います。
