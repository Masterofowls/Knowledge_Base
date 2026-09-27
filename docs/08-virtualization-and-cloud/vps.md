# How a VPS Works

> **Level:** Advanced · **Related:** [Virtualization/Hypervisors](virtualization.md) · [Virtual Machines](virtual-machines.md) · [Web Server](../05-networking-and-web/web-server.md) · [OS](../03-os-and-software/operating-system.md) · [VPN](../05-networking-and-web/vpn.md) · [Proxy](../05-networking-and-web/proxy.md)

## 1. What a VPS is

A **VPS (Virtual Private Server)** is a [virtual machine](virtual-machines.md) rented from a hosting provider: a slice of a powerful physical server, running its own OS, with dedicated resources and full root/administrator access, reachable over the internet with a public IP. You get most of the control of a dedicated server at a fraction of the cost, because one physical host is shared among many isolated VPS tenants via a [hypervisor](virtualization.md).

```
       Physical server in a data center (many CPU cores, lots of RAM, fast NVMe, 10–100 GbE)
                                   │  Hypervisor (KVM, Xen, Hyper-V, VMware)
        ┌──────────────┬──────────────┬──────────────┐
     VPS (you)      VPS (tenant B)  VPS (tenant C)  ...
   4 vCPU, 8 GB     2 vCPU, 4 GB    8 vCPU, 16 GB
   Ubuntu, root     Windows         Debian
   1.2.3.4          1.2.3.5         1.2.3.6
```

## 2. Where a VPS sits among hosting options

| Option | Isolation | Control | Cost | Scaling | Analogy |
|---|---|---|---|---|---|
| **Shared hosting** | Just a user account on a shared server | Minimal (cPanel) | $ | Limited | A room in a shared house |
| **VPS** | Own VM, own OS, root | Full OS control | $$ | Resize/add nodes | Your own apartment |
| **Dedicated server** | Whole physical machine | Total (bare metal) | $$$$ | Buy more boxes | A whole house |
| **Cloud instance** (AWS EC2, Azure VM, GCP) | Own VM + rich platform APIs | Full + automation | $$–$$$$ | Elastic, API-driven | Apartment with concierge services |
| **Containers / PaaS / serverless** | Process/function isolation | App-level only | $–$$ | Auto | A serviced desk |

A "VPS" and a "cloud VM" are technically the same thing (a virtualized server); the difference is ecosystem — cloud instances come with load balancers, managed databases, autoscaling, IAM and pay-per-second billing, while a classic VPS (DigitalOcean Droplet, Linode, Hetzner, Vultr, OVH) is a simpler fixed-price box.

## 3. How resources are allocated

