# Server Sorcery 101 – Documentation

## 1. Preparation  
**Objective**  
- Set up the foundation by preparing virtualization software, OS images, and a network plan.  

**Actions Taken**  
- **Installed VirtualBox**  
  - Downloaded from the official Oracle website.  
  - Installed on host machine (Windows 10).  

- **Downloaded Ubuntu Server ISO**  
  - Version: Ubuntu Server 22.04 LTS (latest LTS at the time).  
  - Source: Ubuntu official downloads page.  
  - Verified ISO by booting into the installer during VM creation.  

- **Allocated resources for each VM**  
  - `lb-server` → 2 CPU, 4 GB RAM  
  - `web1-server` → 2 CPU, 4 GB RAM  
  - `web2-server` → 2 CPU, 4 GB RAM  
  - `app-server` → 2 CPU, 4 GB RAM  

- **Planned network configuration**  
  - All VMs on the same **Host-Only network**.  
  - Only `lb-server` has Internet access (via NAT).  
  - Static IP scheme:  
    - Load Balancer → `192.168.56.10`  
    - Web1 → `192.168.56.11`  
    - Web2 → `192.168.56.12`  
    - App → `192.168.56.13`  

**Challenges & Fixes**  
- Initially all VMs had NAT enabled → this would expose them.  
- Fixed by planning: only keep NAT for `lb-server`, others only Host-Only.  

---

## 2. VM Creation and Hostnames  
**Objective**  
- Create 4 Ubuntu Server VMs, install OS, and assign hostnames.  

**Actions Taken**  
- **Created the first VM (`lb-server`)**  
  - Attached the Ubuntu Server ISO.  
  - Installed Ubuntu Server (minimal install).  
  - After setup, updated packages:  
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```  

- **Cloned the base VM**  
  - VirtualBox → right click → *Clone*.  
  - Created:  
    - `web1-server`  
    - `web2-server`  
    - `app-server`  
  - Each got the same CPU and RAM as planned.  

- **Assigned hostnames**  
    ```bash
    sudo hostnamectl set-hostname lb-server
    sudo hostnamectl set-hostname web1-server
    sudo hostnamectl set-hostname web2-server
    sudo hostnamectl set-hostname app-server
    ```  

- **Verified hostnames**  
    ```bash
    hostnamectl
    ```  

**Challenges & Fixes**  
- Needed to ensure each clone got a **unique MAC address** in VirtualBox to avoid network conflicts.  

---

## 3. Networking  
**Objective**  
- Assign static IPs and confirm VM connectivity.  

**Actions Taken**  
- **Edited Netplan configs** (`/etc/netplan/00-installer-config.yaml`)  

Example – `lb-server`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.10/24]
```

