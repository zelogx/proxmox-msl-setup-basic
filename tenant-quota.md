# [Design] Proxmox Tenant-level CPU/MEM/disk quotas per pool using hookscripts

This is a design note for adding **per-tenant resource quotas** to MSL Setup  
by using Proxmox pools + a common hookscript.

The attached screenshot (in my local notes) is an example of how the Pools
screen would look, with `ZELOGX_QUOTA` stored in the `Comment` field.



---

## 1. What the Quota applies to (scope and assumptions)

### Scope (from the MSL perspective)

- Unit:
  - **Proxmox Pool = MSL tenant** (per project / per team)
- Resources to be limited:
  - **Total vCPU**  
    - Sum of `cores` (or `sockets * cores`) of all VMs in the pool
  - **Total memory**  
    - Sum of `memory` / `maxmem`
  - **Total virtual disk size**  
    - Sum of `size` for each VM’s `virtioX`, `scsiX` disks
- Model:
  - Use **“configured allocation ceiling” (Admission Control)**  
    → check limits at create / change / start / migrate time
  - **Runtime utilization is out of scope** (only monitor it, do not enforce)

### Assumptions

- Storage backend can be LVM-Thin / ZFS / DIR / etc.  
  As long as the **“virtual size”** can be read consistently from the VM config.
- On the MSL Setup side, we assume we can basically wrap:
  - VM creation
  - VM config changes
  - VM start (and migration)  
  even though a user could still operate directly from the Proxmox GUI.

---

## 2. Quota configuration (where and how to store it)

### 2-1. Where to store the hookscript

We place a common hookscript under `local:snippets`, so every VM can share it:

'''bash
# Example: place the hookscript
mkdir -p /var/lib/vz/snippets
cp -p <install_dir>/zelogx-vm-hook.sh /var/lib/vz/snippets/zelogx-vm-hook.sh
chown root:root /var/lib/vz/snippets/zelogx-vm-hook.sh
chmod +x /var/lib/vz/snippets/zelogx-vm-hook.sh
'''

### 2-2. Where to store the quota (Pool comment)

- Use the **Proxmox Pool `comment` field** as the official configuration location.
- Example: `Datacenter → Permissions → Pools → <pj01> → Comment`

'''text
ZELOGX_QUOTA: vCPU=16 MEM_GB=64 DISK_GB=2000
'''

### 2-3. Initial values and UI / CLI behavior

- First, during install, the install shell will set QUOTA for all pools (`PJ01`–`PJXX`) in the Pool `comment`.
  - Initial value:

    '''text
    ZELOGX_QUOTA: vCPU=0 MEM_GB=0 DISK_GB=0
    '''

  - There is an idea to set it to the **maximum** values at install time,  
    but this breaks as soon as you expand hardware (add more CPU/RAM/disk),  
    so we set everything to `0` instead.

- (Low priority) It *might* be convenient to provide the following in an MSL Setup CLI:
  - Safely edit the Pool `comment` field from a CLI or simple edit UI.
  - Since the `comment` is already visible in  
    `Datacenter → Permissions → Pools`, we do **not** need a special display widget.
  - **“No quota” (unlimited)** means:
    - The field value is `0`, or
    - The parameter itself is missing.
  - Initial default tenant quota at MSL Setup install:
    - All zero, e.g.

      '''text
      ZELOGX_QUOTA: vCPU=0 MEM_GB=0 DISK_GB=0
      '''

---

## 3. Enforcement logic (Admission Control core)

### 3-1. Hookscript (pre-start / pre-migrate conceptually)

**Goal**

- Prevent a tenant from exceeding its quota **when a VM starts or migrates**.

**High-level flow**

1. Get `vmid` from the hookscript argument.
2. Use `pvesh get /cluster/resources --type vm --output-format=json`
   to obtain this VM’s **pool name**.
3. Use `pvesh get /pools/<poolid> --output-format=json`
   to read the Pool `comment` and parse the quota configuration.
4. From step 2, enumerate all **currently running VMs** in the same pool,  
   obtain their required CPU/MEM/DISK (configured values),  
   and calculate the total resource usage inside that pool.
5. For the same pool, calculate the **current** vCPU/MEM/DISK usage
   (maximum configured values) of all currently running VMs.
   - Effectively, we consider “running VMs” as the current usage baseline.
6. If **(running VMs + the VM about to start) > `quota_vcpu`**,  
   treat this as `CPU quota exceeds`.
7. If **(running VMs + the VM about to start) > `quota_mem`**,  
   treat this as `MEM quota exceeds`.
8. If **(running VMs + the VM about to start) > `quota_disk`**,  
   treat this as `DISK quota exceeds`.
9. If the quota is not zero and the quota would be exceeded:
   - Return `exit 1` from `pre-start` → fail the start / migration.
   - Log a message like:

     '''text
     CPU quota exceeded: Tenant=pj01 vCPU=4/8/16 MEM_GB=32/32/64 DISK_GB=1000/1500/2000 (Req./Cur./Quota)
     '''

     where `(Req./Cur./Quota)` means:
     - `Req.` : additional resource required by the VM being started
     - `Cur.` : current total of running VMs in the same pool
     - `Quota`: configured quota for that pool

**Example hookscript skeleton**

'''bash
#!/bin/bash
vmid="$1"
phase="$2"

logger -t "PVE-HOOK" "vmid=${vmid} phase=${phase}"

case "$phase" in
  pre-start)
    # Implement steps 1–9 above:
    # - get pool for this vmid
    # - get ZELOGX_QUOTA from pool comment
    # - sum vCPU/MEM/DISK of running VMs in that pool
    # - add this VM's configured values
    # - compare with quota and log/exit 1 if exceeded
    ;;
  post-start)
    ;;
  pre-stop)
    ;;
  post-stop)
    ;;
esac
'''

---

### 3-2. Ensuring the hookscript (snippet) is always applied on VM create / change

**Checkpoints**

- VM creation (through the MSL Setup `create VM` script)
- VM updates
- etc. (any place where MSL wraps VM configuration)

**How to detect updates**

- Use `inotify` or `systemd` to watch:
  - `/etc/pve/nodes/pve1/qemu-server/*.conf` (per-node VM config)
- Whenever a VM config is newly created or modified,  
  ensure the hookscript is attached.

**What needs to happen**

- The target VM must always have the hookscript (snippet) configured.

**Snippet to add to the VM config**

- The VM config should contain a line like:

  '''text
  hookscript: local:snippets/zelogx-quota-hook.sh
  '''

- If a `hookscript` is already specified and it is `local:snippets/msl-quota-hook.sh`,
  then do nothing.
- Do not modify the `.conf` file directly; instead, always use:

  '''bash
  qm set <VMID> --hookscript local:snippets/zelogx-vm-hook.sh
  '''

**Example script to attach the hookscript**

'''bash
#!/usr/bin/env bash
set -euo pipefail

HOOK="local:snippets/guard.sh"

for conf in /etc/pve/qemu-server/*.conf; do
  vmid="$(basename "$conf" .conf)"

  # There is a short window where 'qm config' might fail for a VM
  # that is still being created, so we retry lightly. (Guessing)
  for _ in 1 2 3 4 5; do
    if qm config "$vmid" >/dev/null 2>&1; then
      break
    fi
    sleep 0.2
  done

  # If a hookscript is already configured, do nothing.
  if qm config "$vmid" 2>/dev/null | grep -q '^hookscript:'; then
    continue
  fi

  # Attach the hookscript
  qm set "$vmid" --hookscript "$HOOK"
done
'''
