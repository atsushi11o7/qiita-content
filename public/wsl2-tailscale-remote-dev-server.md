---
title: WSL2 + Tailscale で、外出先から使える Linux ライクな GPU サーバーを作る
tags:
  - WSL2
  - tailscale
  - SSH
  - Docker
  - Windows
private: false
updated_at: '2026-08-23T02:53:49+09:00'
id: 47bb5ed5f9c8dc3095f4
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

以前、自宅の Linux マシンを Tailscale 経由の開発・機械学習サーバーとして使っていました。

https://qiita.com/atsushi11o7/items/f28ce268f7c563e2a534

そのマシンで、物理メモリの一部故障に続いて、CPU クーラーの冷却液がうまく循環しなくなったとみられる不調が発生しました。起動直後から CPU 温度が高くなる状態で、無人のまま高負荷な処理を回し続けるサーバーとして使うのは危険と判断し、退役させることにしました。

長期外出中に使える GPU サーバーはやっぱり欲しいので、Windows PC を活用します。Windows PC は通常通り使えるようにしつつ、WSL2 の Linux 環境に、外部の Mac から Tailscale 経由で SSH 接続できるようにする、というのが今回のゴールです。

具体的には、次のような状態を目指しました。

- 外部の Mac から Tailscale 経由で接続できる
- SSH で CUI 操作できる
- WSL2 上の Ubuntu を、Linux サーバーとして使える
- Docker と GPU を使える
- 誰もログインしていない状態でも、再起動後も、遠隔から使える状態を保てる

この記事では、繋がることを確認し、中身を整え、無人でも繋がり続けるようにし、実際に無人で動くか検証する、という順に進めます。

## Tailscale で接続できるようにする

まず、Windows に Tailscale をインストールし、Mac など既存の端末と同じアカウントに参加させます。Windows 版はタスクトレイに常駐するタイプです。

