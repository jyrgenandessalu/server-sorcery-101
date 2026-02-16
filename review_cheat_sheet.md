# 🧾 Review Cheat Sheet – Server Sorcery 101

This cheat sheet covers **all mandatory** and **extra requirements** for your project review.  
Each section includes:
- Command to run  
- Expected output  
- What to explain to the reviewer  

---

## ✅ Mandatory Requirements

### 1. Documentation is clear
- Already done in your `documentation.md`
- Be ready to explain scope, objectives, procedures, and challenges/solutions.

---

### 2. VM Creation & Resources
- **Check VM names**  
  ```bash
  hostname
  ```  
  **Expected output**: `lb-server`, `web1-server`, `web2-server`, `app-server` (depending on VM).

- **Check resources in VirtualBox UI**  
  Explain: Load balancer and web servers have 2 CPU / 4GB RAM, app server also 2 CPU / 4GB RAM.  
  Reasoning: balanced resources for a small-scale test environment.

---

### 3. Hostnames are resolvable
- From one VM (e.g. lb-server):  
  ```bash
  ping -c 3 web1-server
  ```  
  **Expected output**: Replies with `64 bytes from 192.168.56.11`.  
  Explain: hostnames resolve because they were set with `hostnamectl`.

---

### 4. Static IPs assigned & persistent
- Run:  
  ```bash
  ip a
  ```  
  **Expected output**:  
  - lb-server → `192.168.56.10/24`  
  - web1-server → `192.168.56.11/24`  
  - web2-server → `192.168.56.12/24`  
  - app-server → `192.168.56.13/24`  
  Explain: configured via Netplan YAML files, persistent after reboot.

---

### 5. VM communication works
- Test connectivity:  
  ```bash
  ping -c 3 192.168.56.12
  ```  
  **Expected output**: No packet loss.  
  Explain: all VMs are on the same subnet.

---

### 6. Only Load Balancer is accessible via browser
- In browser: `http://192.168.56.10`  
  **Expected output**: Alternates between “Hello from Web1” and “Hello from Web2”.  
- Try `http://192.168.56.11` → should fail (blocked by firewall).  
  Explain: Only lb-server has NAT + HTTP/HTTPS allowed in UFW.

---

### 7. Security patches up-to-date
- Run:  
  ```bash
  sudo apt update && sudo apt list --upgradable
  ```  
  **Expected output**: Either `All packages are up to date.` or a short list (if any).

---

### 8. DevOps user created
- Run:  
  ```bash
  grep devops /etc/passwd
  ```  
  **Expected output**: Line with `/home/devops` and `/bin/bash`.

---

### 9. Password login disabled (SSH key only)
- Test from host:  
  ```bash
  ssh devops@192.168.56.10
  ```  
  **Expected**: Logs in without password prompt.  
- In `/etc/ssh/sshd_config`, `PasswordAuthentication no`.  
  Explain: This enforces key-based login only.

---

### 10. DevOps in sudo group
- Run:  
  ```bash
  groups devops
  ```  
  **Expected output**: `devops : devops sudo`.

---

### 11. Sudo requires password
- Run:  
  ```bash
  sudo ls /root
  ```  
  **Expected output**: Prompts for `devops` password.  
  Explain: safer than nopasswd sudo.

---

### 12. Root login disabled
- Run:  
  ```bash
  ssh root@192.168.56.10
  ```  
  **Expected output**: `Permission denied`.

---

### 13. Only devops can login
- Try another user:  
  ```bash
  ssh linus@192.168.56.10
  ```  
  **Expected output**: `Permission denied`.

---

### 14. Unused interfaces disabled
- Run:  
  ```bash
  ip link show
  ```  
  **Expected output**: Only `lo` and `enp0s8` (private), `enp0s3` only on lb-server.  
  Explain: Reduces attack surface.

---

### 15. UFW firewall active & correct
- Run:  
  ```bash
  sudo ufw status verbose
  ```  
  **Expected output**:  
  - Default deny incoming, allow outgoing  
  - SSH allowed  
  - HTTP/HTTPS allowed only on lb-server  
  Explain: Policies restrict access to essentials.

---

### 16. Secure umask set
- Run:  
  ```bash
  umask
  ```  
  **Expected output**: `027`.  
  Explain: New files aren’t world-readable/writable.

---

### 17. Automatic updates enabled
- Run:  
  ```bash
  cat /etc/apt/apt.conf.d/20auto-upgrades
  ```  
  **Expected output**:  
  ```
  APT::Periodic::Update-Package-Lists "1";
  APT::Periodic::Unattended-Upgrade "1";
  ```

---

### 18. SSH protocol v2 only
- Run:  
  ```bash
  grep Protocol /etc/ssh/sshd_config
  ```  
  **Expected output**: `Protocol 2`.  

---

## 🚀 Extra Requirements

### A) Intrusion Prevention – Fail2Ban
- Run:  
  ```bash
  sudo systemctl status fail2ban
  ```  
  **Expected output**: `active (running)`.  
  Explain: bans IPs after repeated failed login attempts.

- Show logs:  
  ```bash
  sudo cat /var/log/fail2ban.log | tail
  ```  
  **Expected**: shows jail activity and bans.

---

### B) VPN – WireGuard
- Run:  
  ```bash
  sudo wg show
  ```  
  **Expected output**: Lists interface (wg0), public keys, latest handshake, data transferred.  
  Explain: This proves encrypted tunnel between peers is active.

---

### C) Monitoring – Netdata
- Run in browser:  
  `http://192.168.56.10:19999`  
  **Expected output**: Real-time monitoring dashboard.  
  Explain: Provides CPU, RAM, disk, and network stats for troubleshooting.

---

### D) Additional enhancements
- Examples: stronger firewall rules, custom MOTD banner, systemd app service.  
  Be ready to explain them if asked.
