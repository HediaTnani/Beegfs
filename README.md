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

