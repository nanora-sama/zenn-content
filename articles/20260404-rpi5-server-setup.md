---
title: "Raspberry Pi 5を自宅サーバーにする完全ガイド"
emoji: "🍓"
type: "tech"
topics: ["raspberrypi", "docker", "selfhosted", "linux", "server"]
published: false
---

最近、AIエージェントを自宅で動かすのがトレンドになっています。OpenClawをはじめ、ローカルで常時稼働させるエージェントが増えてきて、「Mac miniを買って自宅サーバーにしよう」という流れも見かけるようになりました。

でも、個人開発レベルなら**月100円のラズパイ5で十分**です。Mac miniは10万円超、消費電力も10〜40W。ラズパイ5なら3万円、消費電力5〜15W。AIエージェントを載せて半年以上運用している私の実感として、8GBモデルで必要十分です。

この記事では、Raspberry Pi 5をサーバーとして立ち上げるまでの手順を、実体験ベースでまとめます。

---

## なぜラズパイ5をサーバーにするのか

「自宅サーバーほしいけど、電気代がなぁ...」という方、多いと思います。

私はRaspberry Pi 5を自宅サーバーとして半年以上運用していますが、**消費電力5〜15W、月の電気代100円以下**です。24時間365日つけっぱなし。エアコンの待機電力より安い。

クラウドVPSと比較するとこうなります。

| | ラズパイ5 | クラウドVPS（2コア4GB） |
|---|---|---|
| 初期費用 | 約3万円（本体+ケース+SSD） | 0円 |
| 月額ランニング | 約100円（電気代のみ） | 2,000〜4,000円 |
| 1年目の総コスト | 約31,200円 | 24,000〜48,000円 |
| 2年目以降の年コスト | 約1,200円 | 24,000〜48,000円 |

1年以内に元が取れて、2年目からは年間1,200円で自分だけのサーバーが動き続けます。しかもデータは完全に手元。クラウドベンダーの値上げも関係ありません。

Raspberry Pi 5はCPUがARM Cortex-A76にアップグレードされて、前モデル（Pi 4）から処理性能が2〜3倍になりました。Dockerコンテナを複数動かしても余裕があります。

---

## ハードウェア構成

私が使っている構成を紹介します。

| パーツ | 製品 | 参考価格 |
|-------|------|---------|
| 本体 | Raspberry Pi 5 (8GB) | 約13,000円 |
| ケース | Pironman 5-MAX | 約10,000円 |
| ストレージ | NVMe SSD 500GB（Samsung 990 EVO等） | 約7,000円 |
| 電源 | USB-C 27W公式電源 | 約2,000円 |

**合計: 約32,000円**

### なぜPironman 5-MAXか

ファン付きアルミヒートシンクケースで、NVMe SSDスロットを内蔵しています。24/365稼働のサーバー用途では冷却が命です。このケースを使うと**CPU温度は常時50℃前後**で安定します。夏場でもサーマルスロットリングなし。

### NVMe SSDが最重要

microSDカードでも動きますが、**サーバー用途なら絶対にNVMe SSDにすべき**です。理由は単純で、I/O性能が10倍以上違います。Dockerイメージのpull、コンテナ起動、ログ書き込み——全部がストレージI/Oなので、SDカードだとあらゆる場面でボトルネックになります。

SDカードは書き込み寿命の問題もあります。サーバーのように常時ログを書くワークロードでは、数ヶ月で壊れるケースも珍しくありません。

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
# ブートローダーの更新
sudo rpi-eeprom-update -a

# ブート順序をNVMe優先に変更
sudo raspi-config
# → Advanced Options → Boot Order → NVMe/USB Boot
```

SD Copierツール、または `dd` コマンドでSDカードの内容をNVMe SSDにコピーしたら、SDカードを抜いて再起動。NVMeから起動すればOKです。

### 固定IPの設定

サーバーなのでIPアドレスは固定にします。

```bash
sudo nmcli con mod "有線接続 1" ipv4.addresses 192.168.1.100/24
sudo nmcli con mod "有線接続 1" ipv4.gateway 192.168.1.1
sudo nmcli con mod "有線接続 1" ipv4.dns "8.8.8.8,8.8.4.4"
sudo nmcli con mod "有線接続 1" ipv4.method manual
sudo nmcli con up "有線接続 1"
```

接続名は環境によって異なるので `nmcli con show` で確認してください。

### Swap拡張

デフォルトのswapは100MBしかありません。Dockerコンテナを複数動かすなら最低4GBに拡張しましょう。

```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=4096/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

### セキュリティの基本設定

サーバーとして公開するなら最低限これはやっておきましょう。

```bash
# SSH鍵認証に切り替え（パスワード認証を無効化）
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# UFW（ファイアウォール）
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# fail2ban（ブルートフォース対策）
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

---

## Dockerで環境構築

### インストール

```bash
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

特に3番目が重要です。8GBという限られたメモリを複数サービスで共有するので、一つのプロセスが暴走してシステム全体を道連れにする事態を防ぐ必要があります。

### docker-compose.yml の例

FastAPI + Redis の最小構成例です。

```yaml
version: "3.8"

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

### 8GBでのメモリ配分のコツ

8GBのうち、OSとシステムプロセスで約1〜1.5GBは確保されます。Dockerに使えるのは実質6〜7GB。

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
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

これだけで、Tailscaleネットワークに参加しているデバイス同士がVPN経由で通信できます。ルーターの設定変更もポート開放も不要。

個人利用は無料プラン（3ユーザー、100デバイス）で十分です。

### Caddy（自動HTTPS リバースプロキシ）

外部に公開する場合は、CaddyでリバースプロキシとHTTPS終端を行います。Let's Encrypt証明書の取得・更新が完全自動です。

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

Tailscale + Caddy の組み合わせで、独自ドメインでの運用ができます。私の環境では `https://dashboard.nanoradev.com` のようなドメインでアクセスしています。DNSレコードをTailscaleのIPに向けるだけです。

---

## 運用のコツ

半年運用してきた中で、「最初からやっておけばよかった」と思うことをまとめます。

### 永続ログの設定

Dockerコンテナはデフォルトだと、再起動するとログが消えます。OOMでコンテナが死んだときこそログを見たいのに、ログも一緒に消えている——この悲劇を防ぐため、ホスト側にログを永続化しましょう。

```yaml
# docker-compose.yml
services:
  app:
    volumes:
      - /home/pi/logs:/var/log/app
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
restic -r /path/to/backup/repo backup /home/pi/data /home/pi/docker-volumes

# cronで自動化
# 毎週日曜 03:00 に実行
0 3 * * 0 restic -r /path/to/backup/repo backup /home/pi/data --quiet
```

Docker volumeの中身もバックアップ対象に含めることをお忘れなく。

---

## 次のステップ

ここまでで、Raspberry Pi 5を自宅サーバーとしてセットアップし、Dockerで複数サービスを動かし、外部からアクセスできる状態になりました。

ただ、これはまだ「箱」を作っただけです。ここから何を載せるかが本番です。

私はこのサーバーの上に**自律AIエージェント**を構築して、SNS投稿やコード修正、インフラ監視を24時間自動化しています。興味がある方は次の記事をどうぞ。

---

## もっと深く知りたい方へ

このサーバーの上にAIエージェントを載せた話は、[こちらの記事](https://zenn.dev/nanora/articles/20260404-rpi5-autonomous-agent)で紹介しています。

さらに実装の裏側（Docker Composeの全設定、セキュリティ対策、OOM対策、半年分の運用コスト実績）は **[note有料記事（300円）](https://note.com/nanora_dev/n/nbae7a3033f74)** でまとめています。
