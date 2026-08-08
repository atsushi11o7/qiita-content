---
title: お盆に実家へ帰っても自宅の ML サーバーを使いたくて Tailscale を導入した
tags:
  - tailscale
  - VPN
  - 自宅サーバー
  - SSH
  - 機械学習
private: false
updated_at: '2026-08-09T02:27:01+09:00'
id: f28ce268f7c563e2a534
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

機械学習の学習や実験は、自宅に置いた GPU マシン（以下 ML サーバー）で回しています。普段は同じ自宅 LAN の中から、計算資源を使いたいときに SSH でつないでいました。

お盆に帰省するあいだも GPU マシンを使いたかったのですが、本格的に LAN の外から使うには、ポート開放や VPN といったひと手間が必要になります。

そこで Tailscale を使ってみました。ポート開放なしで、自宅サーバーと手元の Mac を同じ仮想 LAN に入れてしまい、実家からでも普段どおり SSH できます。最終的には、その SSH 越しに Dev Container を開いて、いつもの環境に入るところまでやりました。

この記事は、その手順（サーバー導入 → 手元端末の追加 → SSH → Dev Container）を残したものです。

## 1. Tailscale とは（ざっくり）

Tailscale は、WireGuard という VPN プロトコルをベースにしたメッシュ型の VPN サービスです。同じアカウントでログインした端末どうしが、tailnet と呼ばれる 1 つの仮想ネットワークに入り、それぞれ `100.x.y.z` のプライベート IP を持ちます。あとはその IP（や後述のマシン名）宛てに、LAN 内と同じ感覚でアクセスできます。

うれしいのは次のあたりです。

- **ルータのポート開放や固定 IP、DDNS が要らない**（NAT 越えは Tailscale がやってくれます）
- **SSH をインターネットに晒さない**（tailnet に入った端末どうしでしか到達できません）
- **個人利用なら無料**（接続台数などに上限はあります）

ベースの WireGuard は速くて堅い一方、鍵の配布や NAT 越えは自前で用意する必要があります。Tailscale はそこを肩代わりして、「ログインするだけでつながる」状態にしてくれます。

このほかにも便利な機能がありますが、今回は「サーバーと手元の端末を直接つなぐ」最小構成なので使いません。

## 2. サーバー側（ML サーバー）に Tailscale を入れる

先に前提を 1 つだけ。最終的に SSH でつなぐので、サーバー側で SSH サーバー（sshd）が動いている必要があります（入っていなければ `sudo apt install openssh-server` で入れておきます）。

そのうえで、自宅の ML サーバー（Ubuntu）に Tailscale を入れます。公式のインストールスクリプトを実行するだけです。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

続いて `sudo tailscale up` を実行すると、認証用の URL が表示されます。

```bash
sudo tailscale up
# To authenticate, visit:
#     https://login.tailscale.com/a/xxxxxxxxxxxx
```

この URL をブラウザで開いてアカウントにログインすると、ターミナルに `Success.` と表示され、このサーバーが tailnet に参加します。

![tailscale_linux.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/3f66bd40-e937-4cc9-b59a-005cb5cf90ff.png)

## 3. 手元の Mac をネットワークに追加する

サーバーを認証すると、Web コンソールが「次は 2 台目のデバイスを追加しよう」と促してきます。この時点で mlserver は追加済みで、2 台目の接続を待っている状態です。ここでは手元の Mac を追加したいので、macOS を選びます。

![tailscale_next.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/39e340e2-5303-45f3-a97e-899e8fd5c520.png)

案内に従ってダウンロードページを開き、「Download Tailscale for macOS」からインストーラを取得します。

![tailscale_download.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/eac0714f-10ae-4414-ab21-5f7c009972dc.png)

あとはインストーラを進めるだけです。

![tailscale_install.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/9adb8c81-8369-4d50-9e2d-533bd3730725.png)

初回起動時に、「ネットワーク機能拡張」の許可を求められます。VPN 系のアプリはこの許可が必要なので、Tailscale Network Extension を有効にして「完了」を押します。

![tailscale_network.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/1a18a847-1a20-4b78-8ffb-03b0ea82b942.png)

サーバーと同じアカウントでサインインすれば、Mac も同じ tailnet に参加します。

## 4. つながったか確認して SSH する

Tailscale の管理コンソールの Machines ページを開くと、tailnet に参加しているデバイスが一覧できます。

![tailscale_machines.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/594a5ff5-6525-480a-b409-fe6ea2b4f4c2.png)

コマンドで確かめたいときは、サーバーや Mac で `tailscale status` を実行すると、参加中のデバイスと IP、接続状態が一覧で見られます。

あとは手元の Mac から、mlserver の欄に表示されているアドレス宛てに SSH するだけです。

```bash
ssh <ユーザー名>@<mlserver の Tailscale アドレス>
```

自宅の LAN にいるときと同じ感覚で、実家からでもそのままつながります。

なお、MagicDNS を有効にしておくと、`100.x.y.z` の代わりに `mlserver` のようなマシン名で SSH できます（今回は表示されたアドレスをそのまま使いました）。

## 5. SSH 越しに Dev Container を開く

私はふだん VS Code の Dev Container で開発しているので、この SSH 越しにそのまま開いてしまいます。

VS Code の「Remote - SSH」で mlserver に接続し、リポジトリを開いて「Reopen in Container」を選ぶだけです。これで実家の Mac から、自宅サーバーの GPU 付き Dev Container に入って、いつもどおり開発できます。

毎回アドレスを打つのが面倒なら、`~/.ssh/config` にホストを登録しておくと、VS Code の接続先一覧にも出てきて楽です。

```text
Host mlserver
    HostName <mlserver の Tailscale アドレス>
    User <ユーザー名>
```

回線越しでも、Tailscale が端末どうしを直接つなごうとするので、体感の遅延は思ったより小さめでした（多分環境によります）。

## おわりに

Tailscale をサーバーと手元の端末に入れるだけで、ポート開放なしに自宅の ML サーバーへ外から SSH し、そのまま Dev Container で開発する、というところまで通せました。帰省中も自宅の GPU がそのまま使えて快適です。

ルータの設定が要らず、SSH をインターネットに晒さず、しかも無料。導入は 10 分ほどでした。

残る心配は、真夏の部屋で GPU サーバーを動かしっぱなしにすることくらいでしょうか……。
