# Base System Installation

## Operating System Setup

### Installation Philosophy

The goal is a **headless server** to minimize resource usage.

- **OS:** **Debian 13 (Trixie)**, but basically any Linux OS will do here.
- **GUI:** None (No GNOME/KDE). Terminal only. Optionally you can install a desktop (e.g. GNOME) but deactivate it just in case you need it sometime.

    ```bash
    sudo systemctl set-default multi-user.target
    ```

- **Filesystem:** ext4.

### "Root" Password Strategy

During the installation process, Debian asks for a `root` password.

- **Action:** We leave the root password field empty.
- **Reasoning:** This forces Debian to disable the direct root account login and automatically adds the primary user to the `sudo` group. This is easier and arguably safer than having a user with the same password as root.

### Software Selection

In the *tasksel* menu (blue screen near the end), we select:

1. SSH Server (Crucial for remote access), if you did not select this refer to [User & Network](02-network.md#ssh-server-installation) for the SSH server installation.
2. Standard System Utilities
