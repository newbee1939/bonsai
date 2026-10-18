# HomeLab PLAN

盆栽のように、小さく作って長く育てる。

## ゴール

- 自宅に「壊しても直せるインフラ」を持ち、Linux / ネットワーク / 仮想化 / 監視を手を動かして学ぶ
- 最終的に Home Assistant を主役に据える
- すべてをコードで再現できる（Terraform で箱、Ansible で中身）
- ランニングコスト（電気代 + ドメイン）を、解約したサブスクで上回る

## 原則

1. **手で設定したら負け** — 手で試すのはよい。動いたらコードに落とし、消して作り直す
2. **ポートは開けない** — 外からは Tailscale
3. **守るのはデータだけ** — OS はいつでも入れ直せる状態を保つ
4. **家族が使い始めたら本番** — 実験は `lab` VM で
5. **詰まったら `docs/` に 3 行** — 何が起きて / 原因は / どう直したか
6. **動いたら終わりにしない** — 1 段下まで説明できるか自問する

> homelab で付くのはインフラの土台（OS・ネットワーク・IaC・監視・物理）。
> スケール・トラフィック・組織は付かないので、そこは仕事で取る。両輪で回す。

## 構成

```
[ Mac ]  制御ノード。git / terraform / ansible を叩く（リポジトリはここだけ）
   │  LAN（SSH / Proxmox API）
   ▼
[ GEEKOM A5 ]  Proxmox VE。24/365 の本番機
   ├─ VM: docker  … Pi-hole / 監視 / 各種サービス
   ├─ VM: haos    … Home Assistant OS
   └─ VM: lab     … 壊す用（道場）
```

- GEEKOM A5: Ryzen 7 7730U（8C/16T）/ 16GB / 512GB NVMe / 2.5GbE。VM 3〜5 台なら十分
- 先に詰まるのは CPU ではなくメモリ。足りなくなったら SO-DIMM を増設する
- Windows PC は使わない

---

## Phase 0 — ミニ PC を Proxmox にする

**目的**: Mac から管理できる Proxmox ホストを 1 台持つ。

- [ ] ワットチェッカーを挟んでアイドル消費電力を測る（以後の基準値）
- [ ] Ventoy で USB に Proxmox VE 9.x の ISO を入れる
- [ ] BIOS で AMD-V（SVM）と IOMMU を有効化、停電復帰で自動起動する設定があれば ON
- [ ] Proxmox をインストール（同梱の Windows は消える）
- [ ] Enterprise リポジトリを無効化し、No-Subscription に切り替えて更新
- [ ] ルータで固定 IP を予約し、Mac から SSH 鍵で入る（パスワード認証は切る）

**Done**: Mac から `ssh pve` と Web UI に入れる / 電源を抜き挿ししても自動で戻る

**学ぶ**: Type-1 ハイパーバイザ、UEFI ブート、Linux ブリッジ（`vmbr0`）、SSH 公開鍵認証

## Phase 1 — Terraform で箱、Ansible で中身

**目的**: まっさらな Proxmox から、コマンドだけで全サービスが復活する。

- [ ] Mac に `terraform` / `ansible` を入れる
- [ ] `.gitignore` に `*.tfstate*` と secrets を**最初に**書く
- [ ] Proxmox に Terraform 用の API トークンを作る（権限は最小）
- [ ] Ubuntu の cloud image から cloud-init テンプレートを作る
- [ ] Terraform（`bpg/proxmox`）で `docker` VM を作る
- [ ] Ansible で初期設定: ユーザー / SSH / `ufw` / タイムゾーン / `unattended-upgrades` / Docker
  - IP・ホスト名は `inventory.yml` に隔離する
- [ ] Compose で Pi-hole を立て、ルータの DNS を向ける
- [ ] Tailscale を入れ、外から Proxmox と Pi-hole に入る
- [ ] GitHub Actions で `terraform validate` と `ansible-lint`

**Done**: `terraform destroy` → `apply` → `ansible-playbook site.yml` で元に戻る

> ⚠️ playbook は箱しか作らない。volume の中身（Pi-hole の設定など）は Phase 2 のバックアップで運ぶ。

**学ぶ**: 冪等性、cloud-init、Terraform state、DNS 解決（`dig` / `resolv.conf`）、Docker と iptables、WireGuard の NAT 越え

## Phase 2 — 監視とバックアップ（SRE の本業）

**目的**: 壊れたら自分より先にシステムが気づき、データは必ず戻せる。

- [ ] Prometheus + Grafana + node_exporter（CPU / メモリ / 温度 / ディスク）
- [ ] smartctl_exporter で SSD の健康度を見る
- [ ] Uptime Kuma で外形監視、ntfy でスマホに通知
- [ ] **SLO を 1 つ決める** — 例: Pi-hole の DNS 応答成功率 月 99.9%。アラートは CPU ではなく SLO 違反で鳴らす
- [ ] バックアップ: VM は vzdump、データは restic で外付けへ
- [ ] リストア手順を `docs/runbook.md` に書き、実際に戻す
- [ ] ドメインを取り、Caddy + DNS-01 で内部サービスを HTTPS 化（Cloudflare DNS を Terraform で管理）
- [ ] Renovate でイメージ / provider の更新を PR で受ける

