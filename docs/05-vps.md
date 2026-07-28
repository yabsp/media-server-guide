# VPS Gateway

## VPS Initial Configuration

We use a lightweight VPS (1 vCPU, 2GB RAM) running Debian. We utilize
the hosting provider's hardware firewall instead of a local firewall
(UFW) as it is easier and there are less possibilities for issues.

### IP Forwarding on Linux

By default, the Linux kernel acts strictly as a host, not a router. This
means it instantly drops any network packet that is not specifically
addressed to its own IP address.

For our VPN Gateway setup, this behavior is a blocker. The VPS needs to
act as a bridge:

-   It receives traffic from the internet (e.g., a request for Plex).

-   It must pass this traffic from the physical network card (`eth0`) to the
    virtual VPN interface (`wg0`).

Enabling `ip_forward` instructs the kernel to permit packets to traverse between
network interfaces. We use the `sysctl.d` directory approach for
configuration persistence.

### Server Setup

```bash
# 1. Update and Install basics 
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard curl git

# 2. Enable IP Forwarding
# We create a dedicated config file in /etc/sysctl.d/ instead of editing the main file.
# This is more robust on minimal Linux images. 
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf

# Apply the changes immediately 
sysctl --system
```

## WireGuard Configuration

We are using WireGuard to connect our media server with our VPS. There
are some guides out there that ease the pain when setting up the whole
thing - for example [one for Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04) that can be used for other
versons and distros as well.
WireGuard uses asymmetric cryptography. We must generate key pairs
(Private/Public) on both the VPS and the Home Server.

```bash
# Run this on BOTH servers to generate keys: 
wg genkey | tee privatekey | wg pubkey > publickey

# Display keys for configuration (Copy these securely!) 
cat privatekey
cat publickey
```

### VPS Server Config (/etc/wireguard/wg0.conf)

On the VPS, we configure the interface and define the Home Server as a
Peer.

```conf
[Interface]
# Internal Tunnel IP of the VPS
Address = 10.10.10.1/24
# Custom Port defined in Firewall 
ListenPort = 51821 
PrivateKey = <VPS_PRIVATE_KEY>

# IP Masquerading Rules (NAT)
# These rules ensure traffic leaving the VPS interface (eth0) appears to come from the VPS IP, \
# enabling the return path for traffic originating from the Home Server. 
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# The Home Server 
PublicKey = <HOME_PUBLIC_KEY> 
# The single IP allowed for the Home Server 
AllowedIPs = 10.10.10.2/32
```

```bash
# Start the WireGuard Service on VPS 
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

## Home Server Client Config

The home server acts as the client. It initiates the connection to the
VPS and uses `PersistentKeepalive` to keep the NAT session open through the home router.

```bash
# 1. Install WireGuard on Home Server 
sudo apt update && sudo apt install wireguard -y

# 2. Configure the Interface 
sudo vim /etc/wireguard/wg0.conf
```

```conf
[Interface] 
# The Internal IP of the Home Server
Address = 10.10.10.2/24 
PrivateKey = <HOME_PRIVATE_KEY>

[Peer] # Connection details for the VPS 
PublicKey = <VPS_PUBLIC_KEY> 
# The Public IP of the VPS and the custom port defined earlier 
# Use this port or any other port you defined (keep in mind that a random port is better)
Endpoint = <VPS_PUBLIC_IP>:51821 
# Route only traffic destined for the VPS internal IP through the tunnel 
AllowedIPs = 10.10.10.1/32 
# Critical: Sends a keepalive packet every 25s to prevent the home router 
# from closing the connection due to inactivity.
PersistentKeepalive = 25
```

```bash
# 3. Start the Tunnel 
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# 4. Verification 
# Ping the VPS internal IP to confirm connectivity
ping 10.10.10.1
```

## Reverse Proxy Setup (VPS)

With the tunnel active, we deploy [Nginx Proxy Manager (NPM)](https://nginxproxymanager.com/guide/) on the VPS.
You can find more information in the NPM
documentation. NPM will terminate SSL
connections from the internet and route traffic through the WireGuard
tunnel to the home server.

```bash
# 1. Install Docker on VPS 
curl -fsSL https://get.docker.com | sh

