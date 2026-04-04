---
title: "Raspberry Pi 5を自宅サーバーにする完全ガイド"
emoji: "🍓"
type: "tech"
topics: ["raspberrypi", "docker", "selfhosted", "linux", "server"]
published: false
---

> **TL;DR**
> - Raspberry Pi 5 (16GB) を自宅サーバーにして半年以上安定運用中
> - 月の電気代は約100円。Mac miniの半額以下の初期投資
> - NVMe SSD換装でDocker起動が180秒→25秒に高速化
> - Tailscale + Caddy で外部からセキュアにアクセス
> - この記事ではOS初期設定からDocker構築、HTTPS化まで全手順をカバー

---

最近、AIエージェントを自宅で動かすのがトレンドになっています。OpenClaw（Claude Codeベースの自律AIエージェントフレームワーク）をはじめ、ローカルで常時稼働させるエージェントが増えてきて、「Mac miniを買って自宅サーバーにしよう」という流れも見かけるようになりました。

でも、個人開発レベルなら**月100円のラズパイ5で十分**です。Mac miniは10万円超、消費電力も10〜40W。ラズパイ5なら約4万円、消費電力7〜15W程度（NVMe SSD接続時）。性能はM4チップに遠く及びませんが、APIを叩いてオーケストレーションするサーバー用途なら十分です。AIエージェントを載せて半年以上運用している私の実感として、16GBモデルで必要十分です。

ちなみに、AIブームでメモリ需要が急増しているので、ラズパイもMac miniも今後値上がりする可能性が高いです。16GBモデルが手に入る今のうちに確保しておくのが賢明かもしれません。

この記事では、Raspberry Pi 5をサーバーとして立ち上げるまでの手順を、実体験ベースでまとめます。

---

## なぜラズパイ5をサーバーにするのか

「自宅サーバーほしいけど、電気代がなぁ...」という方、多いと思います。

私はRaspberry Pi 5を自宅サーバーとして半年以上運用していますが、**消費電力7〜15W程度（NVMe SSD接続時）、月の電気代100円以下**です。24時間365日つけっぱなし。エアコンの待機電力より安い。

クラウドVPSと比較するとこうなります。

| | ラズパイ5 | クラウドVPS（2コア4GB） |
|---|---|---|
| 初期費用 | 約4万円（本体+ケース+SSD） | 0円 |
| 月額ランニング | 約100円（電気代のみ） | 2,000〜4,000円 |
| 1年目の総コスト | 約33,200円 | 24,000〜48,000円 |
| 2年目以降の年コスト | 約1,200円 | 24,000〜48,000円 |

1年以内に元が取れて、2年目からは年間1,200円で自分だけのサーバーが動き続けます。しかもデータは完全に手元。クラウドベンダーの値上げも関係ありません。

Raspberry Pi 5はCPUがARM Cortex-A76にアップグレードされて、前モデル（Pi 4）から処理性能が2〜3倍になりました。Dockerコンテナを複数動かしても余裕があります。

---

## ハードウェア構成

私が使っている構成を紹介します。

| パーツ | 製品 | 参考価格 |
|-------|------|---------|
| 本体 | Raspberry Pi 5 (16GB) | 約18,000円（2026年4月時点の参考価格） |
| ケース | Pironman 5-MAX | 約10,000円 |
| ストレージ | NVMe SSD 500GB（Samsung 990 EVO等） | 約7,000円 |
| 電源 | USB-C 27W公式電源 | 約2,000円 |

**合計: 約32,000円**

### なぜPironman 5-MAXか

ファン付きアルミヒートシンクケースで、NVMe SSDスロットを内蔵しています。24/365稼働のサーバー用途では冷却が命です。このケースを使うと**CPU温度は常時50℃前後**で安定します。夏場でもサーマルスロットリングなし。

### NVMe SSDが最重要

microSDカードでも動きますが、**サーバー用途なら絶対にNVMe SSDにすべき**です。理由は単純で、I/O性能が10倍以上違います。Dockerイメージのpull、コンテナ起動、ログ書き込み——全部がストレージI/Oなので、microSDカードだとあらゆる場面でボトルネックになります。

microSDカードは書き込み寿命の問題もあります。サーバーのように常時ログを書くワークロードでは、数ヶ月で壊れるケースも珍しくありません。

---

## OS & 初期セットアップ

### OSインストール

Raspberry Pi Imager を使って **Raspberry Pi OS (64bit, Lite)** をインストールします。サーバー用途なのでデスクトップ環境は不要です。Lite版を選んでください。

Imagerの詳細設定（歯車アイコン）で以下を事前設定しておくと楽です。

- ホスト名
- SSHの有効化（パスワード認証 or 公開鍵）
- ユーザー名とパスワード
- Wi-Fi設定（有線LAN推奨ですが、初期設定用に）

### NVMe SSDからのブート設定

初回はmicroSDカードで起動し、その後NVMeブートに切り替えます。

