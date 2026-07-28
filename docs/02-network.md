# Initial Configuration

## User Management

Since we skipped the root password, our created user (let's call him `joachim` or
similar) has sudo rights.

### Verifying Sudo Access

To verify that the user can perform administrative tasks:

```bash
# Check if user belongs to sudo group 
groups

# Test sudo access (should ask for user password) 
sudo apt update
```

### Setting the Hostname

To ensure the server is easily identifiable on the network:

```bash
sudo hostnamectl set-hostname mediaserver
```

## Dedicated Service User Setup

For security and clean permission management, we do not run the
applications as the administrative user. Instead, we create a dedicated
system user (named `media`) for all Docker containers.

### Creating the Media User

We create a system user with a specific UID and disable login
capabilities for security.

```bash
# Create user 'media' with UID 13000, create matching group, no login, no home
sudo useradd -u 13000 -U -M -s /bin/false media

# Verify creation
id media
# Output: uid=13000(media) gid=13000(media) groups=13000(media)
```

- **PUID:** 13000

- **PGID:** 13000

- `-U`: Creates a group with the same name and ID.

- `-M`: Do not create a home directory.

- `-s /bin/false`: Disable login.

### Granting Access to the Admin User

To ensure that we (the logged-in admin) can still manually manage files
in the storage pool, we add our current user to the new group.

```bash
sudo usermod -a -G media joachim
```

### Applying Filesystem Permissions

We transfer ownership of the data directory to the new user and group.
We set permissions to `775`, allowing full access to both the service user
and group members (us).

```bash
# Change ownership to media:media 
sudo chown -R media:media /mnt/data

# Set permissions (User=RWX, Group=RWX, Others=RX) 
sudo chmod -R 775 /mnt/data/
```

*NB: r = 4, w = 2, x = 1*

### Enforcing Read-Only Access for Plex

While services like Sonarr need write access to manage files, Plex
should not modify media files.

Instead of complex filesystem ACLs, we will handle this at the Docker
level later.

- All containers run as `PUID=13000`.

- Sonarr/Radarr mount volumes as **Read-Write**.

- Plex mounts the media volume with the `:ro` (Read-Only) flag in Docker
    Compose (see [Docker & Services](04-docker.md)).

## Network Configuration

Since this is a server, it needs a static IP. In my case I added the
following interface:

```bash
# Edit the file /etc/network/interfaces
sudo vim /etc/network/interfaces
```
And add:

```conf
# The primary network interface
allow-hotplug enp4s0
iface enp4s0 inet static
	address 192.168.0.50
	netmask 255.255.255.0
	gateway 192.168.0.1
	dns-nameservers 1.1.1.1 8.8.8.8
```

- `allow-hotplug`: Tells the system to start the network interface enp4s0 as soon as
    the kernel detects the hardware is plugged in or available, prevents
    being stuck at boot as possible with .

- `inet`: Use IPv4 (for IPv6 use `inet6`).

- `gateway`: IP address of the router.

Following commands are useful:

```bash
# 1. Find the interface name (e.g., eth0, enp4s0, eno1) 
ip a

# 2. Find the gateway IP (default via \...) 
ip r
```

## Language & Locale

We set the system language to English but kept the keyboard layout Swiss
(or German) as I am using a Swiss keyboard.

```bash
# Reconfigure Locales (Set to en_US.UTF-8)
sudo dpkg-reconfigure locales

# Check Keyboard layout (Should be 'ch' or 'de') 
cat /etc/default/keyboard
```

## SSH Server Installation

If the SSH server was not selected during the initial OS installation,
it must be installed manually.

### Installation via APT

```bash
# 1. Install the package 
sudo apt update 
sudo apt install openssh-server

# 2. Enable and start the service 
sudo systemctl enable --now ssh

# 3. Verify the status 
sudo systemctl status ssh
```

### Connecting to the Server

From a remote machine (Windows PowerShell or Linux Terminal):