# 2. Create Directory and Config 
mkdir -p ~/nginx-proxy && cd ~/nginx-proxy 
vim docker-compose.yml
```

```conf
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'     # Public HTTP Port
      - '443:443'   # Public HTTPS Port
      - '81:81'     # Admin Web Interface (setup only!)
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      DISABLE_IPV6: 'true'
    depends_on:
      - crowdsec
```

```bash
# 3. Start the Proxy 
docker compose up -d
```

### Configuration Steps

1.  **DNS Record:** Point an A-Record (e.g., `plex.yourdomain.com`) to the VPS Public IP.

2.  **Access Admin UI:** Navigate to `http://<VPS_IP>:81` (Default: admin@example.com /
    changeme).

3.  **Create Proxy Host:**

    -   **Domain Name:** plex.yourdomain.com

    -   **Scheme:** http

    -   **Forward Host:** 10.10.10.2 (Home Server VPN IP)

    -   **Forward Port:** 32400

    -   **Websockets:** Enabled (Critical for Plex)

4.  **SSL Tab:** Request a new Let's Encrypt certificate and enable
    "Force SSL".

## Plex Configuration (Home)

Finally, instruct Plex to use the new public URL.

1.  Open Plex Settings > Network.

2.  Set **Custom server access URLs** to: `https://plex.yourdomain.com:443`

3.  Disable and re-enable Remote Access to force a config refresh.

## Security Hardening

Once the Proxy Host is configured and confirmed working, we must secure
the VPS by closing the administration port.

We switch the SSL validation method to "DNS Challenge". This would
allow us to close Port 80 completely, leaving only the encrypted HTTPS
port open.
*Note:* We are using cloudflare here.

### Cloudflare API Token

1. Go to Cloudflare Dashboard > My Profile > API Tokens.

2. Create a token using the **Edit zone DNS** template for your specific domain.

3. Copy the generated Token.

### Configure NPM for DNS Challenge
In Nginx Proxy Manager, edit the Proxy Host:

- **SSL Tab:** Check "Use a DNS Challenges".

- **Provider:** Cloudflare.

- **Credentials:** Replace the token placeholder:

    ```conf
    dns_cloudflare_api_token = <YOUR_CLOUDFLARE_TOKEN>
    ```

### Final Firewall Rules

With DNS validation active, we can close Port 80. The only open ingress
ports are now for the VPN tunnel and the encrypted HTTPS stream.

| Port  | Protocol | Status | Reason                         |
|------|----------|--------|--------------------------------|
| 22   | TCP      | Allow  | SSH Management                 |
| 443  | TCP      | Allow  | HTTPS Streaming                |
| 51821| UDP      | Allow  | WireGuard Tunnel               |
| 80   | TCP      | DENY   | Not needed (DNS-01 used)       |
| 81   | TCP      | DENY   | Admin UI (Tunnel only)         |

### Secure Admin Access via SSH Tunnel

To access the Nginx Proxy Manager interface in the future without
exposing the port to the internet, we use local port forwarding.

```bash
# Run this on your local machine to create a secure tunnel
# Syntax: ssh -L <LocalPort>:127.0.0.1:<RemotePort> <User>@<Host> 
ssh -L 8888:127.0.0.1:81 debian@<VPS_PUBLIC_IP>
```

Now, open your browser and navigate to: **http://localhost:8888**

The traffic is now encrypted inside the SSH session, making it secure
even over public WiFi.

## Advanced Security: CrowdSec (IPS)

Since we bypass Cloudflare's proxy (Orange Cloud) to allow direct
streaming, the VPS IP address is exposed to the public internet. To
protect the server against scanners, brute-force attacks, and botnets,
we implement **CrowdSec**.

### What is CrowdSec?