```bash
# 現在のブートオーダーを確認
sudo rpi-eeprom-config

# ブートローダーの更新
sudo rpi-eeprom-update -a

# ブート順序をNVMe優先に変更
sudo raspi-config
# → Advanced Options → Boot Order → NVMe/USB Boot
```

Lite版（CUI）を使っている場合、GUIの「SD Copier」は使えません。別PCから Raspberry Pi Imager で直接NVMe SSDにOSを書き込むか、`dd` コマンドでコピーします。

```bash
# まずデバイスを確認（書き込み先を間違えるとデータ全損します）
lsblk

# ddでmicroSD→NVMeにコピーする場合
# ⚠ of= の指定先を絶対に間違えないでください。データが完全に消えます。
sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress

# ddはパーティションテーブルごとコピーするため、SSDの残り領域が未使用のまま
# まずpartedでパーティションを拡張し、その後resize2fsでファイルシステムを拡張する
sudo parted /dev/nvme0n1 resizepart 2 100%
sudo resize2fs /dev/nvme0n1p2
```

microSDカードを抜いて再起動し、NVMeから起動すればOKです。

### 固定IPの設定

サーバーなのでIPアドレスは固定にします。ルーター側でDHCP予約（MACアドレスに対して固定IPを割り当て）を使う方法もあります。OS側で設定する場合は以下の通りです。

```bash
# 接続名を確認（環境によって異なる）
nmcli con show
# "有線接続 1" や "Wired connection 1" 等が表示される

# 以下、表示された接続名に置き換えて実行
CON_NAME="有線接続 1"  # 英語環境では "Wired connection 1"
sudo nmcli con mod "$CON_NAME" ipv4.addresses 192.168.1.100/24
sudo nmcli con mod "$CON_NAME" ipv4.gateway 192.168.1.1
sudo nmcli con mod "$CON_NAME" ipv4.dns "192.168.1.1"  # ルーターIPを推奨（8.8.8.8等でも可）
sudo nmcli con mod "$CON_NAME" ipv4.method manual
sudo nmcli con up "$CON_NAME"
```

### Swap拡張

デフォルトのswapは100MBしかありません。Dockerコンテナを複数動かすなら最低4GBに拡張しましょう。NVMe SSDにswapを置く場合、頻繁な書き込みがSSDの寿命に影響する可能性があります。`vm.swappiness` を低めに設定して、なるべくメモリを使い切ってからswapに逃がすようにしましょう。

```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=4096/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon

# swappinessを低めに設定（デフォルト60 → 10）
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### セキュリティの基本設定

サーバーとして公開するなら最低限これはやっておきましょう。

```bash
# === 別PC（Mac/Windows等）で実行 ===
# SSH公開鍵をラズパイに登録
ssh-copy-id user@raspberrypi.local
# ↑が成功し、鍵認証でログインできることを確認してから次へ進む

# === ラズパイ側で実行 ===
# SSH鍵認証に切り替え（パスワード認証を無効化）
# ⚠ 必ず先に公開鍵でログインできることを確認してから実行してください
# 公開鍵未登録のまま無効化すると締め出されます
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# UFW（ファイアウォール）
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
# Tailscale経由のみにする場合は上の代わりに以下を使用:
# sudo ufw allow in on tailscale0 to any port 22
sudo ufw allow 80/tcp   # Caddyで外部公開する場合のみ
sudo ufw allow 443/tcp  # Tailscaleのみで運用するなら不要
# ※ Cloudflare Tunnelはアウトバウンド接続なのでポート開放は不要です
sudo ufw enable

# fail2ban（不正アクセス試行の検知・通知）
# 鍵認証を有効にしていれば突破される可能性は極めて低いですが、
# 不正アクセス試行のログ検知・通知目的で入れておくと安心です
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

---

## Dockerで環境構築

### インストール

```bash
# ※ curl | sh はスクリプトの中身を確認してから実行することを推奨します
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# ログアウト→ログインで反映
```

Docker Composeは現在Dockerに同梱されているので、別途インストールは不要です。

### なぜDockerか

ラズパイサーバーでDockerを使うメリットは3つあります。

1. **環境の再現性**: `docker-compose.yml` 一つでサーバーを丸ごと再構築できる
2. **コンテナ分離**: あるサービスが暴走しても他に影響しない
3. **メモリ制限**: コンテナ単位でメモリ上限を設定し、OOMキラーの被害を限定できる

特に3番目が重要です。16GBという限られたメモリを複数サービスで共有するので、一つのプロセスが暴走してシステム全体を道連れにする事態を防ぐ必要があります。

### docker-compose.yml の例

FastAPI + Redis の最小構成例です。

```yaml
services:
  app:
    build: .
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"
        reservations:
          memory: 512M
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: redis-server --maxmemory 64mb --maxmemory-policy volatile-lru
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: "0.25"
```

### メモリ配分のコツ

16GBのうち、OSとシステムプロセスで約1〜1.5GBは確保されます。Dockerに使えるのは実質14〜15GB。

私の環境ではこんな配分にしています。

