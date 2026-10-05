---
title: "MiniMax H3 に Veda を入れたら RTX 3090 で 35% 縮んだ。全 step に掛けると GB 素材が抜けない"
emoji: "🪡"
type: "tech"
topics: ["comfyui", "minimaxh3", "rtx3090", "attention", "benchmark"]
published: true
---

> **TL;DR**
> - H3 の attention を約 9 割省く ComfyUI ノード Veda が、RTX 3090 でも動いた
> - attention を間引いていない設定で、6.6 秒の動画が 525.7 秒から 341.2 秒に縮んだ
> - 背景が決まる最初の 3 step にも掛けると、GB 素材の緑が灰色になって抜けない
> - 別の Sparse Attention で間引き済みの作り方は、Veda に替えても速くならない
> - 総時間は読み込みと復号で 365〜628 秒ぶれる。比べるなら 1 step の時間（s/it）

## なぜ試したか

自宅の Windows 機（RTX 3090 24GB）で ComfyUI を常駐させ、MiniMax H3 の reference-to-video でキャラの動画を作っている。合成に使う GB（グリーンバック）素材なので、キャラを緑一色の背景の前に生成し、あとで緑を抜いて別の背景や文字と重ねる。背景が緑でなくなった動画は抜けないので使えない。mid 画質（640x1120）の 6.6 秒で 1 本 9 分近くかかる。