なお、Mac 側に Tailscale を導入する手順に関しては[前回の記事](https://qiita.com/atsushi11o7/items/f28ce268f7c563e2a534)を参考にしてください。

![tailscale_windows_1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/8aaf42d2-6c18-43b8-838b-5c53936f499d.png)

状態は PowerShell から確認できます。

```powershell
tailscale status
tailscale ip -4
```

タスクトレイのメニューからも、Connected になっていることを確認できます。

![tailscale_windows_2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/ac7c8f6c-0777-481c-bb4d-e9418e893c00.png)

次に、Windows 側を SSH で接続される側にします。管理者権限の PowerShell で、OpenSSH Server を追加します。

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

`Online: True` と表示されれば成功です（うまくいかない場合は、Windows の「オプション機能」から追加する方法もあります）。

続いて、sshd を起動し、自動起動に設定します。

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

`Automatic` にしておくのは、Windows を再起動したあとも SSH サーバーが自動で立ち上がるようにするためです。`Get-Service sshd` で `Running` になっていれば OK です。

```bash
ssh user@<Tailscale アドレス>
```

（Microsoft アカウントと Windows Hello を使っている場合、SSH で求められる認証情報と、Windows Hello の PIN は別物なので注意してください。）

Mac から接続すると、初回はホスト鍵の確認が出ます。接続後に `hostname` や `whoami` を叩けば、Windows に入れたことを確認できます。

```bash
$ ssh user@<Tailscale アドレス>
$ hostname
$ whoami
```

この状態では、SSH すると Windows の PowerShell に入り、そこから `wsl` と打って初めて WSL2 の Ubuntu に入れます。毎回打つのは面倒なので、OpenSSH の DefaultShell を wsl.exe に変更し、SSH した瞬間に直接 WSL2 に入るようにしました。

```powershell
New-ItemProperty `
  -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Windows\System32\wsl.exe" `
  -PropertyType String `
  -Force
```

これで「Mac → Tailscale → Windows → WSL2」という接続経路がひととおり繋がりました。

## WSL2 を Linux サーバーとして整える

続いて、WSL2 の Ubuntu を Linux サーバーらしく使えるように整えていきます。

まず systemd を有効にします（このあと Docker Engine のサービスを `systemctl` で管理するために必要です）。最近の WSL では最初から有効になっていることもあるので、まず確認します。

```bash
systemctl status
```

無効であれば、`/etc/wsl.conf` に次を追記します。

```
[boot]
systemd=true
```

Windows 側から `wsl --shutdown` を実行して WSL を再起動すれば反映されます。

Docker は、最初 Docker Desktop の WSL Integration を使っていました。ただ、SSH 経由で WSL に入った場合に `docker` コマンドがうまく機能しないことがありました。Docker Desktop 自体が GPU に対応していないわけではないのですが、Windows の GUI セッションや Docker Desktop の起動状態への依存をできるだけ減らし、無人の SSH サーバーとして扱いやすくするために、WSL の中に Docker Engine を直接入れる方針に変えました。

Docker 公式の手順で、WSL の Ubuntu に Docker Engine を導入します。一般ユーザーで使えるようにしておきます。

```bash
sudo usermod -aG docker $USER
```

（反映には再ログイン、または `wsl --shutdown` での WSL 再起動が必要です。）

```bash
docker ps
systemctl status docker
```

`sudo` なしで `docker ps` が通れば OK です。

（Docker Desktop を使っていた名残で、`~/.docker/config.json` に `"credsStore": "desktop.exe"` が残っていると Engine 直接運用の妨げになることがあるので削除しました。）

最後に、Docker コンテナから GPU を使えるようにします（Windows 側に WSL2 対応の NVIDIA ドライバを導入済みであることが前提です）。NVIDIA Container Toolkit を導入します。

```bash
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

確認します。

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.4.0-base-ubuntu22.04 \
  nvidia-smi
```

GPU 情報が表示されれば、Windows の NVIDIA ドライバ→WSL2→Docker Engine→NVIDIA Container Toolkit→コンテナまで、一気通貫で動いています。

## 無人でも繋がり続けるようにする

一度、PC を再起動したところ、次のような挙動をしていました。Windows 自体は起動していて、Tailscale 関連のサービスも動いているのに、Mac から接続できないのです。PC の前で PIN を入力してログインすると、そのとたんに繋がるようになりました。

サーバーとして利用する上で、誰もログインしなければ、Tailscale に繋がらないことは避けたいです。

解決策は、Tailscale の Unattended Mode を有効にすることです。タスクトレイの Tailscale メニューから、Preferences（または Settings）→ Run unattended を有効にします。CLI からなら次のコマンドでも設定できます。

```powershell
tailscale up --unattended
```

ユーザーの対話的なログインに依存せず、Tailscale を使える状態にするのが目的です。設定したら、必ず Windows を再起動し、ログインしない状態で Mac から繋がることを実際に確かめます。

無人運用では、電源設定も重要です。

まず、スリープを無効にします。設定 → システム → 電源 → 画面とスリープで、「デバイスをスリープ状態にする」を「なし」にします。

```powershell
powercfg /change standby-timeout-ac 0
```

スリープすると、SSH や Tailscale の通常の接続ができなくなるためです。

休止状態も無効にしておきます。

```powershell
powercfg /hibernate off
```

これを実行すると、高速スタートアップも合わせて無効になります（休止状態の仕組みを利用しているためです）。サーバー用途では、起動・シャットダウンの挙動を単純にしておきたいので、無効化しています。

最後に Windows Update です。長時間処理を回している最中に自動再起動されると、被害が大きくなります。長期の外出前は、設定 → Windows Update → 更新の一時停止で、しばらく更新されないようにしておきます。

なお、画面がロックされているだけなら問題ありません。Windows は動作中で、sshd もサービスとして動いているので、SSH で普通に接続できます。気をつけるべきは、スリープ・休止・シャットダウン、そして再起動後にサービスがうまく復旧しないケースです。

## 本当に無人で動くか検証する

ここまでの設定が本当に効いているかは、実際に検証してみないと分かりません。

まず、再起動テストです。Windows を再起動し、その後は一切操作しません。PIN を入力してログインする、といったこともしません。

```powershell
Restart-Computer
```

その状態で、Mac から接続します。

```bash
ssh user@<Tailscale アドレス>
docker ps
nvidia-smi
```

確認したいのは、次の経路がすべて自動でつながることです。

Windows 起動 → ログインなし → Tailscale 接続 → OpenSSH Server 起動 → SSH → WSL 起動 → systemd → Docker Engine → GPU 利用可能

1 回だけでなく、何度か再起動して確認しておくと安心です。

もう一つ、別ネットワークからの接続テストも行いました。同じ自宅の Wi-Fi から繋がるだけでは、外出先からの接続を再現できていません。そこで Mac をスマートフォンのテザリングに切り替えて、同じように接続を確認します。これで、実際に外出先から使うときに近い経路を検証できます。

なお、遠隔からの再起動やシャットダウンは慎重に行う必要があります。再起動後に Tailscale・sshd・WSL・Docker のどれかがうまく復旧しなければ、物理的にアクセスできるまで接続できなくなってしまいます。事前に再起動テストを十分にしていない状態では、安易に遠隔から再起動しないようにしています。

また、Tailscale も SSH も、あくまで Windows が正常に起動していることが前提です。OS がフリーズしたり、ブルースクリーンになったり、停電したりした場合は、どちらも役に立ちません。ソフトウェアだけではどうにもならない領域がある、ということは頭に置いておく必要があります。

## おわりに

Tailscale と OpenSSH Server、そして WSL2（systemd・Docker Engine・NVIDIA Container Toolkit）を組み合わせることで、Windows PC を、「電源さえ入っていれば Linux サーバーに近い形で遠隔から使える PC」にできました。

なお、運用を始めて数日経ったころ、接続していた Mac の電源を落とすと、WSL2 ごと落ちてしまう、という現象に遭遇しました。原因はまだ特定できていません。再接続は問題なくできるので現在はあまり困っていませんが、そのうち対応したいと思います。
