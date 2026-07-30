# Storage Setup (MergerFS)

We utilize **MergerFS** to combine multiple physical hard drives into a
single logical storage pool. Unlike RAID, this allows drives of
different sizes to be mixed, and data remains readable on the individual
drives if the pool fails. Speed is a minor tradeoff here, but it should be
no problem for what we are planning with our server.

## Prerequisites

We need the necessary software packages to handle the pool and different
filesystems (like exFAT for migration).

```bash
# Install MergerFS and filesystem tools 
sudo apt update 
sudo apt install mergerfs fuse3 exfatprogs
```

## Disk Preparation

### Identifying Drives

Before formatting, we must identify the correct device identifiers
(e.g., `/dev/sda`, `/dev/sdb`).

```bash
# List block devices and their file systems 
lsblk -f
```

### Formatting to ext4

Linux works best with the `ext4` filesystem. When preparing a new drive for
media storage, we use specific flags to maximize usable space.

**Warning:** This command wipes all data on the target drive!

```bash
# Format the drive (Replace sdX with actual identifier)
sudo mkfs.ext4 -m 0 -L disk1 /dev/sdX
```

**Parameter Explanation:**

- `-m 0`: By default, ext4 reserves 5% of the disk space for the root
    user/system services. On a 10TB drive, this would waste 500GB. Since
    this is a data drive, we set the reserved blocks to 0%.

- `-L label`: Assigns a label (name) to the partition, making it easier to
    identify later.

## Mounting Drives

We do not mount drives using their device name (e.g., `/dev/sdb`) because these
names can change after a reboot. Instead, we use the **UUID**.

### Getting the UUID

```bash
# Display UUIDs for all partitions 
sudo blkid
```

*Copy the UUID string (without quotes) for the next step. If you are
doing this in a shell-only environment but have a mouse available, you
might want to use [`gpm`](https://linux.die.net/man/8/gpm) to copy and paste.*

### Configuring fstab

We edit `/etc/fstab` to mount the drives automatically at boot.

```bash
# Create mount points first 
sudo mkdir -p /mnt/disk1 
sudo mkdir -p /mnt/disk2

# Edit the file system table 
sudo vim /etc/fstab
```

Add the following lines for the physical disks:

```conf
# Physical Disk 1 
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /mnt/disk1 ext4 defaults 0 2

# Physical Disk 2 
UUID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy /mnt/disk2 ext4 defaults 0 2
```

## The MergerFS Pool

Finally, we define the virtual pool that combines `/mnt/disk1`, `/mnt/disk2`, etc., into `/mnt/data`.

Add this line to the end of `/etc/fstab`:

```conf
# MergerFS Pool
# Syntax: branches mount_point type options dump pass 
/mnt/disk* /mnt/data fuse.mergerfs defaults,nonempty,allow_other,use_ino,category.create=mfs,minfreespace=10G,fsname=mergerfs 0 0
```

### Understanding the MergerFS Arguments

- `/mnt/disk*`: The source. The asterisk (\*) tells MergerFS to include **any**
    folder starting with "disk" in that directory. This makes adding
    new drives (disk3, disk4) easy later.

- `defaults`: Standard mount settings.

- `allow_other`: **Crucial for Docker.** Allows users other than the one who
    mounted the filesystem (root) to access it. Without this, Plex would
    not see the files.

- `use_ino`: Passes the original inode numbers from the physical disks.
    Important for hardlinks to work correctly across the pool.

- `category.create=mfs`: **"Most Free Space"**. When a new file is written to the pool,
    MergerFS will write it to the physical disk that has the most
    available space. This keeps drive usage balanced.

- `minfreespace=10G`: Prevents a drive from becoming 100% full, which can cause
    filesystem corruption or crashes. If a drive has less than 10GB
    left, MergerFS moves to the next drive.

## Applying Changes

Once `fstab` is saved, create the pool directory and mount everything.

```bash
# Create the pool mount point 
sudo mkdir -p /mnt/data

# Mount all filesystems listed in fstab 
sudo mount -a

# Verify the pool size (Should be sum of disk1 + disk2) 
df -h /mnt/data
```

