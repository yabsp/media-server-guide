# Energy Optimization

Running a server 24/7 impacts the electricity bill. Since a media server
stays idle for most of the day, we optimize the hardware to enter
low-power states when not in use.

## Hard Drive Spindown

Mechanical hard drives consume approx. 6–10 Watts when spinning, but
less than 1 Watt in standby. Since we use **MergerFS** (instead of RAID
5/ZFS), we have a massive advantage: we can spin down drives
individually. When watching a movie, only the specific drive containing
that file needs to wake up; the others remain asleep.

### Configuration via hdparm

We use `hdparm` to set a timeout. If a drive is inactive for a specific time, the
motor stops.

```bash
sudo apt install hdparm -y
```

To ensure the configuration persists across reboots and works even if
drive letters change (e.g., /dev/sdb becomes /dev/sdc), we identify
drives by their UUID.

```bash
# 1. Find the UUIDs of your HDDs (ignore the SSD/Boot drive!) 
lsblk -f

# 2. Edit the configuration 
sudo vim /etc/hdparm.conf
```

Add a block for **each** mechanical HDD at the end of the file:

```conf
# HDD 1
/dev/disk/by-uuid/YOUR-UUID-HERE {
    # 241 = 30 min, 242 = 1 hour
    spindown_time = 241
    # Aggressive power management
    apm = 127
}
# HDD 2
/dev/disk/by-uuid/NEXT-UUID-HERE {
    spindown_time = 241
    apm = 127
}
```

*Note:* A spindown time of 30–60 minutes is recommended. Setting it
too short (e.g., 5 mins) causes frequent start-stop cycles, which wears
out the drive mechanics.

## Verification

To check if your drives are actually sleeping without waking them up:

```bash
sudo hdparm -C /dev/sd?
```

*Output:*

- `drive state is: active/idle` -> Drive is spinning (consuming power).

- `drive state is: standby` -> Drive is sleeping (saving power).