**Done**: VM を消してバックアップから戻せた（所要時間を記録）/ アラートが実際にスマホに届いた

**学ぶ**: メトリクスの型と PromQL、SLO とエラーバジェット、3-2-1 バックアップ、TLS と DNS-01

## Phase 3 — Home Assistant

**目的**: 家が便利になる。ここからが盆栽。

- [ ] HAOS を VM として建てる（Terraform で）
- [ ] 既存の家電を統合する
- [ ] Zigbee ドングル + 温湿度センサー 1 個 → 「暑くなったらエアコン」
- [ ] 設定の Git 管理と自動バックアップ
- [ ] サッカー（Team Tracker）: キックオフ通知 / ゴールで照明 / ネタバレ防止モード

**Done**: HA を消して戻せた / 自動化 1 つが毎日動く / 家族が 1 つ日常的に使っている

> 家族の「電気がつかない」はインシデント。`docs/postmortem/` に書く。

## Phase 4 — 広げる（やりたい順に。全部やらなくていい）

- [ ] **脱クラウド** — Immich（写真）/ Vaultwarden / Paperless-ngx。⚠️ 写真はリストアを確認してから iCloud を解約
- [ ] **ネットワーク可視化** — Pi-hole ログ → Grafana / smokeping / TL-SG105E のミラーポート + ntopng / VLAN で IoT 隔離
- [ ] **クラウドも Terraform へ** — Tailscale ACL / GitHub。既存は `import` で差分ゼロにしてから
- [ ] **ローカル音声** — HA Assist + Whisper + Piper
- [ ] **ローカル LLM** — Ollama + Open WebUI。16GB なら CPU で 3B〜8B（4bit）。HA 連携は「操作させない」から
- [ ] **k3s** — 最後でいい。`lab` VM 3 台で組む

## Phase 5 — 電気代を下げる

- [ ] B ルート + Wi-SUN ドングルで電力を HA に取り込む
- [ ] 1 年貯める → 無料で削る（待機電力・プラン見直し）→ HA で自動化 → 投資はその後

---

## セキュリティの最低ライン

- [ ] ルータのポートを開けない（Cloudflare Tunnel を使うなら必ず Access を付け、管理画面は載せない）
- [ ] 初期パスワードを全部変える
- [ ] SSH のパスワード認証を切る
- [ ] 自動更新を入れる
- [ ] バックアップの 1 つはオフライン（外付けを月 1 回抜く）
- [ ] Secrets は平文で Git に置かない（SOPS / Ansible Vault）

## 道場（月 1 回、`lab` VM で壊して直す）

手順: 予想を 1 行 → 壊す → **ログとエラーコードだけで**推測 → 一次情報で答え合わせ → 外れた点を `docs/` に 3 行

- [ ] `/etc/fstab` に存在しないディスク → 起動シーケンス、emergency shell
- [ ] ディスクを 100% 埋める → block と inode、ログが書けないとき
- [ ] メモリを食い潰す → OOM Killer、`exit 137`、cgroup
- [ ] デフォルトゲートウェイを消す → `ip route`、疎通と DNS の切り分け
- [ ] ポートを衝突させる → `ss -tulpn`、iptables の NAT
- [ ] 時計を進めて証明書を切らす → TLS 検証、NTP
- [ ] 覗く: `tcpdump port 53` / Wireshark で TLS ハンドシェイク / `strace -f -p`
- [ ] Go で 100 行の exporter を書く → Prometheus は HTTP GET してテキストを parse しているだけ
- [ ] **年 1 回**: ミニ PC を消し、Git とバックアップだけから全復旧。所要時間を記録する

## コスト

| 項目 | 費用 |
|---|---|
| GEEKOM A5（購入済み） | 71,155 円 |
| 電気代 | Phase 0 で実測して記入（10W なら約 2,700 円/年） |
| ドメイン | 約 1,500 円/年 |
| Zigbee ドングル + センサー | 約 5,000 円 |
| ミラーポート対応スイッチ（Phase 4） | 約 3,000 円 |

解約候補: iCloud+ / パスワード管理 / VPN / 家計簿アプリ。電気代を削るより、サブスクを 1 個解約する方が速い。

## リポジトリ構成

```
homelab/
├── PLAN.md
├── terraform/   # Proxmox の VM、Cloudflare / Tailscale
├── ansible/     # inventory.yml / site.yml / roles/
├── compose/     # サービスごとの compose.yml
└── docs/        # 詰まった記録 / runbook / postmortem
```

## 詰まったら

- 30 分詰まったら検索する（道場のときは別）
- 1 日で解けなければ寝る
- 手が止まったら道場に切り替える
- 1 週間触らなくても気にしない

## 参考

- [Proxmox VE](https://pve.proxmox.com/pve-docs/) / [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs)
- [Ansible](https://docs.ansible.com/ansible/latest/) / [Terraform `import`](https://developer.hashicorp.com/terraform/language/import)
- [Prometheus](https://prometheus.io/docs/introduction/overview/) / [Tailscale](https://tailscale.com/kb)
- [Home Assistant](https://www.home-assistant.io/installation/) / [ha-teamtracker](https://github.com/vasqued2/ha-teamtracker)
