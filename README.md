# Beegfs
## The Master Node 
On the master the disk `/dev/nvme3n1` is the 8T one. 

```
# 1. Format
sudo mkfs.ext4 /dev/nvme3n1

# 2. Add GPG key
sudo wget https://www.beegfs.io/release/beegfs_8.3/gpg/GPG-KEY-beegfs \
  -O /etc/apt/trusted.gpg.d/beegfs.asc

# 3. Add repository
sudo wget https://www.beegfs.io/release/beegfs_8.3/dists/beegfs-jammy.list \
  -O /etc/apt/sources.list.d/beegfs.list

# 4. Update
sudo apt install apt-transport-https
sudo apt update

# 5. Install
# Master only :
sudo apt install -y beegfs-mgmtd beegfs-meta beegfs-storage \
                    beegfs-client beegfs-utils

# 6. Create mount point
sudo mkdir -p /data/beegfs/disk1

# 7. Mount
sudo mount /dev/nvme3n1 /data/beegfs/disk1

# 8. Persist on reboot
echo "/dev/nvme3n1  /data/beegfs/disk1  ext4  defaults  0 0" | sudo tee -a /etc/fstab

# 9. Verify
df -h | grep nvme3n1

```

## Worker 01

```
# 1. Format
sudo mkfs.ext4 /dev/nvme3n1

# 2. Add GPG key
sudo wget https://www.beegfs.io/release/beegfs_8.3/gpg/GPG-KEY-beegfs \
  -O /etc/apt/trusted.gpg.d/beegfs.asc

# 3. Add repository
sudo wget https://www.beegfs.io/release/beegfs_8.3/dists/beegfs-jammy.list \
  -O /etc/apt/sources.list.d/beegfs.list

# 4. Update
sudo apt install apt-transport-https
sudo apt update

# 5. Install
# Workers only :
sudo apt install -y beegfs-storage beegfs-client beegfs-utils

# 6. Create mount point
sudo mkdir -p /data/beegfs/disk1

# 7. Mount
sudo mount /dev/nvme3n1 /data/beegfs/disk1

# 8. Persist on reboot
echo "/dev/nvme3n1  /data/beegfs/disk1  ext4  defaults  0 0" | sudo tee -a /etc/fstab

# 9. Verify
df -h | grep nvme3n1

```

## Worker 02

```
# 1. Format
sudo mkfs.ext4 /dev/nvme0n1p1

# 2. Add GPG key
sudo wget https://www.beegfs.io/release/beegfs_8.3/gpg/GPG-KEY-beegfs \
  -O /etc/apt/trusted.gpg.d/beegfs.asc

# 3. Add repository
sudo wget https://www.beegfs.io/release/beegfs_8.3/dists/beegfs-jammy.list \
  -O /etc/apt/sources.list.d/beegfs.list

# 4. Update
sudo apt install apt-transport-https
sudo apt update

# 5. Install
# Workers only :
sudo apt install -y beegfs-storage beegfs-client beegfs-utils

# 6. Create mount point
sudo mkdir -p /data/beegfs/disk1

# 7. Mount
sudo mount /dev/nvme0n1p1 /data/beegfs/disk1

# 8. Persist on reboot
echo "/dev/nvme0n1p1  /data/beegfs/disk1  ext4  defaults  0 0" | sudo tee -a /etc/fstab

# 9. Verify
df -h | grep nvme0n1p1

```

## Worker 03 

```
# 1. Format
sudo mkfs.ext4 /dev/nvme0n1

# 2. Add GPG key
sudo wget https://www.beegfs.io/release/beegfs_8.3/gpg/GPG-KEY-beegfs \
  -O /etc/apt/trusted.gpg.d/beegfs.asc

# 3. Add repository
sudo wget https://www.beegfs.io/release/beegfs_8.3/dists/beegfs-jammy.list \
  -O /etc/apt/sources.list.d/beegfs.list

# 4. Update
sudo apt install apt-transport-https
sudo apt update

# 5. Install
# Workers only :
sudo apt install -y beegfs-storage beegfs-client beegfs-utils

# 6. Create mount point
sudo mkdir -p /data/beegfs/disk1

# 7. Mount
sudo mount /dev/nvme0n1 /data/beegfs/disk1

# 8. Persist on reboot
echo "/dev/nvme0n1  /data/beegfs/disk1  ext4  defaults  0 0" | sudo tee -a /etc/fstab

# 9. Verify
df -h | grep nvme0n1

```

Now let's configure BeeGFS. 

## Master — BeeGFS configuration

`/etc/beegfs/beegfs-mgmtd.toml` - Management Daemon
This is the brain of the cluster. It knows every node, every disk, every file location.

`db-file = "/data/beegfs/disk1/mgmtd/mgmtd.sqlite"`

BeeGFS stores all cluster information in a SQLite database. We put it on the 8TB disk (`/data/beegfs/disk1`) and not on the OS disk so it doesn't fill up the system.

`tls-disable = true`

BeeGFS 8 requires TLS certificates by default for encrypted communication between nodes. We disable it because we are on a private cluster network - no need for encryption between trusted nodes.

`license-disable = true`

BeeGFS 8 has enterprise features that require a paid license (quota management, mirroring, etc). We disable the license check because we don't need those features. Basic storage pooling is free.

`/etc/beegfs/beegfs-meta.conf` - Metadata Server
The metadata server tracks where every file and directory lives across the cluster. It doesn't store file content, only :

-  filenames
-  permissions
-  which storage targets hold the data

`sysMgmtdHost = localhost`

Tells the metadata server where the management daemon is. Since both run on master, we use `localhost`.

`storeMetaDirectory = /data/beegfs/disk1/meta`

Where metadata is physically stored on disk. BeeGFS creates this directory automatically on first start.

`/etc/beegfs/beegfs-storage.conf` - Storage Server
The storage server is what actually holds your data — FASTQs, BAMs, VCFs, everything. It splits files into chunks and stores them on the disk.

`sysMgmtdHost = localhost`

Same as meta — points to the management daemon on master.

`storeStorageDirectory = /data/beegfs/disk1/storage`

Where file chunks are stored on the 8TB disk. When you write a file to `/mnt/beegfs`, the data physically ends up here.


