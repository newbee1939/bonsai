# HomeLab PLAN

## ゴール

- 壊しても直せるインフラを自宅に持ち、SRE としての土台を手を動かして学ぶ
- すべてをコードで再現できる（Terraform で箱、Ansible で中身）
- 最終的に Home Assistant を主役に据える

## 進め方

- **手で作る → 説明する → コードにする → 消して作り直す**、の順を崩さない
- チェックを入れるのは「動いた」ときではなく「**何が起きたか 1 行で説明できた**」とき
- 詰まったら `docs/` に 3 行（何が起きた / 原因 / どう直した）

## 構成

```
[ Mac ]  git / terraform / ansible を叩く
   ▼
[ GEEKOM A5 ]  Proxmox VE 9.2（Ryzen 7 7730U / 16GB / 512GB）
   ├─ VM: docker  … Pi-hole / 監視 / 各種サービス
   ├─ VM: haos    … Home Assistant OS
   └─ VM: lab     … 壊す用
```

- **Proxmox VE** — 1 台の PC の上で複数の VM（仮想マシン = ソフトで作った PC）を動かす OS
- **Terraform** — VM などの「箱」をコードで作る / 消す
- **Ansible** — SSH で入って箱の「中身」（パッケージ・設定）を整える

## Phase 0 — ミニ PC を Proxmox にする

- [ ] ワットチェッカーでアイドル消費電力を測る（電気代の基準値）
- [ ] Ventoy で USB に Proxmox VE 9.2 の ISO を入れる
  - Ventoy: USB に ISO（OS のインストールイメージ）をコピーするだけで起動できるツール
- [ ] BIOS で AMD-V（SVM）と IOMMU を有効化
  - AMD-V: CPU の仮想化支援機能。無効だと VM が動かない
  - IOMMU: VM に USB 機器などを直接渡すための機能（Zigbee ドングルで使う）
- [ ] Proxmox をインストールし、No-Subscription リポジトリに切り替える
  - 初期設定は有料版の更新サーバーを向いていて、更新でエラーになるため
- [ ] ルータで固定 IP を予約し、Mac から SSH 鍵で入る（パスワード認証は切る）

**Done**: Mac から `ssh pve` と Web UI に入れる
**説明できる**: 電源 ON から Proxmox のログイン画面が出るまでに何が起きているか（UEFI → ブートローダ → カーネル → systemd）

## Phase 1 — 手で作る

コードにする前に、一度すべて手でやる。ここで「何を自動化するのか」が分かる。

- [ ] Proxmox の Web UI で VM を 1 台作り、Ubuntu Server 26.04 LTS を入れる
- [ ] SSH で入り、手で Docker を入れる
- [ ] 手で Pi-hole を立て、Mac の DNS だけ向けて広告が消えるのを確認する
  - Pi-hole: 家の DNS サーバー。広告ドメインへの問い合わせを握りつぶして広告を消す
- [ ] やった手順を `docs/manual.md` に全部書き出す（これが Phase 2 の設計書になる）
- [ ] VM を消す

**Done**: `docs/manual.md` だけ見て、同じ VM をもう一度手で作れる
**説明できる**: `dig example.com` を叩いたとき、問い合わせがどこを通って答えが返るか

## Phase 2 — コードにする

`docs/manual.md` の手順を、上から順にコードへ置き換える。

- [ ] `.gitignore` に `*.tfstate*` と secrets を書く
  - tfstate: Terraform が「今何を作ったか」を記録するファイル。API トークンが平文で入るので Git に載せない
- [ ] Proxmox に Terraform 用の API トークンを作る（権限は必要最小限）
- [ ] Ubuntu 26.04 の cloud image から cloud-init テンプレートを作る
  - cloud-init: VM の初回起動時に、ユーザー・SSH 鍵・IP を自動で設定する仕組み
- [ ] Terraform（`bpg/proxmox`、バージョンは固定）で `docker` VM を作る
- [ ] Ansible で初期設定（ユーザー / SSH / `ufw` / 自動更新 / Docker）
  - ufw: Linux のファイアウォールを簡単に設定するコマンド
- [ ] Docker Compose で Pi-hole を立て、ルータの DNS を向ける
- [ ] Tailscale で外から入る（ルータのポートは開けない）
  - Tailscale: 自分の端末同士だけを暗号化トンネル（WireGuard）でつなぐ VPN
- [ ] GitHub Actions で `terraform validate` と `ansible-lint`

