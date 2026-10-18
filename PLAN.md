# HomeLab PLAN

## ゴール

- 壊しても直せるインフラを自宅に持ち、SRE としての土台を手を動かして学ぶ
- すべてをコードで再現できる（Terraform で箱、Ansible で中身）
- 最終的に Home Assistant を主役に据える

## 構成

```
[ Mac ]  git / terraform / ansible を叩く
   ▼
[ GEEKOM A5 ]  Proxmox VE（Ryzen 7 7730U / 16GB / 512GB）
   ├─ VM: docker  … Pi-hole / 監視 / 各種サービス
   ├─ VM: haos    … Home Assistant OS
   └─ VM: lab     … 壊す用
```

## Phase 0 — ミニ PC を Proxmox にする

- [ ] ワットチェッカーでアイドル消費電力を測る
- [ ] Ventoy で USB に Proxmox VE 9.x の ISO を入れる
- [ ] BIOS で AMD-V（SVM）と IOMMU を有効化
- [ ] Proxmox をインストールし、No-Subscription リポジトリに切り替える
- [ ] ルータで固定 IP を予約し、Mac から SSH 鍵で入る（パスワード認証は切る）

**Done**: Mac から `ssh pve` と Web UI に入れる

## Phase 1 — Terraform で箱、Ansible で中身

- [ ] `.gitignore` に `*.tfstate*` と secrets を書く
- [ ] Proxmox に Terraform 用の API トークンを作る
- [ ] Ubuntu の cloud-init テンプレートを作る
- [ ] Terraform（`bpg/proxmox`）で `docker` VM を作る
- [ ] Ansible で初期設定（ユーザー / SSH / `ufw` / 自動更新 / Docker）
- [ ] Compose で Pi-hole を立て、ルータの DNS を向ける
- [ ] Tailscale で外から入る（ルータのポートは開けない）
- [ ] GitHub Actions で `terraform validate` と `ansible-lint`

**Done**: `terraform destroy` → `apply` → `ansible-playbook` で元に戻る

## Phase 2 — 監視とバックアップ

- [ ] Prometheus + Grafana + node_exporter + smartctl_exporter
- [ ] Uptime Kuma で外形監視、ntfy でスマホに通知
- [ ] SLO を 1 つ決め、SLO 違反でアラートを鳴らす（例: DNS 応答成功率 月 99.9%）
- [ ] VM は vzdump、データは restic で外付けへバックアップ
- [ ] リストア手順を `docs/runbook.md` に書き、実際に戻す
- [ ] Caddy + DNS-01 で内部サービスを HTTPS 化

**Done**: VM を消してバックアップから戻せた / アラートがスマホに届いた

## Phase 3 — Home Assistant

- [ ] HAOS を VM として建てる（Terraform で）
- [ ] 既存の家電を統合する
- [ ] Zigbee ドングル + 温湿度センサーで「暑くなったらエアコン」
- [ ] 設定の Git 管理と自動バックアップ
- [ ] サッカー（Team Tracker）でキックオフ通知 / ゴールで照明

**Done**: 自動化が毎日動き、家族が 1 つ日常的に使っている

## Phase 4 — 広げる（やりたいものだけ）

- [ ] Immich / Vaultwarden / Paperless-ngx
- [ ] ネットワーク可視化（Pi-hole ログ → Grafana / ミラーポート + ntopng / VLAN）
- [ ] ローカル LLM（Ollama + Open WebUI）
- [ ] k3s（`lab` VM 3 台で）
- [ ] 電力の可視化（B ルート + Wi-SUN → HA）

## 道場（月 1 回、`lab` VM で壊して直す）

- [ ] `/etc/fstab` に存在しないディスク → 起動シーケンス
- [ ] ディスクを 100% 埋める → block と inode
- [ ] メモリを食い潰す → OOM Killer、cgroup
- [ ] デフォルトゲートウェイを消す → ルーティングと DNS の切り分け
- [ ] 時計を進めて証明書を切らす → TLS 検証、NTP
- [ ] Go で exporter を書く
- [ ] 年 1 回、Git とバックアップだけから全復旧する