[CrowdSec](https://docs.crowdsec.net/u/getting_started/installation/linux/) is a modern, open-source Intrusion Prevention System
(IPS). Unlike legacy tools like Fail2Ban,
it works on a collaborative model:

1. **Detect (The Agent):** It analyzes log files (Nginx, SSH, System) in real-time to detect aggressive behavior.

2. **Protect (The Bouncer):** Once a threat is detected, the "Bouncer" interacts with the firewall to drop the connection immediately.

3. **Share (The Crowd):** When your server blocks an IP for malicious activity, this information is shared with the global CrowdSec community. Conversely, your server automatically receives a blacklist of IPs that have attacked other users, blocking them before they even touch your server.

### Architecture Setup
We use a hybrid setup:

- **The Agent** runs inside a **Docker container**. It reads the logs.

- **The Bouncer** runs directly on the **Host System (VPS)**. It manages the `iptables` firewall rules.

### 1. Docker Compose Modification (The Agent)
We modify the `docker-compose.yml` to add CrowdSec. **Critical Changes:**

- We map the host's `auth.log` to protect SSH.

- We expose Port `8080` to localhost so the Host-Bouncer can talk to the Docker-Agent.

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # Public HTTP Port
      - '80:80'
      # Public HTTPS Port
      - '443:443'
      # Admin Web Interface (setup only!)
      - '81:81'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      DISABLE_IPV6: 'true'
    depends_on:
      - crowdsec

  crowdsec:
    image: crowdsecurity/crowdsec:latest
    restart: unless-stopped
    environment:
      # load Nginx and SSHD rules
      - COLLECTIONS=crowdsecurity/nginx-proxy-manager crowdsecurity/sshd
    volumes:
      # Nginx logs
      - ./data/logs:/var/log/npm-logs:ro
      # ssh logs of host to container
      - /var/log/auth.log:/var/log/host-auth.log:ro
      # CrowdSec data
      - ./crowdsec-db:/var/lib/crowdsec/data
      - ./crowdsec-config:/etc/crowdsec
    ports:
      - 8080:8080
    security_opt:
      - no-new-privileges=true

```

### 2. Log Acquisition Configuration

CrowdSec needs to know exactly which files to analyze. We create `crowdsec-config/acquis.yaml` mapping the Nginx logs and the system authentication logs.

```conf
# --- Nginx proxy manager logs ---
filenames:
  - /var/log/npm-logs/proxy-host-*_access.log
  - /var/log/npm-logs/proxy-host-*_error.log
  - /var/log/npm-logs/fallback_access.log
  - /var/log/npm-logs/fallback_error.log
labels:
  type: nginx-proxy-manager

# --- SSH host logs ---
filenames:
  - /var/log/host-auth.log
labels:
  type: syslog

```

Apply the changes by restarting the stack:

```bash
docker compose up -d 
docker compose restart crowdsec
```

### 3. Installing the Enforcer (Firewall Bouncer)

The Docker container detects the attacks, but it cannot control the
host's firewall directly. We install the `iptables-bouncer` on the VPS host system.

```bash
# 1. Add Repository and Install 
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
sudo apt install crowdsec-firewall-bouncer-iptables -y

# 2. Generate API Key (inside Docker) 
# The bouncer needs this key to authenticate with the Docker agent 
docker exec -t nginx-proxy-crowdsec-1 cscli bouncers add firewall-bouncer 
# > Copy the API KEY generated here!

# 3. Configure Bouncer 
sudo vim /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

Edit the configuration file to point to the local Docker port:

```conf
mode: iptables
update_frequency: 10s
log_mode: file
log_dir: /var/log/
log_level: info
# Point to localhost port 8080 (exposed by Docker)
api_url: http://127.0.0.1:8080/
# Paste your generated key here
api_key: <YOUR_GENERATED_KEY>
disable_ipv6: true
```

```bash
# 4. Enable Protection 
sudo systemctl restart crowdsec-firewall-bouncer

# 5. Verify Status
# Check if the bouncer is registered in the container 
docker exec -t nginx-proxy-crowdsec-1 cscli bouncers list
```

### Enrollment & Verification

By default, a fresh CrowdSec installation has an empty blocklist. To
immediately receive the full "Community Blocklist" (approx. 20,000+
malicious IPs) and visualize attacks, we enroll the instance in the
CrowdSec Console.

1. Go to [https://app.crowdsec.net](https://app.crowdsec.net) and create a free account.

2. Select "Enroll my first instance" to receive an enrollment key.

3. Run the enrollment command inside the Docker container:

```bash
# Replace <KEY> with the code from the website 
docker exec -t nginx-proxy-crowdsec-1 cscli console enroll <KEY>

# Restart to force an immediate blocklist update
docker compose restart crowdsec
```

**Verification:** The download of the global blocklist (CAPI) takes
about 5-10 minutes. Afterward, verify that your server has protected
itself against thousands of known bad actors:

```bash
# Check the number of banned IPs from the Community List 
docker exec -t nginx-proxy-crowdsec-1 cscli decisions list --origin CAPI -a | grep -v "No decisions" | wc -l
```

*Expected Result:* A number greater than **15,000**. If the result is 0
or 1, wait a few more minutes and try again.

