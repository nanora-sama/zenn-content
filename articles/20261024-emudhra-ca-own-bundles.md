---
title: "eMudhra 中間 CA 漏洩、OS もブラウザも最新なのにコンテナ 9 個と Python 環境 16 個が偽の証明書を信じていた"
emoji: "🔓"
type: "tech"
topics: ["security", "ssl", "certificate", "raspberrypi", "windows"]
published: true
---

> **TL;DR**
> - OS もブラウザも最新だったのに、コンテナ 9 個と Python 環境 16 個が偽の証明書を信じていた
> - eMudhra の中間 CA 8 本の秘密鍵が、証明書から誰でも復元できる状態だった
> - ブラウザと OS は塞がったが、自前の証明書を持つアプリは自分で上げるまで偽サイトを信じる
> - 確かめ方は `curl` 1 行。証明書エラーなら安全、繋がったら要更新
> - 直す場所は環境しだい（うちは venv・pip 本体・コンテナ・Edge・VS Code・uv）
> - 「更新した」の表示と、実際に動いている版は別物だった

![OS を更新しても、自前の証明書を持つものは古いまま](/images/20261024-emudhra-ca-own-bundles/where-ca-lives.png)

OS もブラウザも最新だったので、うちは関係ないと思っていた。念のため Claude Code に Raspberry Pi と Windows を調べさせたら、コンテナ 25 個中 9 個と Python の環境（venv）16 個が、漏れた鍵で作った偽の証明書を本物として受け入れた。通信を横取りされたら、API キーを抜かれ、`pip install` で偽のパッケージを入れられる状態だった。

Python・Node.js・Docker・Electron 製のアプリを使っているなら、ブラウザのチェックサイトで確かめただけでは終わらない。まず下の「確かめ方」の curl を打ってほしい。

## 何が起きたか

