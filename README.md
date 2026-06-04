# BeeGFS 8.3 on a Slurm Cluster (1 master + 3 workers, 8 TB each)

A single parallel filesystem mounted at `/mnt/beegfs` on every node.
Total usable pool ≈ 4 × 8 TB ≈ 32 TB (striped, RAID0 — see the warning at the end).

| Node       | Role(s)                                  | Data disk        |
|------------|------------------------------------------|------------------|
| master     | mgmtd + meta + storage + client          | `/dev/nvme3n1`   |
| worker01   | storage + client                         | `/dev/nvme3n1`   |
| worker02   | storage + client                         | `/dev/nvme0n1p1` |
| worker03   | storage + client                         | `/dev/nvme0n1`   |

> **v8 note:** BeeGFS 8 dropped the old `beegfs-setup-mgmtd/meta/storage` helper
> scripts. The management daemon is now Rust-based with a SQLite database and a
> TOML config, and the old `beegfs-ctl` tool is replaced by a new `beegfs` CLI.
> Binaries live in `/opt/beegfs/sbin/` and are **not** on `$PATH`.

---

## 0. Disk + package install (all nodes)

Same on every node; only the device name changes (see table above).

```bash
# Format the data disk (use the right device for this node)
sudo mkfs.ext4 /dev/nvme3n1          # worker02 -> nvme0n1p1, worker03 -> nvme0n1

# Repo + key
sudo wget https://www.beegfs.io/release/beegfs_8.3/gpg/GPG-KEY-beegfs \
  -O /etc/apt/trusted.gpg.d/beegfs.asc
sudo wget https://www.beegfs.io/release/beegfs_8.3/dists/beegfs-jammy.list \
  -O /etc/apt/sources.list.d/beegfs.list
sudo apt install -y apt-transport-https
sudo apt update

# Master:
sudo apt install -y beegfs-mgmtd beegfs-meta beegfs-storage beegfs-client beegfs-utils
# Workers:
sudo apt install -y beegfs-storage beegfs-client beegfs-utils

# Mount the data disk
sudo mkdir -p /data/beegfs/disk1
sudo mount /dev/nvme3n1 /data/beegfs/disk1        # adjust device per node
echo "/dev/nvme3n1  /data/beegfs/disk1  ext4  defaults  0 0" | sudo tee -a /etc/fstab
df -h | grep nvme
```

---

## 1. MASTER — configure and start the services

### 1a. Management daemon — `/etc/beegfs/beegfs-mgmtd.toml`

```toml
db-file = "/data/beegfs/disk1/mgmtd/mgmtd.sqlite"
tls-disable = true          # private trusted network, no inter-node encryption
license-disable = true      # skip enterprise license (no mirroring/quota)
```

> Without `tls-disable`/`license-disable`, mgmtd fails looking for
> `/etc/beegfs/cert.pem` and `/etc/beegfs/license.pem`. This is the error you hit.

### 1b. Metadata — `/etc/beegfs/beegfs-meta.conf`

```
sysMgmtdHost            = localhost
storeMetaDirectory      = /data/beegfs/disk1/meta
storeAllowFirstRunInit  = true
connAuthFile            = /etc/beegfs/conn.auth
connDisableAuthentication = false
```

### 1c. Storage — `/etc/beegfs/beegfs-storage.conf`

```
sysMgmtdHost            = localhost
storeStorageDirectory   = /data/beegfs/disk1/storage
storeAllowFirstRunInit  = true
connAuthFile            = /etc/beegfs/conn.auth
connDisableAuthentication = false
```

### 1d. Create dirs, init the mgmtd DB, create the shared secret

```bash
# Sub-directories on the 8TB disk
sudo mkdir -p /data/beegfs/disk1/{mgmtd,meta,storage}

# Initialize the management database (NOT `--config`, it's `--config-file`)
sudo /opt/beegfs/sbin/beegfs-mgmtd --init --config-file /etc/beegfs/beegfs-mgmtd.toml

# Shared connection-auth secret (required because connDisableAuthentication=false)
sudo dd if=/dev/random bs=128 count=1 | sudo tee /etc/beegfs/conn.auth > /dev/null
sudo chmod 400 /etc/beegfs/conn.auth
```

> **This `conn.auth` file is the cluster password.** Every node must have a
> byte-identical copy or it cannot join. Keep it safe — you copy it to the
> workers in step 2.

### 1e. Start services in order

```bash
sudo systemctl enable --now beegfs-mgmtd
sudo systemctl enable --now beegfs-meta
sudo systemctl enable --now beegfs-storage
sudo systemctl status beegfs-mgmtd beegfs-meta beegfs-storage   # all "active (running)"
```

### 1f. Client config + mount

```bash
sudo sed -i 's/^sysMgmtdHost.*/sysMgmtdHost = localhost/' /etc/beegfs/beegfs-client.conf
echo "/mnt/beegfs /etc/beegfs/beegfs-client.conf" | sudo tee /etc/beegfs/beegfs-mounts.conf
sudo mkdir -p /mnt/beegfs
```

Also make sure the client knows the auth file — in `/etc/beegfs/beegfs-client.conf`:

```
connAuthFile             = /etc/beegfs/conn.auth
connDisableAuthentication = false
```

**Do NOT start the client yet** if Secure Boot is on — go to step 1g first.

### 1g. Secure Boot — the client module blocker

The client is a kernel module (`beegfs.ko`) compiled on the fly. With Secure Boot
enabled, the kernel rejects the unsigned module:

```
modprobe: ERROR: could not insert 'beegfs': Key was rejected by service
```

