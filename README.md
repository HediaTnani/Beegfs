# Beegfs

On the master 

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


```