**Done**: `terraform destroy` → `apply` → `ansible-playbook` で元に戻る
**説明できる**: `terraform plan` が差分をどう計算しているか（コード / tfstate / 実物の 3 つの比較）

## Phase 3 — 監視とバックアップ

SRE の本業。「壊れたら自分より先にシステムが気づき、データは必ず戻せる」状態を作る。

- [ ] Prometheus + Grafana + node_exporter + smartctl_exporter
  - Prometheus: 各サーバーから数値（CPU 使用率など）を定期的に集めて貯める
  - Grafana: 貯めた数値をグラフにする
  - exporter: 数値を Prometheus が読める形で出す小さなプログラム（smartctl は SSD の健康度）
- [ ] Uptime Kuma で外形監視、ntfy でスマホに通知
  - 外形監視: 外からアクセスして「応答があるか」だけを見る監視
- [ ] SLO を 1 つ決め、SLO 違反でアラートを鳴らす（例: DNS 応答成功率 月 99.9%）
  - SLO: 「ここまでは落ちてもよい」という信頼性の目標値。CPU が高くても利用者が困っていなければ鳴らさない
- [ ] VM は vzdump、データは restic で外付けへバックアップ
  - vzdump: Proxmox 標準の VM 丸ごとバックアップ / restic: 暗号化と重複排除つきのファイルバックアップ
- [ ] リストア手順を `docs/runbook.md` に書き、実際に戻す
  - runbook: 障害時に誰でも同じ手順で直せるようにした手順書
- [ ] Caddy + DNS-01 で内部サービスを HTTPS 化
  - DNS-01: ドメインの DNS に特定のレコードを置いて所有を証明し、証明書を取る方式。外部公開しなくてよい

**Done**: VM を消してバックアップから戻せた（所要時間を記録） / アラートがスマホに届いた
**説明できる**: Prometheus が何を、何秒おきに、どうやって取りに行っているか

## Phase 4 — Home Assistant

- [ ] HAOS を VM として建てる（Terraform で）
  - HAOS: Home Assistant 専用の OS。アドオン（追加機能）が使える公式推奨の形
- [ ] 既存の家電を統合する
- [ ] Zigbee ドングル + 温湿度センサーで「暑くなったらエアコン」
  - Zigbee: センサーや電球向けの省電力無線。ドングルを挿すと HA がその親機になる
- [ ] 設定の Git 管理と自動バックアップ
- [ ] サッカー（Team Tracker）でキックオフ通知 / ゴールで照明

**Done**: 自動化が毎日動き、家族が 1 つ日常的に使っている
**説明できる**: センサーの値が変わってからエアコンが動くまでの経路

> 家族が使い始めたら本番。止まったら `docs/postmortem/` に書く（何が起きた / 影響 / 原因 / 再発防止）。

## Phase 5 — 広げる（やりたいものだけ）

- [ ] Immich（写真）/ Vaultwarden（パスワード管理）/ Paperless-ngx（書類の OCR と検索）
- [ ] ネットワーク可視化（Pi-hole ログ → Grafana / ミラーポート + ntopng / VLAN）
  - ミラーポート: スイッチに全通信のコピーを 1 ポートへ流させる機能 / VLAN: 1 台のスイッチを論理的に別ネットワークに分ける
- [ ] ローカル LLM（Ollama + Open WebUI）
- [ ] k3s（`lab` VM 3 台で）
  - k3s: 軽量版 Kubernetes（コンテナを複数台にまたがって管理する仕組み）
- [ ] 電力の可視化（B ルート + Wi-SUN → HA）
  - B ルート: スマートメーターから 30 分ごとの電力量を直接もらう仕組み（電力会社に申請、無料）

## 道場（月 1 回、`lab` VM で壊して直す）

手順: 予想を 1 行 → 壊す → ログとエラーコードだけで推測 → 一次情報で答え合わせ → `docs/postmortem/` に書く

- [ ] `/etc/fstab` に存在しないディスク → 起動シーケンス
- [ ] ディスクを 100% 埋める → block と inode
- [ ] メモリを食い潰す → OOM Killer、cgroup
- [ ] デフォルトゲートウェイを消す → ルーティングと DNS の切り分け
- [ ] 時計を進めて証明書を切らす → TLS 検証、NTP
- [ ] Go で exporter を書く
- [ ] 年 1 回、Git とバックアップだけから全復旧する（所要時間を記録）