```bash
# Syntax: ssh username@ip-address 
ssh user@192.168.0.50
```

## SSH Key Authentication (Passwordless)

To increase security and convenience, we configure SSH Key
Authentication. This allows logging in without typing a password, using
a cryptographic key pair instead.

### 1. Generating a Key Pair (On Client)

**Note:** These commands are to be executed on your **Laptop/PC**. We
use the `ed25519` algorithm.

```bash
# Run this on your local machine 
ssh-keygen -t ed25519 -C "my-laptop"

# Press Enter to accept the default file location.
```

### 2. Copying the Public Key to the Server

We transfer the public part of the key to the server using `ssh-copy-id`.

```bash
# Replace user and IP with your server details 
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.50
```

### 3. Verifying and Hardening

Try to log in again. You should not be asked for a password. Once
verified, we disable password authentication to prevent brute-force
attacks.

```bash
# Open the SSH config on the SERVER 
sudo vim /etc/ssh/sshd_config
```

Find and modify the following lines:

```conf
# Disable root login (Security Best Practice)
PermitRootLogin no

# Disable password authentication 
PasswordAuthentication no
ChallengeResponseAuthentication no 
UsePAM yes
```

Restart SSH to apply:

```bash
sudo systemctl restart ssh
```

## Firewall Setup (UFW)

To secure the host system, we use **UFW** (Uncomplicated Firewall).
While Docker manages its own network rules deeply in the system
(iptables), UFW is essential to protect the host services (like SSH) and
serves as a second layer of defense.

### Installation and Default Policies

First, we install UFW and define the fundamental behavior: block all
incoming traffic by default, but allow all outgoing traffic (so the
server can download updates).

```bash
# Install UFW sudo apt update 
sudo apt install ufw

# Set defaults (Do not enable yet) 
sudo ufw default deny incoming 
sudo ufw default allow outgoing
```

### Defining Rules

We distinguish between critical system access (SSH) and application
access (Plex, Web interfaces).

#### 1. Securing SSH (Crucial)

We limit SSH access strictly to the local home network (LAN). This
prevents anyone from the outside internet from even trying to log in,
even if they had the key.

**Note:** Replace `192.168.0.0/24` with your actual subnet.

```bash
# Allow SSH only from local LAN 
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
```

#### 2. Application Ports (Pre-Configuration)

Although we haven't installed Docker yet, we will pre-configure the
firewall for the planned services.

**Public Access (Internet & LAN):**

```bash
# Plex (Streaming) - Standard Port 32400 
sudo ufw allow 32400/tcp
```

**Private Admin Access (LAN Only):** Management interfaces like Sonarr,
Radarr, or Portainer should not be exposed to the open internet. We
restrict them to the local network.

```bash
# CasaOS / Dashboard (Port 80), only if you want to use this
sudo ufw allow from 192.168.0.0/24 to any port 80 proto tcp

# Overseerr/Seerr (Request Management - Port 5055) 
sudo ufw allow from 192.168.0.0/24 to any port 5055 proto tcp

# Sonarr (8989) & Radarr (7878) 
sudo ufw allow from 192.168.0.0/24 to any port 8989 proto tcp 
sudo ufw allow from 192.168.0.0/24 to any port 7878 proto tcp

# NZBGet / SABnzbd (6789) 
sudo ufw allow from 192.168.0.0/24 to any port 6789 proto tcp
```

**Note**: Since we are using Docker to deploy these tools, we don't strictly need these UFW rules; by default Docker manipulates `iptables` directly and bypasses UFW. They are listed here for completeness and for any services you run directly on the host.

### Enabling the Firewall

**Warning:** Ensure the SSH rule (Section [SSH Server Installation](02-network.md#ssh-server-installation)) matches your current IP
address, or you might lock yourself out immediately.

```bash
# Enable UFW 
sudo ufw enable

# Check status and numbered rules
sudo ufw status numbered
```