**Option A — disable Secure Boot (recommended for a private cluster).**
Reboot into UEFI/BIOS → Security → disable Secure Boot → save & reboot. Then:

```bash
mokutil --sb-state               # should now say: SecureBoot disabled
sudo systemctl enable --now beegfs-client
mount | grep beegfs              # /mnt/beegfs should be mounted
```

**Option B — keep Secure Boot, sign the module (more work).**
You must enroll a Machine Owner Key (MOK) and re-sign `beegfs.ko` after every
build/kernel update (BeeGFS uses autobuild, not DKMS, so there's no auto-resign):

```bash
# one-time: create + enroll a key (sets a password, then reboot into MOK manager to enroll)
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der \
  -nodes -days 36500 -subj "/CN=BeeGFS module signing/"
sudo mokutil --import MOK.der
sudo reboot   # blue MOK screen on boot -> Enroll MOK -> enter password

# after each build: sign the freshly built module, then load
KVER=$(uname -r)
sudo /usr/src/linux-headers-$KVER/scripts/sign-file sha256 MOK.priv MOK.der \
  /lib/modules/$KVER/updates/fs/beegfs/beegfs.ko
sudo systemctl restart beegfs-client
```

Given the per-update maintenance, Option A is the pragmatic choice here.

---

## 2. WORKERS — configure and start (worker01/02/03)

Disks + packages were done in step 0. Now wire each worker to the master.

### 2a. Copy the shared secret from the master (run on master, once per worker)

```bash
# byte-identical copy is mandatory
scp /etc/beegfs/conn.auth admin@worker01:/tmp/conn.auth
# then on the worker:
sudo mv /tmp/conn.auth /etc/beegfs/conn.auth
sudo chmod 400 /etc/beegfs/conn.auth
```

### 2b. Storage config — `/etc/beegfs/beegfs-storage.conf`

```
sysMgmtdHost            = <MASTER_IP>      # the master's cluster IP, NOT localhost
storeStorageDirectory   = /data/beegfs/disk1/storage
storeAllowFirstRunInit  = true
connAuthFile            = /etc/beegfs/conn.auth
connDisableAuthentication = false
```

```bash
sudo mkdir -p /data/beegfs/disk1/storage
sudo systemctl enable --now beegfs-storage
sudo systemctl status beegfs-storage      # registers with master, gets a target ID
```

### 2c. Client config — `/etc/beegfs/beegfs-client.conf`

```
sysMgmtdHost            = <MASTER_IP>
connAuthFile            = /etc/beegfs/conn.auth
connDisableAuthentication = false
```

```bash
echo "/mnt/beegfs /etc/beegfs/beegfs-client.conf" | sudo tee /etc/beegfs/beegfs-mounts.conf
sudo mkdir -p /mnt/beegfs
```

### 2d. Secure Boot on each worker

Same module-rejection issue applies on every worker. Disable Secure Boot
(step 1g, Option A), then:

```bash
sudo systemctl enable --now beegfs-client
mount | grep beegfs
```

Repeat 2a–2d for worker02 and worker03 (their disks are `nvme0n1p1` and `nvme0n1`).

---

## 3. Verify the cluster (run on master)

BeeGFS 8 uses the new `beegfs` CLI (the old `beegfs-ctl` is gone). It auto-detects
the management service when `/mnt/beegfs` is mounted. If auth/TLS settings make it
complain, point it at the secret with a flag/env var (`beegfs --help`).

```bash
beegfs health capacity     # lists every node + target, with free space
beegfs pool list           # storage pools and their member targets
```

Expected: **1 metadata node** (master), **4 storage targets** (master + 3 workers,
~8 TB each), and **4 clients**. If a worker target is missing, check
`journalctl -u beegfs-storage` on that worker and the mgmtd log on the master —
usually a wrong `sysMgmtdHost` or a mismatched `conn.auth`.

Quick functional test:

```bash
echo hello | sudo tee /mnt/beegfs/test.txt
cat /mnt/beegfs/test.txt          # readable from every node
beegfs entry info /mnt/beegfs/test.txt   # shows which targets hold the chunks
```

---

## 4. (Optional) Tune striping for distributed throughput

By default files are striped RAID0 across the targets. For large sequential
genomics files (FASTQ/BAM/VCF) you often want all 4 targets and a larger chunk:

```bash
# stripe new files across all 4 targets with 1 MB chunks (applies to new files)
beegfs entry set --pattern --chunksize=1m --numtargets=4 /mnt/beegfs
```

Existing files keep their old pattern; only newly created files inherit the change.

---

## 5. Using it from Slurm

After step 3, `/mnt/beegfs` is a single shared POSIX filesystem visible identically
on the master and all 3 workers. That's exactly what Slurm jobs need for shared I/O.

- Create per-user space and fix ownership so jobs can write:
  ```bash
  sudo mkdir -p /mnt/beegfs/{scratch,data,projects}
  sudo chmod 1777 /mnt/beegfs/scratch
  ```
- Point job working directories, shared inputs, and outputs at `/mnt/beegfs`,
  e.g. in a batch script:
  ```bash
  #SBATCH --chdir=/mnt/beegfs/scratch/$USER
  ```
- Any node a job lands on sees the same paths, so no staging between nodes is needed.

### Important: no redundancy

You disabled the license, so **buddy mirroring is unavailable**, and the pool is
RAID0 across single disks. **One failed disk loses data striped across the whole
pool.** Treat `/mnt/beegfs` as fast scratch, not durable storage:

- Keep `/home`, Slurm state, and anything you can't lose on a separate, backed-up
  filesystem (e.g. NFS).
- Back up important results off the BeeGFS pool.
