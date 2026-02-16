# 🗺️ Server Sorcery 101 – Architecture Diagram

Here’s the full system layout:

```mermaid
flowchart LR

  subgraph Internet
    Client[Client Browser / VPN]
  end

  subgraph DMZ["Public Access Zone"]
    LB[LB-Server<br/>192.168.56.10<br/>Nginx Load Balancer<br/>UFW + Fail2Ban<br/>WireGuard VPN<br/>Netdata Monitoring]
  end

  subgraph PrivateNet["Internal Network - Subnet"]
    Web1[Web1-Server<br/>192.168.56.11<br/>Nginx Static Pages<br/>Proxy to App]
    Web2[Web2-Server<br/>192.168.56.12<br/>Nginx Static Pages<br/>Proxy to App]
    App[App-Server<br/>192.168.56.13<br/>Python HTTP App<br/>Port 5000]
  end

  Client -->|HTTP/HTTPS| LB
  Client -->|VPN Tunnel| LB
  LB --> Web1
  LB --> Web2
  Web1 -->|/app| App
  Web2 -->|/app| App