| コンテナ | メモリ上限 | 用途 |
|---------|----------|------|
| agent | 4GB | メインアプリケーション（FastAPI） |
| browser | 768MB | Playwright + Chromium |
| redis | 128MB | キュー・状態管理 |
| proxy | 64MB | Docker Socket Proxy |

**ポイント: `deploy.resources.limits.memory` は必ず設定する。** 未設定だとコンテナがメモリを食い尽くしてOOMキラーに殺されます。何が殺されるかは運次第なので、最悪の場合sshd が死んで物理アクセスが必要になります。

---

## 外部からアクセスする

自宅サーバーを外部から使うための選択肢を紹介します。

### Tailscale（推奨）

**ポート開放不要でセキュリティリスクが最小**。これが一番おすすめです。

```bash
# ※ curl | sh はスクリプトの中身を確認してから実行することを推奨します
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

これだけで、Tailscaleネットワークに参加しているデバイス同士がVPN経由で通信できます。ルーターの設定変更もポート開放も不要。

個人利用は無料プラン（3ユーザー、100デバイス）で十分です。

### Caddy（自動HTTPS リバースプロキシ）

外部に公開する場合は、CaddyでリバースプロキシとHTTPS終端を行います。Let's Encrypt証明書の取得・更新が完全自動です。

```bash
# Caddyのインストール（公式リポジトリから）
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

```
# /etc/caddy/Caddyfile

myapp.example.com {
    reverse_proxy localhost:8080
}

dashboard.example.com {
    reverse_proxy localhost:3000
}
```

設定ファイルを書くだけで、HTTPS証明書の取得から更新まで全部やってくれます。Nginx + certbot の時代は終わりました。

### Cloudflare Tunnel（代替手段）

Cloudflareのドメインを使っているなら、Cloudflare Tunnelも選択肢です。Tailscaleと同様にポート開放不要で、Cloudflareのエッジ経由で接続できます。

### 独自ドメインの設定

Tailscale + Caddy の組み合わせで、独自ドメインでの運用ができます。私の環境では `https://dashboard.nanoradev.com` のようなドメインでアクセスしています。TailscaleのIP（100.x.y.z）はTailscaleネットワーク内でのみ到達可能なので、パブリックDNSに登録しても外部からはアクセスできません。TailscaleのMagicDNS機能を使うか、Cloudflare Tunnel等と組み合わせて外部公開する方法が一般的です。

---

## 運用のコツ

半年運用してきた中で、「最初からやっておけばよかった」と思うことをまとめます。

### 永続ログの設定

Dockerコンテナはデフォルトだと、コンテナが再作成（`docker compose up` 等）されたときにログが消えます。OOMでコンテナが死んだときこそログを見たいのに、ログも一緒に消えている——この悲劇を防ぐため、ホスト側にログを永続化しましょう。

```yaml
# docker-compose.yml
services:
  app:
    volumes:
      - /home/<ユーザー名>/logs:/var/log/app
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

アプリケーション側でもファイルにログを出力しておくと、コンテナが消えてもホスト側から調査できます。

### 自動更新

セキュリティアップデートは自動適用にしておきます。

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 温度監視

サーバーの温度は定期的に確認しましょう。

```bash
vcgencmd measure_temp
# → temp=48.9'C
```

Pironman 5-MAXのようなファン付きケースを使っていれば通常50℃前後で安定しますが、ケースなしだと80℃超えもありえます。70℃を超えるようなら冷却を見直してください。

### バックアップ

サーバーのデータは定期的にバックアップしましょう。私はresticを使って週次でWindows PCのHDDにバックアップしています。

```bash
# resticでバックアップ（初回）
restic -r /path/to/backup/repo init
restic -r /path/to/backup/repo backup /home/<ユーザー名>/data /home/<ユーザー名>/docker-volumes

# cronで自動化
# 毎週日曜 03:00 に実行
# ※ cronから実行する場合、RESTIC_PASSWORD（またはRESTIC_PASSWORD_FILE）の
# 環境変数をcrontabまたはラッパースクリプト内で設定する必要があります
0 3 * * 0 RESTIC_PASSWORD_FILE=/etc/restic-password restic -r /path/to/backup/repo backup /home/<ユーザー名>/data --quiet
```

Docker volumeの中身もバックアップ対象に含めることをお忘れなく。

---

## 次のステップ

ここまでで、Raspberry Pi 5を自宅サーバーとしてセットアップし、Dockerで複数サービスを動かし、外部からアクセスできる状態になりました。

ただ、これはまだ「箱」を作っただけです。ここから何を載せるかが本番です。

私はこのサーバーの上に**自律AIエージェント**を構築して、SNS投稿やコード修正、インフラ監視を24時間自動化しています。

---

## もっと深く知りたい方へ

実装の裏側（Docker Composeの全設定、セキュリティのレイヤー設計、NVMe換装のハマりポイント、半年運用の失敗談3選）は **[note有料記事（300円）](https://note.com/nanora_dev/n/nf916e7274f0b)** でまとめています。
