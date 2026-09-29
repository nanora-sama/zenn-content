---
title: "MiniMax H3 の動画 VAE を int8 に替えたら、RTX 3090 で 5 秒動画が 95 秒から 80 秒になった"
emoji: "🎞️"
type: "tech"
topics: ["comfyui", "minimaxh3", "rtx3090", "vae", "benchmark"]
published: true
---

> **TL;DR**
> - MiniMax H3 の動画 VAE は、int8 版に差し替えるだけで decode が速くなる
> - RTX 3090 の 5 秒動画が、画質そのままで 95.3 秒から 80.2 秒に縮んだ
> - ブログの「2 倍」は VAE 単体の話。実生成の 7 割はサンプリングの時間
> - `--fast fp16_accumulation` は別の画像モデルの絵まで変えるので入れなかった
> - ベンチ中に Windows が 2 回落ちた。GPU の電力制限は再起動で外れる

## なぜ試したか

自宅の Windows 機（RTX 3090 24GB）で ComfyUI を常駐させ、MiniMax H3 で画像から 5 秒前後の動画を作っている。1 本 1〜7 分かかるので、少しでも縮むなら入れたい。

そこへ ComfyUI 公式ブログに [Making the MiniMax H3 video VAE 2x faster](https://blog.comfy.org/p/making-the-minimax-h3-video-vae-2x) が出た。RTX 5090 で VAE の encode と decode の往復が 24.3 秒から 12.7 秒になったという。自分の生成が実際に何秒縮むのかを、普段使っているのと同じワークフローで測った。

## ブログの施策は 3 つ

| 施策 | 必要なもの | 手元の状況 |
|---|---|---|
| encoder の fused kernel（GroupNorm・SiLU・padding を 1 パスに） | ComfyUI v0.36.0 以上 | 0.37.0 で動いていたので、とっくに入っていた |
| fp16 accumulation に対応した畳み込み（encoder 約 2.2 倍） | 起動引数 `--fast fp16_accumulation` | 試したが入れなかった（後述） |
| int8 decoder（decode 約 1.4 倍） | int8 の VAE ファイル | 入れた |

fused kernel は `comfy/ldm/minimax/vae.py` の `_fused_norm_pad` にある。版を上げていれば何もしなくていい。逆に言うと、0.36.0 以上で測った「従来」の数字にはこの改善がもう入っている。

## 入れ方

int8 の VAE は Hugging Face の `Comfy-Org/MiniMax-H3` にある。fp16 版（5.2GB）の差し替え用で、2.61GB。

```bash
curl -L -o models/vae/minimax_h3_video_vae_int8_convrot.safetensors \
  https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_int8_convrot.safetensors
```

あとはワークフローの Load VAE で `minimax_h3_video_vae_fp16.safetensors` をこれに替えるだけ。I2V のテンプレートでは、同じ VAE が「開始フレームの encode（MiniMaxH3ImageToVideo）」と「最後の decode（VAE Decode）」の両方に繋がっている。

うちは自前のアプリからワークフローを組み立てて ComfyUI に投げているので、ファイルがあるときだけ差し替えるようにした。ComfyUI の `/object_info/VAELoader` を読めば、選べるファイルの一覧が取れる。

```python
info = requests.get(f"{COMFY}/object_info/VAELoader").json()
spec = info["VAELoader"]["input"]["required"]["vae_name"]
choices = spec[0] if isinstance(spec[0], list) else spec[1].get("options", [])
if "minimax_h3_video_vae_int8_convrot.safetensors" in choices:
    graph[vae_node]["inputs"]["vae_name"] = "minimax_h3_video_vae_int8_convrot.safetensors"
```

ファイルを消せば次の生成から fp16 に戻る。切り戻しにアプリの再デプロイは要らない。

## 測り方

普段と同じグラフ（Turbo 8step LoRA・6 step・SageAttention・上半身の I2V）で、変えるのは VAE だけ。同じ seed を 2 本ずつ流す。

踏んだ罠は、そのまま手順に入れた。

- **各条件の最初の 1 本は捨てる。** モデルの読み込みと、初めて見る解像度・尺の初期化が乗る。今回、再起動直後の 1 本目は 1,114 秒かかった
- **同じグラフを 2 回投げても計測にならない。** ComfyUI がキャッシュを返して何も走らない。VAE 単体の計測では、参照画像を毎回別名でアップロードしてキャッシュを外した
- **ノードごとの秒は websocket で取る。** サンプリング中に `/history` をポーリングすると処理が止まることがある。`executing` イベントの切り替わり時刻を差し引けば、ノードごとの時間が出る

```python
ws = websocket.create_connection(f"ws://{HOST}/ws?clientId={cid}")
cur, t0, times = None, 0.0, {}
while True:
    msg = ws.recv()
    if isinstance(msg, bytes):          # プレビュー画像は読み飛ばす
        continue
    m = json.loads(msg)
    if m.get("type") != "executing" or m["data"].get("prompt_id") != prompt_id:
        continue
    now = time.time()
    if cur is not None:
        times[cur] = times.get(cur, 0.0) + now - t0
    cur, t0 = m["data"]["node"], now
    if cur is None:                     # node が None ならそのプロンプトは終わり
        break
```

## 結果

RTX 3090・電力制限 270W・ComfyUI 0.37.0。数字は 2 seed の平均。

| 通常画質・5 秒 | fp16 VAE | int8 VAE |
|---|---|---|
| 全体 | 95.3 秒 | **80.2 秒（-16%）** |
| サンプリング | 67.3 秒 | 62.8 秒 |
| decode | 24.9 秒 | **14.5 秒** |

VAE だけを取り出した往復（1344x768・129 フレーム。ブログと同じ条件）はこうなった。

| | encode | decode | 往復 |
|---|---|---|---|
| fp16 VAE | 42.3 秒 | 40.5 秒 | 82.8 秒 |
| int8 VAE | 47.9 秒 | **22.5 秒** | 70.4 秒 |

画質は、動きの量・鮮鋭度・色相の変化を数値にしたものがどちらも同じ値になった。同じ seed の動画を並べても見分けがつかない。fp16 VAE の出力に対する PSNR は、フレーム単体で 56.7dB。VAE 自体が元画像を再構成するときの誤差（32.1dB）よりずっと小さい。

1344x768 の final 画質では、fp16 VAE の decode が 45 秒の回と 668 秒の回に割れた。int8 VAE では 21 秒と 22 秒でそろった。H3 本体（約 20GB）と fp16 VAE（5.2GB）が 24GB の VRAM に収まらず、退避と読み込み直しが起きていたのだと見ている（ここは推測）。

## ブログの 2 倍が 16% になった理由

ブログの数字は VAE の往復だけを測ったもの。実生成では時間の 71% をサンプリングが使っていて、VAE の decode は 26%。decode を半分近くにしても、全体では 16% 前後にしかならない。計算どおりの結果だった。

encode 側の改善はほぼ取り込んでいない。fused kernel は「従来」の数字にすでに入っていたし、`--fast fp16_accumulation` は入れていない。そもそも I2V で encode するのは開始フレームの 1 枚だけなので、encode が速くなっても実生成にはほとんど効かない。

逆に decode は、ブログの 1.4 倍より大きい 1.8 倍になった。ブログの 1.4 倍は「以前の int8 VAE」との比較で、こちらは fp16 VAE との比較になっているからだ。

## `--fast fp16_accumulation` を入れなかった理由

VAE 単体では encode が 43.6 秒から 33.7 秒になり、効果はあった（どちらも電力制限をかける前の 350W で計測）。それでも入れなかった理由は 2 つある。

1 つ目は、このフラグが `PRIORITIZE_FP16` も立てること（`comfy/model_management.py`）。fp16 を受け付けるモデルは、bf16 から fp16 に切り替わる。`supported_models.py` で見ると、Anima・Flux2・Wan などが該当する。同じマシンで Anima の画像も作っているので、同じ seed で前後を比べてみると、絵が変わっていた。構図とキャラはそのままで、髪飾りなどの細部だけ違う。seed を控えて作り直す使い方をしていると、これは困る。なお H3 本体と Qwen-Image は bf16 専用なので影響を受けない。

2 つ目は、効くのが I2V ではほとんど使わない encode だけだったこと。

## 躓き 1: ベンチ中に Windows が 2 回落ちた

1 回目はフラグを付けて 5 秒動画を連続で回していたとき。ComfyUI が `CUDA error: unknown error` で落ち、再起動後は `cudaGetDeviceCount()` が失敗し続けた。GPU が OS から見えなくなり、画面も真っ暗。電源ボタンで落とすしかない。イベントログに残っていたのは電源ボタン由来のシャットダウンだけで、原因は ComfyUI のログにしか書かれていなかった。

再起動後に GPU の電力制限を見ると、普段 270W に絞っているはずが既定の 350W になっていた。`nvidia-smi -pl` の設定は再起動で消えるので、事故のときも外れていた可能性が高い。起動時に掛け直すタスクを作った。

```bat
schtasks /Create /F /TN GPU_PowerLimit_270W /TR "C:\Windows\System32\nvidia-smi.exe -i 0 -pl 270" ^
  /SC ONSTART /DELAY 0001:00 /RU SYSTEM /RL HIGHEST
```

一度 300W にしてからタスクを手で実行し、270W に戻ることまで確認済み。270W でも速度は落ちていない。フラグ無しの Anima は 24.1 秒から 20.8 秒、VAE の往復は 82.8 秒で、350W のときと同じ水準だった。

2 回目は 270W・フラグ無しで、5 秒動画を作るために H3 本体をロードしている最中に自動再起動した。今度はバグチェック 0x44 のダンプが残っている。Windows SDK の cdb で読むと、落ちたのは GPU ではなく USB のドライバだった。

```bat
cdb.exe -z C:\Windows\Minidump\<dump>.dmp -c "!analyze -v; q"
:: FAILURE_BUCKET_ID:  0x44_USBXHCI!Isoch_Transfer_CompleteCancelable
```

Isochronous 転送は USB のオーディオ機器などが使う転送方式で、つないでいたのは VR ヘッドセットのマイクだった。この PC は int8 VAE を入れる前から、重い生成の最中に 0x133 や 0x116（GPU のタイムアウト）で落ちたことがある。巨大なモデルを VRAM に出し入れするたびに、系全体が不安定になっているのだと思う。int8 VAE のせいだという証拠は出ていない。

## 躓き 2: 裏で Unity が動いていた

final 画質の int8 計測だけ、サンプリングが 40〜90 秒遅い。VAE を替えてもサンプリングの計算は変わらないはずなので、おかしい。

Windows 側で起動中のプロセスを見ると、計測を始める 30 分前から Unity エディタが立ち上がっていた。計測のあと ComfyUI が待機している間も、GPU 使用率は 21%。GPU を取り合っていたので、この回の全体時間は比較に使えない。decode だけを見れば、21〜22 秒で安定している。

通常画質の比較は、どちらも Unity を起動する前に取っていたので影響はない。

## 自分の環境で確かめるなら

- ComfyUI の版を見る。0.36.0 以上なら fused encoder はもう入っている
- int8 VAE に替えて、同じ seed で 2 本ずつ流す。最初の 1 本は捨てる
- 比べるのは全体時間より decode の秒。サンプリングは VAE で変わらない
- 計測前に、待機中の GPU 使用率が 0% 近いか見る（ゲームエンジンやブラウザが裏にいないか）
- `--fast fp16_accumulation` を試すなら、ほかの画像モデルも同じ seed で前後を比べる
- 長時間回す前に `nvidia-smi --query-gpu=power.limit --format=csv` で電力制限を確かめる