- **vCPUs**: virtual cores time-sliced onto physical cores by the hypervisor scheduler. Cheap plans **oversubscribe** (more vCPUs sold than exist), so a busy neighbor can cause **CPU steal** (`%st` in `top`/`vmstat` = time your vCPU was ready but waiting for a real core). "Dedicated CPU"/"compute-optimized" plans pin cores to you.
- **RAM**: usually not oversubscribed on quality providers; guaranteed via [EPT/NPT](virtualization.md#32-memory-virtualization). Cheap hosts may use ballooning.
- **Storage**: a [virtual disk](virtual-machines.md#31-virtual-disks) on NVMe SSDs, often network-attached **block storage** (replicated, resizable, snapshottable) rather than local disk. Measured in IOPS and throughput, sometimes capped.
- **Network**: a virtual NIC with a public IPv4 (and IPv6), bandwidth and monthly transfer quotas; private networking (VLAN/VPC) between your own servers.
- **Noisy neighbors**: contention for shared cache, memory bandwidth, disk and network — the main downside of shared virtualization (see [Virtualization §9](virtualization.md#9-overhead-and-pitfalls)).

## 4. Provisioning: from click to running server

```mermaid
sequenceDiagram
  participant U as You
  participant API as Provider control panel / API
  participant HV as Hypervisor on a host
  participant VM as Your VPS
  U->>API: Create VPS (region, size, OS image, SSH key)
  API->>HV: Pick a host with capacity; clone OS template
  HV->>VM: Boot VM; cloud-init injects hostname, SSH key, network
  VM-->>U: Public IP + you can SSH/RDP in ~30–60s
```

Providers expose this as an [API](../05-networking-and-web/api.md), so servers can be created programmatically or with **Infrastructure as Code** (Terraform/OpenTofu, Ansible, Pulumi). A **golden image** + **cloud-init** makes each new VPS reproducible — see [Virtual Machines §3.3](virtual-machines.md#33-vm-images-templates-and-reproducibility).

## 5. Connecting and operating

**Linux VPS** — via SSH:

```bash
ssh-keygen -t ed25519                                  # once, on your machine
ssh root@1.2.3.4                                        # key-based login (disable passwords!)

# First-hour hardening
adduser deploy && usermod -aG sudo deploy
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/;s/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh
ufw allow OpenSSH && ufw allow 80,443/tcp && ufw enable # firewall
apt update && apt upgrade -y
```

**Windows VPS** — via RDP (Remote Desktop, port 3389) or PowerShell Remoting/SSH; secure with strong passwords, NLA, firewall rules, and ideally a VPN in front of RDP.

Diagnostics you'll actually use: `htop`/`top` (watch `%st` for steal), `free -h` (RAM), `df -h` (disk), `iostat`/`iotop` (disk I/O), `ss -tlnp` (listening ports), `journalctl -u <service>`; Windows: Task Manager, Resource Monitor, `Get-Counter`, Event Viewer. See [OS §12](../03-os-and-software/operating-system.md#12-hands-on-map-where-do-i-look).

## 6. What people run on a VPS

- **Web apps & APIs** — [web server](../05-networking-and-web/web-server.md) (nginx/Caddy) + app server + database.
- **Databases, caches, queues** — PostgreSQL, Redis, RabbitMQ.
- **Personal services** — Nextcloud, Git server, mail, media (Jellyfin), home-lab dashboards.
- **[VPN](../05-networking-and-web/vpn.md) / [proxy](../05-networking-and-web/proxy.md) endpoints** — WireGuard, a private exit node.
- **Game servers, bots, CI runners, containers** (Docker/Podman) and Kubernetes nodes.

A minimal production web stack on a fresh VPS:

```bash
# Docker-based deploy of an app behind a reverse proxy with automatic HTTPS
curl -fsSL https://get.docker.com | sh
cat > compose.yml <<'EOF'
services:
  caddy:                       # reverse proxy + auto TLS (Let's Encrypt)
    image: caddy:2
    ports: ["80:80", "443:443"]
    command: caddy reverse-proxy --from example.com --to app:3000
  app:
    image: myorg/myapp:latest
    restart: always
EOF
docker compose up -d
```

Point your domain's DNS **A record** at the VPS IP (see [Web Protocols §4](../05-networking-and-web/web-protocols.md#4-dns-names-to-addresses)) and it's live over HTTPS.

## 7. Scaling and reliability

- **Vertical scaling**: resize the VPS to more vCPU/RAM (usually a reboot; some resize live).
- **Horizontal scaling**: run several VPSs behind a **load balancer**; keep app servers stateless, put sessions/data in a shared database or cache (see [Web Server §8](../05-networking-and-web/web-server.md#8-performance--scaling)).
- **Availability**: a single VPS is a single point of failure — spread across regions/providers, use health checks and failover.
- **Backups**: provider **snapshots** (point-in-time full-disk) + application-level backups stored off-box (object storage). Test restores. Snapshots are not backups if they live on the same infrastructure.
- **Monitoring/alerting**: uptime checks, metrics (Prometheus/Grafana, Netdata), log aggregation.

## 8. Security responsibilities

A VPS gives you root — and the duty to secure it. The provider secures the hypervisor and physical layer; **everything inside the guest is yours** (the cloud "shared responsibility model").

- SSH keys only, no password/root login; change nothing-defaults.
- **Firewall** (ufw/nftables, security groups) — expose only needed ports.
- Patch promptly (unattended-upgrades); minimize installed services.
- **fail2ban**/crowdsec against brute force; put admin panels behind a [VPN](../05-networking-and-web/vpn.md).
- Least-privilege app users; secrets in a manager, not in code.
- TLS everywhere ([Encryption](../06-security-and-data/encryption.md)); watch for [malware](../06-security-and-data/malware.md)/cryptominers (unexpected CPU, outbound connections).
- Isolation caveat: you share hardware, so cross-VM side channels exist — clouds mitigate, but sensitive workloads may want dedicated/bare-metal.

## 9. Cost model

- Billed hourly or monthly by size (vCPU/RAM/disk) plus bandwidth overages, extra storage, snapshots, and add-ons (load balancers, backups, floating IPs).
- Classic VPS: flat monthly, generous transfer. Cloud: per-second, many à-la-carte services, easy to overspend — set budgets/alerts.
- Right-size using real metrics; downsize idle boxes; reserved/committed pricing for steady workloads.

## Further reading
- DigitalOcean, Linode and Hetzner tutorials/docs (practical VPS setup and hardening)
- AWS/Azure/GCP "shared responsibility model" documentation
- *The Cloud Resume Challenge*; Terraform / cloud-init documentation; `ssh_config`/`sshd_config` man pages
