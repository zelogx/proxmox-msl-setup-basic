# MSL Setup Quota 機能（最小構成案）

## 1. 何に対して Quota を掛けるか（対象と前提）

### 対象（MSL的な前提）

- 単位：
  - Proxmox Pool = MSL テナント（PJ/チーム）
- 制限対象リソース：
  - vCPU 合計（テナント内 VM の `cores` または `sockets * cores`）
  - メモリ合計（`memory` / `maxmem` の合計）
  - ディスク仮想サイズ合計（各 VM の `virtioX`, `scsiX` の `size`）

### モデル

- 「割当上限（Admission Control）」で制御
- 実行中の使用率は制御対象外（監視のみ）

### 前提

- ストレージは LVM-Thin / ZFS / DIR 等を問わず、「仮想サイズ」は VM 設定から一貫して読める前提。
- MSL Setup 側は「VM 作成・変更・起動」を基本的にラップできる立場。

---

## 2. Quota 設定機能（どこにどう書くか）

### 2-1. hookscript を置く場所

```bash
# 例: hookscript を置く
mkdir -p /var/lib/vz/snippets
cp -p <install_dir>/zelogx-vm-hook.sh /var/lib/vz/snippets/zelogx-vm-hook.sh
chown root:root /var/lib/vz/snippets/zelogx-vm-hook.sh
chmod +x /var/lib/vz/snippets/zelogx-vm-hook.sh
```

### 2-2. 設定の置き場所（Pool comment）

- Proxmox Pool の comment を公式設定箇所にする。
- 例：`Datacenter → Permissions → Pools → <pjXX> → Comment` (XX:01-NUM_PJ)

```text
ZELOGX_QUOTA: vCPU=16 MEM_GB=64 DISK_GB=2000
```

- 初期値はインストール時に **全 Pool を 0**（＝無制限）で埋める。

```text
ZELOGX_QUOTA: vCPU=0 MEM_GB=0 DISK_GB=0
```

- 解釈ルール：
  - vCPU / MEM_GB / DISK_GB が **0 の場合は「制限なし」**
  - コメントに `ZELOGX_QUOTA` 自体が無い場合も「制限なし」

### 2-3. 設定 UI / CLI（優先度：低）

- まずは install shell で `pj01〜pjXX` のコメントを一括初期化。
- 将来の候補として：
  - MSL Setup CLI から Pool comment を安全に書き換え。
  - GUI 的には Proxmox の Pools 一覧で Comment がそのまま見えるので、専用画面は必須ではない。

---

## 3. 強制ロジック（Admission Control 本体）

### 3-1. Hookscript（pre-start / pre-migrate）

**目的**

- VM 起動・マイグレーションでテナント枠を超えないようにする。

**処理フロー（概要）**

1. 引数から `vmid` を取得。
2. `pvesh get /cluster/resources --type vm --output-format=json` 等で、この VM の pool 名を取得。
3. `pvesh get /pools/<poolid> --output-format=json` で Pool comment を取得し、Quota をパース。
4. 同じ Pool に属する **起動中 VM** を列挙し、vCPU / MEM / DISK の使用量を集計。
5. 「起動中の VM ＋ 起動しようとしている VM」の合計を計算。
6. `quota_vcpu` / `quota_mem` / `quota_disk` と比較。
7. Quota がゼロでない場合に限り、超過していたら NG。
8. NG のとき：
   - `exit 1` で pre-start 失敗 → 起動 / マイグレ失敗。
   - ログに現在値と枠を出力。

**ログメッセージ例**

```text
DISK quota exceeded: Tenant=pj01 vCPU=4/8/16 MEM_GB=32/32/64 DISK_GB=1000/1500/2000 (Req./Cur./Quota)
```

**hookscript の枠組みイメージ**

```bash
#!/bin/bash
vmid="$1"
phase="$2"

logger -t "ZELOGX-HOOK" "vmid=${vmid} phase=${phase}"

case "${phase}" in
  pre-start)
    # 1〜8 のチェック処理をここに実装
    # 超過していたら:
    #   logger -t "ZELOGX-HOOK" "CPU quota exceeded: ..."
    #   exit 1
    ;;
  post-start)
    ;;
  pre-stop)
    ;;
  post-stop)
    ;;
esac

exit 0
```

---

## 3-2. VM 作成・リソース変更時に hookscript(snippet) を必ず適用する機能

### チェックポイント

- インストール時に全てのVMに付与（またはpool指定のあるもののみ？）
  →Poolの指定なし→ありに変更したとき、.confの更新があるか確認要。更新がある
  のであれば、現在poolに属しているもののみに限ってhookscriptを付与すれば良い。
- VM 作成（MSL Setup の create VM スクリプト）
- VM 更新（CPU/MEM/ディスク変更）
- その他、MSL がラップする操作

### 監視方式の案

- `inotify` または `systemd` timer で `/etc/pve/nodes/<node>/qemu-server/*.conf` を監視。
- 新規 / 更新された VM config を検出したら、必ず hookscript を適用。
- hookscript付与により.confがファイル更新された場合、ファイル更新イベントが発生し、
  二重更新→永久ループにならないよう現在hookscriptが設定済みかチェックしてから更新する。

### 付与する snippet のイメージ

- VM config 内に次の行が入っている状態を保証する：

```text
hookscript: local:snippets/zelogx-vm-hook.sh
```

- すでに `hookscript:` が存在し、`local:snippets/zelogx-vm-hook.sh` が指定済みなら何もしない。
- 設定は `.conf` を直接書き換えず、`qm set` を使う。

```bash
qm set <VMID> --hookscript local:snippets/zelogx-vm-hook.sh
```

### 一括付与用スクリプト例(インストール時動作)

```bash
#!/usr/bin/env bash
set -euo pipefail

HOOK="local:snippets/zelogx-vm-hook.sh"

for conf in /etc/pve/qemu-server/*.conf; do
  vmid="$(basename "$conf" .conf)"

  # qm config が読めない瞬間を軽くリトライ
  for _ in 1 2 3 4 5; do
    if qm config "$vmid" >/dev/null 2>&1; then
      break
    fi
    sleep 0.2
  done

  # 既に hookscript があるなら何もしない
  if qm config "$vmid" 2>/dev/null | grep -q '^hookscript:'; then
    continue
  fi

  # hookscript 付与
  qm set "$vmid" --hookscript "$HOOK"
done
```

---

## 4. 画面イメージ

- `Datacenter → Permissions → Pools` の一覧で、  
  各 Pool の `Comment` に次のように表示される想定：

```text
ZELOGX_QUOTA: vCPU=0 MEM_GB=0 DISK_GB=0
ZELOGX_QUOTA: vCPU=16 MEM_GB=64 DISK_GB=2000
...
```

- 専用 UI を用意しなくても、Proxmox 標準の Pools 画面が  
  そのまま「Quota 一覧画面」として機能する。