Example – `web1-server`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.11/24]
```

(Similar for `web2-server` → `.12`, `app-server` → `.13`)  

- **Applied changes**  
    ```bash
    sudo chmod 600 /etc/netplan/00-installer-config.yaml
    sudo netplan generate
    sudo netplan apply
    ```  

- **Verified IPs**  
    ```bash
    ip a
    ```  
  - Confirmed `192.168.56.10–13` were bound to `enp0s8`.  

- **Tested connectivity**  
    ```bash
    ping 192.168.56.11
    ping 192.168.56.12
    ping 192.168.56.13
    ```  
  - All responded successfully.  

- **Checked Internet**  
  - Only `lb-server` could ping `8.8.8.8` or `google.com`.  
  - Others could not → as intended.  

**Challenges & Fixes**  
- Got a permissions warning:  
    ```
    WARNING: Permissions for /etc/netplan/00-installer-config.yaml are too open
    ```  
  - Fixed with:  
    ```bash
    sudo chmod 600 /etc/netplan/00-installer-config.yaml
    ```  
- Sometimes had to manually bring the interface up:  
    ```bash
    sudo ip link set enp0s8 up
    ```  

---

## 4. Linux Administration & Security Setup
**Objective**  
- Secure each VM by creating a dedicated user, configuring SSH, firewalls, umask, and automatic updates.

**Actions Taken**  
- **Created `devops` user on all servers**
    ```bash
    sudo adduser devops
    sudo usermod -aG sudo devops
    ```
    - Confirmed with:
    ```bash
    grep devops /etc/passwd
    groups devops
    ```

- **SSH Security**
    - Generated SSH key on host machine:
    ```bash
    ssh-keygen -t ed25519
    ```
    - Copied key to each VM:
    ```bash
    ssh-copy-id devops@<VM_IP>
    ```
    - Edited `/etc/ssh/sshd_config` on each server:
        ```
        PermitRootLogin no
        PasswordAuthentication no
        Protocol 2
        AllowUsers devops
        ```
    - Restarted SSH service:
    ```bash
    sudo systemctl restart ssh
    ```

- **Disabled unused interfaces**
    - Checked with:
    ```bash
    ip link show
    ```
    - Brought down unused adapters (e.g., enp0s3 on non-LB servers):
    ```bash
    sudo ip link set enp0s3 down
    ```

- **Configured Firewall (UFW)**
    ```bash
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    # On lb-server only:
    sudo ufw allow http
    sudo ufw allow https
    sudo ufw enable
    ```

- **Set secure umask**
    - Edited `/etc/profile` to include:
    ```
    umask 027
    ```

- **Configured automatic security updates**
    ```bash
    sudo apt install unattended-upgrades
    sudo dpkg-reconfigure --priority=low unattended-upgrades
    ```
    - Verified via `/etc/apt/apt.conf.d/20auto-upgrades`:
    ```
    APT::Periodic::Update-Package-Lists "1";
    APT::Periodic::Unattended-Upgrade "1";
    ```

**Challenges & Fixes**  
- SSH keys initially mismatched due to special characters in username path → fixed by re-copying keys and adjusting config.  
- Warnings in netplan configs were resolved by setting correct file permissions.  
- Confirmed UFW rules allow only necessary services.  

---

## 5. Load Balancer Setup
**Objective**  
- Configure `lb-server` with Nginx as a load balancer.

**Actions Taken**
- Installed Nginx:
    ```bash
    sudo apt install nginx -y
    ```
- Edited `/etc/nginx/sites-available/default`:
    ```nginx
    upstream backend_servers {
        server 192.168.56.11;
        server 192.168.56.12;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
- Restarted Nginx:
    ```bash
    sudo systemctl restart nginx
    ```
- Verified in browser → `localhost:8080` rotated between `Hello from Web1` and `Hello from Web2`.

---

## 6. Web Servers Setup
**Objective**  
- Configure `web1-server` and `web2-server` with Nginx serving static HTML pages.

**Actions Taken**
- Installed Nginx:
    ```bash
    sudo apt install nginx -y
    ```
- Replaced default index.html on each server:
    ```bash
    echo '<h1>Hello from Web1</h1>' | sudo tee /var/www/html/index.html
    echo '<h1>Hello from Web2</h1>' | sudo tee /var/www/html/index.html
    ```
- Verified locally:
    ```bash
    curl http://127.0.0.1
    ```
    - Web1 returned `Hello from Web1`
    - Web2 returned `Hello from Web2`

**Challenges & Fixes**  
- Initially UFW blocked HTTP traffic. Fixed by allowing port 80:
    ```bash
    sudo ufw allow 80/tcp
    ```

---

## 7. App Server Setup
**Objective**  
- Deploy a simple app server and configure web servers to proxy requests if needed.

**Actions Taken**
- Created app directory:
    ```bash
    sudo mkdir -p /opt/app
    echo '<h1>Hello from App Server</h1>' | sudo tee /opt/app/index.html
    ```
- Started simple Python HTTP server as a systemd service:
    ```bash
    sudo nano /etc/systemd/system/appserver.service
    ```
    ```ini
    [Unit]
    Description=Simple Python HTTP server (no deps)

    [Service]
    ExecStart=/usr/bin/python3 -m http.server 5000 --directory /opt/app
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now appserver
    ```
- Verified service:
    ```bash
    systemctl status appserver
    curl http://127.0.0.1:5000
    ```
    → Returned `Hello from App Server`
- Allowed firewall traffic on port 5000:
    ```bash
    sudo ufw allow from 192.168.56.0/24 to any port 5000 proto tcp
    ```
- Updated web servers’ Nginx config to proxy traffic to the app server:
    ```nginx
    location /app/ {
        proxy_pass http://192.168.56.13:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    ```
- Restarted Nginx:
    ```bash
    sudo systemctl reload nginx
    ```
- Verified via browser and curl:
    - Accessing `/app/` on web1/web2 returned `Hello from App Server`.

**Challenges & Fixes**
- At first, `nginx -t` failed due to putting `proxy_pass` outside of `location {}`. Fixed by moving directives into the correct block.
- UFW initially blocked port 5000; resolved by explicitly allowing it.

---

## 8. Documentation Process
Throughout the project, every step was documented in detail, including commands executed, configuration changes, and justifications for each decision.  
Logs were maintained in parallel with implementation to ensure accuracy and reproducibility.  
Challenges encountered (such as firewall rules, WireGuard key mismatches, and monitoring installation issues) were noted with their solutions.  
This process guarantees that another DevOps engineer can replicate the environment from scratch using only this documentation.

## 9. Extra Challenges

### 9A. Fail2Ban (Intrusion Prevention)
Fail2Ban was installed and configured to protect against brute-force SSH login attempts.  
- Installed with: `sudo apt install fail2ban -y`  
- Default jail configuration was reviewed at `/etc/fail2ban/jail.conf` and customized in `/etc/fail2ban/jail.local`.  
- A rule for SSH was enabled, banning IPs after 5 failed attempts for 10 minutes.  
- Service was enabled with `sudo systemctl enable --now fail2ban`.  
This ensures automated intrusion prevention and reduces attack surface.

### 9B. WireGuard VPN
A secure private network was established using WireGuard between `lb-server` and `web1-server`.  
- Keys were generated with `wg genkey | tee privatekey | wg pubkey > publickey`.  
- Each server’s `/etc/wireguard/wg0.conf` was configured with `[Interface]` (private key, address, listen port) and `[Peer]` (public key, endpoint, allowed IPs).  
- Firewall rules were added to allow UDP port 51820 and only allow access from VPN subnet `10.0.0.0/24`.  
- Handshake verification was confirmed with `sudo wg show`, showing live transfer of packets.  
This VPN ensures encrypted communication between servers.

### 9C. Netdata Monitoring
Netdata was installed on the load balancer to provide real-time performance monitoring.  
- Installed with: `bash <(curl -Ss https://my-netdata.io/kickstart.sh)`  
- Service confirmed running with `sudo systemctl status netdata`.  
- Web interface accessed at `http://192.168.56.10:19999`.  
Metrics collected include CPU usage, memory, network traffic, and system health.  
This enables proactive monitoring of server resources and early detection of issues.

## 10. Validation Testing
System validation was performed to ensure all requirements were met:

- **Hostnames** verified with `hostnamectl` and resolved via `ping <hostname>`  
- **Static IPs** confirmed persistent with `ip a` after reboot  
- **Connectivity** tested between VMs with `ping <ip>` (no packet loss)  
- **Restricted access** confirmed: only `lb-server` accessible via browser  
- **System updates** confirmed with `sudo apt update && sudo apt list --upgradable`  
- **DevOps user** verified in `/etc/passwd`, SSH key login tested, sudo group confirmed with `groups devops`  
- **Root login** disabled, password login prevented (SSH config checked)  
- **Firewall** validated with `sudo ufw status verbose` (only necessary ports open)  
- **Umask** set to `027`, tested with file creation  
- **Automatic updates** enabled via `/etc/apt/apt.conf.d/20auto-upgrades`  
- **SSH protocol** set to version 2 only in `/etc/ssh/sshd_config`  

Extra validation:  
- Fail2Ban bans confirmed via `sudo fail2ban-client status sshd`  
- WireGuard handshake checked with `sudo wg show`  
- Netdata dashboard verified at `http://192.168.56.10:19999`

## Challenges & Lessons Learned
- **Firewall conflicts**: Initially, WireGuard did not handshake due to missing UFW rules. Solved by explicitly allowing UDP 51820 and VPN subnet.  
- **Key mismatches**: Wrong public keys were used during WireGuard setup, causing failed connections. Solution was to regenerate and carefully match pairs.  
- **Monitoring installation**: Netdata installer returned a redirect error; solved by adjusting curl command.  
Lesson learned: Always validate configurations step by step and troubleshoot incrementally.

## Final Architecture Summary
The final system architecture includes:  
- **1 Load Balancer** (`lb-server`, Nginx, Netdata monitoring, VPN endpoint)  
- **2 Web Servers** (`web1-server`, `web2-server`, serving content behind load balancer)  
- **1 App Server** (`app-server`, backend logic)  

Security measures implemented:  
- Dedicated `devops` user with SSH key authentication only  
- Disabled root login and password login  
- Strict firewall rules (UFW)  
- Fail2Ban for intrusion prevention  
- WireGuard VPN for encrypted server communication  
- Secure umask and unattended upgrades for baseline hardening  

**Network Diagram (conceptual):**  
`[ Internet ] → [ LB-Server ] → [ Web1 / Web2 ] → [ App Server ]`  
VPN overlay (`10.0.0.0/24`) secures traffic between nodes.

### Recommendations for Future Improvements
- Add centralized logging (e.g., ELK or Loki + Grafana)  
- Expand monitoring with Prometheus + Grafana dashboards  
- Implement configuration management (e.g., Ansible) for automation  
- Consider TLS termination and Let’s Encrypt integration on load balancer  
- Add redundancy (HAProxy cluster, multiple app servers) for resilience