2026-10-20、eMudhra（インドの認証局）の中間 CA 証明書 8 本の秘密鍵が漏れていたと公表された。公表元は [we-have-a-problem.com](https://emudhra.we-have-a-problem.com/)（[日本語の要約](https://emudhra.we-have-a-problem.com/ja.html)）。

- 中間 CA は、ルート CA の下で実際にサイトの証明書を発行する CA。この鍵があれば、どのサイトの証明書でも作れる
- 原因は CA 側の鍵の作り方の誤りで、証明書から鍵を復元できた。2025 年 4 月に証明書が作られた時点から復元できたとされている
- Chrome と Firefox は 10-17、Microsoft は 10-20、Apple は 10-22 から遮断している

問題はブラウザの外にある。公表元はこう書いている: 失効の情報を使わないクライアントは、ソフトウェアを更新するまで漏れた中間 CA を信じ続ける。自前の CA バンドル（信頼するルート証明書の一覧）を持つアプリやライブラリ、Slack や Discord のような Electron アプリがこれに当たる。

もう 1 つ厄介なのが CT（Certificate Transparency、発行された証明書を公開ログに残す仕組み）。ブラウザは CT に載っていない証明書を拒否するので、漏れた鍵で作った偽の証明書にも気づける。curl をはじめ、ブラウザ以外の多くのクライアントは CT を見ない。つまり**ブラウザ以外のクライアントは、気づかれないまま中間者攻撃を受けうる**。

## 確かめ方

公表元がテスト用のホストを立てている。漏れた中間 CA で発行した証明書を返すので、繋がったらそのクライアントは偽サイトを信じる。

```bash
curl https://dv-g3b-emudhra.we-have-a-problem.com/
curl https://dv-g3b-aaa-emudhra.we-have-a-problem.com/
```

証明書エラーで落ちれば安全。中身が返ってきたら要更新。さらに CT に載っていない証明書も通すかを見るなら、`noct` の付いたホストを使う。

```bash
curl https://dv-g3b-noct-emudhra.we-have-a-problem.com/
curl https://dv-g3b-aaa-noct-emudhra.we-have-a-problem.com/
```

直したあとのうちの Raspberry Pi では、こう返る。

```text
$ curl -sS -o /dev/null https://dv-g3b-noct-emudhra.we-have-a-problem.com/
curl: (60) SSL certificate problem: unable to get local issuer certificate

$ node -e "fetch('https://dv-g3b-noct-emudhra.we-have-a-problem.com/').catch(e=>console.log(e.cause?.code))"
CERT_REVOKED
```

**`curl` が安全でも、他のクライアントが安全とは限らない。** 証明書の一覧はクライアントごとに別の場所にある。1 つずつ繋いで確かめる。

## うちで見つかったもの

使った環境:

| | |
|---|---|
| サーバー | Raspberry Pi 5（Debian 13）、Docker Engine 29.9.2、常駐コンテナ 25 個 |
| 開発機 | Windows 11（ComfyUI・LoRA 学習などの Python venv が多数） |
| 調査と修正 | Claude Code に任せ、判断と再起動の時刻は人が決めた |

10-23 に棚卸しした結果:

| 場所 | 見つかったもの | 直し方 |
|---|---|---|
| Pi 本体 | 証明書の置き場所が少なくとも 5 系統（Debian の `/etc/ssl`、Homebrew、pip の certifi、Node 同梱、uv 同梱） | `ca-certificates` を deb13u1 に、`brew upgrade ca-certificates`、Node 24.22.1、uv 0.13.5 |
| コンテナ | **25 個中 9 個が偽の証明書を信じていた**（うち 1 個は、起動のたびに走る `pip install` の pip 本体） | イメージの更新。配布元が直していないものは、ホストの `/etc/ssl/certs` を読み取り専用でマウント |
| Windows | **Python の venv 9 個が古い certifi のまま** | 全部 certifi 2026.10.19 に |
| Windows | Node 24.4.1 | 24.22.1 に |
| Windows・Pi | **pip 本体の古いもの**（venv 7 個。Windows は pip 23.0.1、Pi は PyPI から入れた 26.2.2 未満）。certifi を上げた venv とは別 | venv ごとに `python -m pip install -U pip`（26.2.2 以上） |

コンテナはイメージごとに証明書の一覧を持つので、ホストを直しても変わらない。全部の中から繋いで確かめた。1 個だけ、外へ TLS で繋がない（Docker ソケットの受け口だけの）コンテナは、配布元の更新待ちとして残している。

## 直す場所は人によって違う

同じ件でも、やることは使っているものしだい。うちで実際に分かれた点を並べる。

| 使っているもの | 証明書をどこから読むか | やること |
|---|---|---|
| Debian・Ubuntu の curl、システムの Python | OS の `/etc/ssl/certs` | OS の `ca-certificates` を更新するだけ。Debian 版の `python3-certifi` も OS の一覧を指していた |
| venv に pip で入れた certifi（requests など） | venv の中の certifi | venv ごとに `pip install -U certifi`（2026.10.19 以上） |
| pip 本体（`pip install` のときの接続） | pip の中に同梱した certifi。うちの実測では、Windows の pip 24.2 以上と Debian 版の pip は OS の一覧を使って拒否した。Pi で PyPI から入れた pip は 26.2.2 未満がすべて繋がった | `python -m pip install -U pip`（26.2.2 以上）。Docker の `pip install` はベースイメージの pip で走る（`python:3.11-slim` は pip 24.0 で繋がった） |
| Node.js | Node の中に同梱 | 26.12.1 / 24.22.1 / 22.23.4 以上 |
| uv | uv の中に同梱 | 0.13.5 以上 |
| Homebrew の curl | Homebrew の一覧（OS とは別） | `brew upgrade ca-certificates` |
| Git for Windows | 設定 `http.sslBackend` しだい | `schannel` なら OS の一覧を使うので Windows Update で足りる。`openssl` なら 2.56.0.3 以上 |
| macOS の組み込み curl・git・python3 | `/etc/ssl/cert.pem`（更新されていない） | 公表元に回避策がある |
| Electron アプリ（Slack・Discord・VS Code など） | アプリの中の Electron | アプリの更新と**再起動** |
| コンテナ | イメージごと | イメージの更新。全部の中から繋いで確かめる |

うちの Git for Windows は 2.55.0 で、公表元がパッチ済みとする 2.56.0.3 より古い。ところが `http.sslBackend` が `schannel` だったので、テスト用ホストには `CRYPT_E_REVOKED`（失効済み）で繋がらなかった。版の一覧だけで判断していたら、これを「要更新」に数えていた。逆に、版は新しいのに再起動待ちで古いまま動いているものもある（次の節）。**判定は、実際に繋いだ結果で決める。**

## 「更新した」のに古かったもの

ここがいちばん時間を食った。どれも「成功」と表示されているのに、動いている版は古い。

**Edge**: 自動更新のタスクは 10-23 19:38 と 10-24 9:08 に、どちらも結果 0x0（成功）。それでも 10-22 公開の 156 は来ておらず、155 のまま。手動で更新チェック（`MicrosoftEdgeUpdate.exe /ua`）をかけても変わらない。仕方がないので公式の配布 API（`https://edgeupdates.microsoft.com/api/products`）から企業向けの MSI を取り、ハッシュと署名を照合して入れた。

それでもまだ終わらない。MSI は exit 0 なのに、`msedge.exe` は 155 のまま。Edge が起動中だったので、新しい版は `new_msedge.exe` という名前で横に置かれ、Edge が終わるのを待っている状態だった。起動していたのはログオン時に裏で立ち上がる「スタートアップブースト」で、ウィンドウは 0 個。これを止めて、レジストリに入っている差し替えコマンド（`setup.exe --rename-msedge-exe`）を流したら、やっと 156 になった。終了コードは 23 で、これは「リネーム成功」を表す。

**WebView2 Runtime**: アプリが中に埋め込んで使う Edge のエンジン。Edge 本体とは別製品なので、Edge を 156 にしても 155 のまま残っていた。こちらは配布 API に載っていないので、Microsoft の配布リンクから入れ直し用のインストーラを取り、署名を確かめて入れてある。

**VS Code**: 登録上の版はパッチ済みの 1.143.1 なのに、`code --version` は 1.142.0。インストール先に `new_Code.exe` があり、Edge と同じく再起動待ち。

**uv**: `uv self update` が `Self-update is only available for uv binaries installed via the standalone installation scripts.` で拒否された。インストール記録（`uv-receipt.json`）は残っていたものの、入れた先が Git Bash 形式のパス（`/c/Users/...`）で書かれている。公式インストーラで同じ場所に入れ直すと、Windows 形式の記録が書かれて `self update` が通るようになった。拒否の理由がパスの形式なのかは、uv のコードを読んでいないので推測。

確かめるときは、インストーラの終了コードでも登録上の版でもなく、**実体のファイルの版**を見る。

```powershell
(Get-Item "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe").VersionInfo.ProductVersion
Get-ChildItem "C:\Program Files (x86)\Microsoft\Edge\Application" -Name   # new_msedge.exe が残っていたら差し替え待ち
```

## ついでに見つかったもの

CA の件で棚卸しを始めたら、ほかにも出てきた。

- Pi に**セキュリティ更新が 578 件溜まっていた**。`apt-daily-upgrade.timer` は毎日動いているのに、`unattended-upgrades` が入っていないので何も当たらない作りだった
- Cloudflare Tunnel に、認証（Cloudflare Access）の無い公開ホスト名が 3 つ。裏のサービスが止まっていて 502 を返していただけで、起動した瞬間に認証なしで外へ出る
- LAN に出ている VNC のパスワードが既定値のまま
- カーネルの更新は再起動が要るので、時刻を決めて翌朝 3 時に再起動。再起動の前に動いていたコンテナの一覧を保存し、起動後に戻っていないものを起こす確認を 1 回だけ走らせてある

最後に毎日の監査を置いた。テスト用ホストへ全コンテナから繋ぐ、未適用の更新を数える、公開ホスト名を見る、を毎朝回し、結果が変わったときだけ通知する。今は指摘 0 件。

## 手で直さないなら

うちは調べるのも直すのも Claude Code に任せた。「全部のクライアントとコンテナから繋いで、全部が証明書エラーになるまで」を完了条件にしたプロンプトと、そこで起きたことは note に書いた: https://note.com/nanora_dev/n/nfd4aa13e563c

## 自分の環境で確かめるなら

- `curl https://dv-g3b-emudhra.we-have-a-problem.com/` を、OS の curl と、ほかに入っている curl の両方で打つ
- pip 本体は版でなく繋いで見る: `python -m pip download --no-deps -d /tmp/x --index-url https://dv-g3b-emudhra.we-have-a-problem.com/simple/ dummy` が `SSLError` なら安全、`Skipping page ... Content-Type: text/plain` が出たら繋がっている（`No matching distribution` は拒否でもオフラインでも出るので目安にならない）
- Python の venv ごとに `python -c "import certifi; print(certifi.__version__, certifi.where())"` を見る。`where()` が venv の中を指していて版が 2026.10.19 より古ければ要更新
- `docker ps` の全コンテナの中から、上のホストに繋いでみる
- Electron アプリとエディタは、更新のあとに再起動したか確かめる
- Git for Windows は `git config --get http.sslBackend` を見る
- macOS なら公表元の回避策を読む