そこへ [@sep_is_heim さんの投稿](https://x.com/sep_is_heim/status/2106745352967864322)で、H3 用の高速化 Veda に公式の ComfyUI ノードが出たと知った。配布元の数字は RTX 5070 で 342 秒 → 130 秒（2.9 倍）。ただし RTX 30 系は「静的レビューのみ・実機未検証」と書いてある。自分の 3090 で、普段のワークフローのまま何秒縮むかを測った。

## 使ったもの

| 役割 | 使ったもの |
|---|---|
| 動画生成モデル | MiniMax H3 の reference-to-video（`minimax_h3_ref2va_pruned_int8_convrot.safetensors`。Comfy-Org/MiniMax-H3 の配布） |
| テキストエンコーダ・VAE | `qwen3vl_32b_minimax_h3_nvfp4_awq` / `minimax_h3_video_vae_int8_convrot` / `minimax_h3_audio_vae_fp32` |
| 高速化 LoRA（turbo） | `minimax_h3_ref2v_turbo_8step_v1.0_768p_comfyui_bf16` |
| 今回試した高速化 | [Veda-on-ComfyUI](https://github.com/veda-sparse/Veda-on-ComfyUI) 0.2.0（fd59c72）+ predictor `minimax_h3_t2va_veda_8nfe_600step_preview_fp8`（275MB） |
| 比べた既存の高速化（手本付きの作り方） | カスタムノード h3-optimizations の `H3SparseAttention`（video_budget 0.15）と `H3MemoryOptimization`、RIFE（`rife47.pth`）でのコマ補間 |
| サンプラー | `res_multistep`・scheduler `beta`。最初の 3 step は 12 step の刻み、残り 6 step は 8 step の刻み |
| 入力 | キャラの参照画像 2 枚（全身・顔）、曲（音声参照）。手本付きの作り方は、Kimodo で作ったヒップホップのモーションを Blender の人形に付けて緑背景で描いた手本動画 |
| 出力 | 640x1120・24fps。手本なしは 158 コマ（6.6 秒）、手本付きは 73 コマを作って RIFE で 145 コマ |
| 環境 | Windows・RTX 3090 24GB・ComfyUI 0.37.0・PyTorch 2.11.0+cu130・triton-windows 3.7.1 |

## Veda は何をするか

[Veda](https://github.com/veda-sparse/Veda-on-ComfyUI)（ICML 2026）は、小さな predictor（275MB）が attention マップのどのタイルが効くかを先に当てて、上位 10% ほどだけ計算する。重みには触らず、H3 の attention の呼び出しを差し替えるノードなので、LoRA や量子化した H3 にそのまま挟める。

注意書きは 2 つ。

- ComfyUI 自身の Sparse Attention ノードと重ねると Veda が呼ばれない
- predictor は 1344x768 などの 5 / 10 / 14 秒、8 step の turbo LoRA（高速化用の LoRA）で学習してある。それ以外のサイズ・step 数は学習範囲の外

## 入れ方

ComfyUI 0.37.0（Windows、torch 2.11.0+cu130、triton-windows は導入済み）で、ノードを clone して predictor を置き、再起動しただけ。

```bat
cd C:\ComfyUI\custom_nodes
git clone https://github.com/veda-sparse/Veda-on-ComfyUI
cd Veda-on-ComfyUI && git checkout fd59c72
mkdir C:\ComfyUI\models\veda
curl.exe -L -o C:\ComfyUI\models\veda\minimax_h3_t2va_veda_8nfe_600step_preview_fp8.safetensors ^
  https://huggingface.co/Veda-Sparse/Minimax-H3-T2VA-Veda-8NFE-600Step-Preview/resolve/main/minimax_h3_t2va_veda_8nfe_600step_preview_fp8.safetensors
```

要件は ComfyUI 0.38.0 以上だけど、0.38 の API は import できなければ素通しするコードになっていて、0.37.0 でも動いた。最初の実行でカーネルをコンパイルし、ログに `triton-int8: ok` が出た。

```text
Veda: backends on RTX 3090 (SM86): triton-int8: ok
Veda attention time: 70.55 s (11.76 s per model call x 6)
```

## うちの H3 の回し方

拡散モデルの動画生成は、ノイズから少しずつ絵を仕上げる処理を何回か繰り返す（この 1 回を step と呼ぶ）。うちは合計 9 step で、最初の 3 step で背景と構図が決まり、後半 6 step で細部を詰める。

turbo LoRA（ref2v 8step v1.0）単独だと、緑一色の背景を指示しても店内やピンクの壁を描く。そこで最初の 3 step を LoRA なしの本体で回して背景と構図を決め、残り 6 step を turbo に渡している（12 step の刻みの 3 番目と、8 step の刻みの 2 番目が同じ sigma になる組）。

手本動画（ダンスの参照）付きのショットは別の作り方で、半分のコマで作って RIFE で補間し、本体の直後に別のカスタムノードの Sparse Attention（`H3SparseAttention`、budget 0.15）を挟んでいる。

## 測り方

同じ入力・同じ seed（乱数の種）で 1 本ずつ投げ、ComfyUI の `/history` から実行時間を、ログから step ごとの時間を取った。緑背景かどうかは、1 秒ごとのコマの外周の画素で「G − max(R, B)」の中央値を取り、全コマ 20 以上なら合格とした（緑を切り抜く処理と同じ基準）。

## 結果: attention を間引いていない設定で 35% 縮んだ

手本なし・6.6 秒（158 コマ）・640x1120・turbo 3/12。

| 版 | seed 51 | seed 5 | turbo の 1 step | 緑（seed 51） |
|---|---|---|---|---|
| そのまま（全 attention） | 525.7 秒 | 529.9 秒 | 54〜57 秒 | 80 |
| turbo の 6 step だけ Veda | **341.2 秒** | **400.8 秒** | 26〜27 秒 | 86 |
| 全 step Veda | 268.2 秒 | 299.5 秒 | 28 秒 | −2 |

![同じ seed の 3 本を並べた比較。左からそのまま 525.7 秒、turbo の 6 step だけ Veda 341.2 秒、全 step Veda 268.2 秒（背景が灰色）](/images/20261005-minimax-h3-veda-rtx3090/compare_noref_s51.gif)
*seed 51。左: そのまま / 中: turbo の 6 step だけ Veda / 右: 全 step Veda（背景が灰色で抜けない）*

turbo の 1 step はほぼ半分になった。最初の 3 step は全 attention のままなので、全体では 24〜35%。seed 5 はそのままの版でも緑を守らずパーティ会場のような部屋になったので（turbo の既知のクセ）、画の比較は seed 51 で。turbo 側だけの版は顔・衣装の柄がそのままの版と同等で、足元の影はむしろ少なかった。

![3 秒目の顔の切り出し。左がそのまま、右が turbo の 6 step だけ Veda](/images/20261005-minimax-h3-veda-rtx3090/face_s51.jpg)

![step ごとの時間を足した積み上げ横棒。最初の 3 step は 153 / 159 / 77 秒、後半 6 step は 325 / 162 / 166 秒](/images/20261005-minimax-h3-veda-rtx3090/chart_sampling_s5.png)
*seed 5 のログから。後半 6 step（紫）だけが半分になる*

## 最初の 3 step にも掛けると GB の緑が崩れる

全 step に掛けた版は一番速い。でも背景が灰色になった。手本付きの作り方で全 step に掛けたときも、緑が白っぽい薄緑（13）になった。2 つの作り方・2 つの seed で同じことが起きて、人物はどれも崩れていない。

背景と構図は最初の数 step で決まるので、そこを間引くと背景の指示が弱まるのだと見ている（中身までは調べていない）。turbo の 6 step だけに掛ければ緑は 86 で、そのままの版より高い。だから Veda は turbo LoRA の後ろにだけ置いた。

GB 素材だから困ったけど、背景ごと作る普通の動画なら、全 step に掛けて 268 秒（ほぼ半分）もアリかもしれない。ただ背景の指示が弱まるのは同じなので、狙いどおりの背景になるかは見て決めることになる（背景ありの動画では試していない）。

![9 step のうち、後半 6 step にだけ Veda を掛けた図。最初の 3 step に掛けると GB の緑が灰色になる](/images/20261005-minimax-h3-veda-rtx3090/steps_veda.png)

```text
UNETLoader ─┬─ (LoRA なし・全 attention) ─ 最初の 3 step ─┐ latent を引き継ぐ
            └─ turbo LoRA ─ VedaSparseAttention ───────── 残り 6 step ─ 復号
```

Veda は attention の差し替えなので、置いた位置から下流のサンプラーにだけ効く。最初の 3 step のサンプラーは LoRA の手前のモデルを使っているので、そこには効かない。

## 間引き済みの作り方は速くならない

手本付き（半分のコマ + `H3SparseAttention`）の作り方で、Sparse を Veda に差し替えた結果。

| 版 | サンプリング（3 step + 6 step） | 総時間 | 緑 |
|---|---|---|---|
| そのまま（H3SparseAttention） | 237 秒 | 627.7 秒 | 65 |
| 全 step Veda（2 回） | 214 / 228 秒 | 387.9 / 364.8 秒 | 13 |
| turbo 側だけ Veda | 258 秒 | 371.7 秒 | 71 |

![手本付きの 3 本を並べた比較。仕上げ時間は 237 / 258 / 214 秒でほぼ同じ。全 step Veda は背景が白っぽい](/images/20261005-minimax-h3-veda-rtx3090/compare_fast_s5.gif)

サンプリング（9 step の合計）はぶれの範囲で、attention の時間も 1 呼び出し 10〜13 秒と変わらなかった。もともと間引いている作り方には上乗せが無い。

総時間の列だけ見ると、628 秒 → 365 秒で 4 割速いように見える。この差はモデルの読み込みと復号で、同じ構成でも 365〜628 秒ぶれた（このとき Windows の空きメモリは 0MB だった）。比べるならログの `s/it` を見る。

![1 本の総時間を、9 step の仕上げとそれ以外に分けた積み上げ横棒。仕上げは 214〜258 秒でほぼ同じ、それ以外が 114〜391 秒とぶれる](/images/20261005-minimax-h3-veda-rtx3090/total_vs_steps_fast.png)

## どう入れたか

自分のパイプラインでは、turbo を使って手本なしで作るショットにだけ、turbo LoRA の後ろへ `VedaSparseAttention` を挟むようにした。手本付きの作り方と最初の 3 step はそのまま。ComfyUI にノードが無いときは警告を出して Veda なしで作る。止まらないかわりに、遅いまま完走しても気づけるよう、出力のメタデータに Veda を使ったかを残している。

## 自分の環境で確かめるなら

- 全 step に掛けた版と、turbo 側（後半の step）だけに掛けた版の両方を作る。速さだけで選ばず、背景の指示が守られているかを数字で見る
- 比べる seed は、Veda なしの版で背景が正しく出るものを選ぶ
- 別の Sparse Attention やメモリ節約のノードがすでに入っている作り方では、差し替え前後を同じ seed で比べる。効かないことがある
- 総時間でなく、ログの `s/it` か `Veda attention time` で比べる
- ComfyUI 自身の Sparse Attention ノードとは重ねない
