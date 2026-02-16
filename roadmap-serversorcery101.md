# 🗺️ Project Roadmap: Server Sorcery 101

## 1. Preparation✅
- Install virtualization software (VirtualBox, VMware, or Vagrant with VirtualBox for automation) ✅
- Prepare Ubuntu Server ISO (latest LTS, e.g., Ubuntu 22.04 LTS) ✅
- Decide resource allocations per VM ✅
  - Load Balancer: 2 CPU, 4GB RAM
  - Web Servers (2x): 2 CPU, 4GB RAM each (cloned from lb-server)
  - App Server: 2 CPU, 4GB RAM
- Prepare network plan (all VMs in same private subnet) ✅
  - Load Balancer: `192.168.56.10`
  - Web1: `192.168.56.11`
  - Web2: `192.168.56.12`
  - App: `192.168.56.13`

## 2. VM Creation and Hostnames
- Create 4 VMs in VirtualBox ✅
- Assign hostnames ✅
  - `lb-server`
  - `web1-server`
  - `web2-server`
  - `app-server`
- Verify hostnames with `hostnamectl`

## 3. Networking
- Assign static IPs in `/etc/netplan/*.yaml` ✅
- Ensure all machines are on same subnet ✅
- Test connectivity with `ping <IP>` and `ping <hostname>` ✅
- Restrict external access ✅
  - Only `lb-server` has NAT adapter (Internet + external access)
  - Others only use internal/private network

## 4. Linux Administration & Security Setup✅
- **User Management**✅
  - Create `devops` user  
  - Add to `sudo` group
- **SSH Hardening**✅
  - Generate SSH key on host and copy to VMs  
  - Edit `/etc/ssh/sshd_config`:  
    - `PermitRootLogin no`  
    - `PasswordAuthentication no`  
    - `Protocol 2`  
    - `AllowUsers devops`  
  - Restart SSH service
- **Network Interfaces**✅
  - Disable unused with `ip link set <iface> down`
- **Firewall (UFW)**✅
  - Deny incoming, allow outgoing  
  - Allow SSH everywhere, HTTP/HTTPS only on lb-server  
- **Security Settings**✅
  - Set secure umask (`027`)  
  - Enable unattended upgrades

## 5. Load Balancer Setup✅
- Install Nginx or HAProxy on lb-server
- Configure to balance requests between web1-server and web2-server
- Test: only lb-server should be reachable in browser

## 6. Web Servers Setup✅
- Install Nginx or Apache
- Serve simple `index.html` (“Hello from Web1”, “Hello from Web2”)
- Verify load balancer rotates requests

## 7. App Server Setup✅
- Deploy simple app (Python Flask, Node.js, or static)
- Ensure web servers proxy requests to app server if needed

## 8. Documentation✅
- Keep a detailed log of:  
  - Commands used  
  - Configs modified  
  - IP scheme  
  - Challenges and fixes  
- Draw architecture diagram: `LB → Web Servers → App`

## 9. Extra Challenges (Optional)✅
- Install Fail2Ban for intrusion prevention
- Configure VPN (WireGuard) for secure connections
- Install monitoring (Netdata / Prometheus + Grafana)

## 10. Validation Testing✅
- ✅ Hostname check  
- ✅ Ping between VMs  
- ✅ SSH only with devops key  
- ✅ Root login disabled  
- ✅ UFW running with correct rules  
- ✅ Automatic updates configured  
- ✅ Load balancer accessible via browser, web servers not directly  
- ✅ Devops user in sudo group  
- ✅ Check umask  

