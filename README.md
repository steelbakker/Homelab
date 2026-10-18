# Homelab Infrastructure — Architecture and Security Overhaul (October 2026)

## Project Overview

This project began as a self-hosted game server environment running on Proxmox VE, using Pterodactyl to manage Minecraft and Project Zomboid servers. Over time, it evolved into a more comprehensive homelab project involving virtualization, network security, reverse proxies, private remote access, automated backups, and identity-based access control.

The latest iteration focuses on reducing the public attack surface, improving reliability, and introducing more deliberate separation between administrative services and game-server access.

The primary active workload is now a Minecraft Java server running NeoForge, managed through Pterodactyl. Minecraft Bedrock has been deprecated, while Project Zomboid remains a potential future project.

> **Network addressing and hostname note:** All private IP addresses and infrastructure hostnames in this documentation are fictional placeholders. IP addresses use the example subnet `192.168.50.0/24`, and public-facing domain examples use the reserved `example.com` domain. These values illustrate the infrastructure's relationships and do not represent the actual addresses or hostnames used in the deployment.

---

## 1. Original Infrastructure

The original environment consisted of a Proxmox host running separate Ubuntu virtual machines for the Pterodactyl Panel, Wings game-server node, and Nginx reverse proxy.

**Original architecture:**

- **Proxmox VE:** Hosted and managed the virtual machines.
- **Pterodactyl Panel:** Provided a web interface for administering game servers.
- **Pterodactyl Wings:** Managed Docker containers running Minecraft and other game servers.
- **Nginx reverse proxy:** Terminated HTTPS connections and routed requests to the Panel and Wings API.
- **Cloudflare:** Managed public DNS records for the `example.com` domain and proxied selected HTTPS services.
- **Comcast gateway:** Used port forwarding to expose services through the home's public IP address.
- **Let's Encrypt:** Issued HTTPS certificates using Certbot's Nginx integration.

Administrative services were accessible using public hostnames such as `panel.example.com` and `node1.example.com`. Incoming connections passed through Cloudflare and the Comcast gateway before reaching the reverse proxy.

Game servers were also made accessible through publicly forwarded ports.

Although functional, this design introduced several limitations:

- Administrative interfaces depended on publicly reachable network paths.
- Multiple services required inbound port forwarding.
- Public DNS records referenced the home internet connection.
- Certificate renewal depended on HTTP-based validation.
- Server accessibility and administrative permissions were not separated as strictly as desired.
- Backup automation and off-site retention required further development.

The infrastructure was functional, but it relied primarily on perimeter firewall configuration and publicly reachable endpoints.

---

## 2. Migration to Twingate Private Access

The largest architectural change was replacing public inbound access with Twingate.

A dedicated Ubuntu 24.04 unprivileged LXC container was deployed on Proxmox to run the Twingate Connector.

**Connector configuration:**

- Proxmox container ID: `4005`
- Hostname: `twingate-connector`
- Internal IP: `192.168.50.11`
- CPU: 1 core
- Memory: 1 GB
- Storage: 8 GB
- Network: `vmbr0`

Unlike the original public-access design, the Connector establishes outbound communication with Twingate. Authorized users can reach explicitly defined internal services without opening inbound ports on the Comcast gateway.

All public port-forwarding rules associated with the homelab services were removed.

### Resource-based access control

Twingate Resources were created to restrict access to individual services and ports.

| Resource | Internal destination | Allowed access |
|---|---|---|
| Pterodactyl Panel | `192.168.50.12:443` | Homelab Administrators |
| Pterodactyl Wings | `192.168.50.12:443` | Homelab Administrators |
| Minecraft Java | `192.168.50.14:27050` | Game Server Players |

Twingate hostname aliases preserve the existing service addresses:

- `panel.example.com`
- `node1.example.com`
- `minecraft.example.com`

This allows users to continue connecting through familiar hostnames while traffic is routed privately through Twingate rather than through public DNS and port forwarding.

The administrative Resources are restricted to TCP 443, while Minecraft Java is restricted to TCP 27050. Unnecessary UDP and ICMP access is disabled.

A previously configured direct Panel Resource was also removed to eliminate redundant access paths.

**Result:** Administrators and game-server players receive different levels of access according to their responsibilities.

---

## 3. Reverse Proxy and Internal Network Routing

The Nginx reverse proxy remains part of the architecture, but its role has changed.

Previously, it accepted traffic originating from publicly forwarded ports. It now primarily serves as an internal HTTPS entry point for authorized Twingate users and communication between Pterodactyl components.

**Current reverse proxy:**

- Hostname: `reverse-proxy`
- Internal IP: `192.168.50.12`
- Operating system: Ubuntu Server
- HTTPS: TCP 443

Nginx routes requests according to the requested hostname:

| Hostname | Internal upstream |
|---|---|
| `panel.example.com` | `192.168.50.13:80` |
| `node1.example.com` | `192.168.50.14:443` |

Internal `/etc/hosts` entries were added where necessary so that the Panel and Wings node could resolve these hostnames to the reverse proxy without relying on public DNS.

This required troubleshooting communication in both directions between the Panel and Wings, including API requests, server status updates, and backup downloads.

### HTTPS-only configuration

The reverse proxy's obsolete port 80 listeners, HTTP-to-HTTPS redirects, and HTTP-01 certificate validation configuration were removed.

Nginx now listens on TCP 443 only.

The reverse proxy's firewall permits HTTPS connections from the Twingate Connector, the Panel VM, and the Wings VM, while SSH access is restricted to the administrative workstation.

The Nginx configuration was backed up and tested before applying these changes.

---

## 4. Cloudflare DNS and Automated Certificate Renewal

An important part of the migration was separating **public DNS hosting from private service accessibility**.

Previously, Cloudflare held public DNS records for the Pterodactyl services and participated in routing publicly accessible traffic.

Those public records were no longer required after migrating the services to Twingate.

The Cloudflare DNS configuration was reduced to records needed for other services, including:

- The public OAuth documentation site hosted through GitHub Pages.
- An existing Google domain-verification TXT record.

Pterodactyl and Minecraft no longer require permanent public DNS A or CNAME records.

### Migration from HTTP-01 to DNS-01

Let's Encrypt certificate renewal was migrated from Certbot's Nginx-based HTTP validation to Cloudflare DNS-01 validation.

The reverse proxy uses the `python3-certbot-dns-cloudflare` plugin and a restricted Cloudflare API token to create temporary DNS TXT records during certificate validation.

This allows publicly trusted HTTPS certificates to be issued and renewed even though the Panel and Wings services themselves are not publicly reachable.

The configuration includes:

- Cloudflare API credentials stored in a root-restricted configuration file.
- Automatic Certbot renewal through its systemd timer.
- A deployment hook that validates and reloads Nginx after certificate renewal.
- Successful renewal dry-run testing for the Panel and Wings certificates.

**Result:** HTTPS certificate renewal is independent of public inbound port forwarding.

---

## 5. Google OAuth Integration and Automated Google Drive Backups

Another major addition to the project was an automated off-site backup workflow using Google Drive.

This involved configuring Google OAuth access for `rclone`, establishing a publicly accessible OAuth information site, and developing a Python script to automate the backup lifecycle.

### Google OAuth and GitHub Pages

The Google Drive integration required setting up OAuth consent and providing application information, including a homepage and privacy policy.

Initially, the OAuth-related website was hosted through the homelab's Nginx infrastructure.

As part of the migration away from public inbound services, that content was moved to GitHub Pages.

The public website is now available at:

- `https://oauth.example.com/`
- `https://oauth.example.com/privacy/`

Cloudflare provides the public CNAME record that directs this hostname to GitHub Pages.

This allows the OAuth information pages to remain publicly accessible without requiring any inbound connectivity to the homelab.

The site serves as the OAuth application's public informational presence; it is not a game-server endpoint or a self-hosted OAuth callback service.

### Automated Minecraft backup workflow

A custom Python backup script runs on the Pterodactyl Wings VM and coordinates the backup process through the Pterodactyl Client API.

The workflow:

1. Sends a Discord notification that the backup job has started.
2. Checks the Minecraft server's current state.
3. Stops the server gracefully if it is running.
4. Creates a Pterodactyl backup.
5. Waits for backup completion.
6. Applies local automatic-backup retention rules.
7. Restarts the Minecraft server and verifies recovery.
8. Downloads the backup archive using a temporary URL generated by Pterodactyl.
9. Uploads the archive to Google Drive using `rclone`.
10. Verifies the uploaded archive's size.
11. Applies tiered cloud-backup retention rules.
12. Sends a Discord completion or failure notification.

The script uses a `flock` lock to prevent overlapping backup jobs.

### Backup retention

The configured cloud retention policy provides:

- 7 daily dates
- 4 ISO weekly periods
- 6 monthly periods

Retention tiers may overlap, preventing unnecessary duplicate retention.

Manual or locked Pterodactyl backups are handled separately from automatic backup cleanup.

### Backup troubleshooting and verification

Removing public DNS records initially caused an automated backup failure because Wings could no longer resolve the hostname in its temporary archive download URL.

An internal `/etc/hosts` mapping for `node1.example.com` corrected the issue.

A complete end-to-end test subsequently confirmed:

- Successful Minecraft shutdown and restart.
- Successful local backup creation.
- Successful archive download.
- Successful Google Drive upload and file-size verification.
- Successful retention processing.
- Successful Discord notifications.

The verified backup was approximately 183 MiB.

**Result:** The active Minecraft server now has automated local and off-site backups, including server recovery checks and notification reporting. A full restore from Google Drive remains a future validation task.

---

## 6. Firewall Hardening and Docker Network Isolation

The migration also included a security audit of the virtual machines, container networking, and Proxmox host.

### Pterodactyl Wings VM

The Wings VM (`192.168.50.14`) uses UFW with default-deny incoming traffic.

Administrative SSH is restricted to the Windows management workstation, and Wings API access is permitted from authorized internal sources.

Minecraft Java runs inside a Docker container managed by Wings.

During testing, an important limitation was discovered: **Docker's automatically generated forwarding rules can allow published container traffic before it reaches UFW's normal forwarding restrictions.**

To address this, additional firewall rules were installed in Docker's `DOCKER-USER` chain.

These rules:

- Permit established and related traffic.
- Restrict new TCP connections to Minecraft Java on port 27050 to the Twingate Connector (`192.168.50.11`).
- Block unnecessary UDP traffic to the published Java server port.

The rules use connection tracking to match the original destination address and port, avoiding reliance on Docker's dynamically assigned container IP addresses.

A systemd service and shell script were created to reapply the restrictions when Docker starts.

Testing demonstrated that:

- Minecraft remained accessible through Twingate.
- Direct TCP access from the administrative PC to the game server's LAN IP was blocked.
- Firewall counters recorded the blocked traffic.
- The restrictions were restored successfully following a Docker service restart.

### Pterodactyl Panel VM

The Panel VM (`192.168.50.13`) was reviewed for unnecessary listening services and firewall exposure.

Its incoming access is restricted to:

- SSH from the administrative workstation.
- HTTP from the reverse proxy.

MariaDB and Redis listen on loopback addresses rather than on externally accessible interfaces.

### Twingate Connector

The Twingate Connector was deployed as an unprivileged Proxmox LXC container.

Unnecessary inbound SSH service was disabled, and a dedicated Proxmox container firewall was configured.

Its network interface has Proxmox firewall filtering enabled, with a default-deny incoming policy and outbound communication permitted.

IPv4 and IPv6 firewall rules were generated and validated. The Connector was restarted successfully, and access to the Pterodactyl Panel and Minecraft Java was tested afterward.

---

## 7. Proxmox Host Security Hardening

The Proxmox host itself was also reviewed and hardened.

**Host configuration:**

- Proxmox VE 9.1
- Management IP: `192.168.50.10`
- Linux bridge: `vmbr0`
- Administrative workstation: `192.168.50.100`

### Removal of unnecessary services

Two services were identified as unnecessary for the current host configuration:

- Nginx serving a default HTTP website on port 80.
- `rpcbind` listening on TCP/UDP 111.

The host uses local directory and LVM-thin storage, with no NFS mounts detected.

Both services were disabled, leaving the primary management services running:

- SSH — TCP 22
- Proxmox HTTPS web interface — TCP 8006
- SPICE proxy — TCP 3128

### Host firewall restrictions

The Proxmox host firewall was enabled with explicit access rules for the administrative workstation.

During verification, it became apparent that Proxmox automatically included the entire `192.168.50.0/24` subnet in its trusted management IP set.

This was more permissive than intended.

The automatically detected management network was overridden through the `local_network` alias, and the management IP set was narrowed to:

- `192.168.50.100` — Administrative workstation
- `192.168.50.10` — Proxmox host

The resulting firewall configuration compiled successfully.

Testing confirmed that the administrative workstation retained SSH and HTTPS access, while an iPhone connected to the same home Wi-Fi could no longer establish a connection to the Proxmox web interface.

**Result:** Proxmox management is no longer automatically accessible to every device on the local subnet.

---

## 8. Final Architecture

The current design separates administrative access from game-server access and eliminates the need for direct public exposure of these services.

### Administrative traffic

```text
Authorized Administrator
         |
         v
    Twingate Client
         |
         v
   Twingate Connector
       192.168.50.11
         |
         v
   Nginx Reverse Proxy
       192.168.50.12
         |
         +---- Pterodactyl Panel
         |       192.168.50.13:80
         |
         +---- Pterodactyl Wings API
                 192.168.50.14:443
```

### Minecraft player traffic

```text
Authorized Minecraft Player
         |
         v
    Twingate Client
         |
         v
   Twingate Connector
       192.168.50.11
         |
         v
   Pterodactyl Wings VM
       192.168.50.14
         |
         v
  Docker: Minecraft Java
        TCP 27050
```

### Backup traffic

```text
Minecraft Java Server
         |
         v
  Pterodactyl Backup API
         |
         v
    Backup Archive
         |
         v
   Python Backup Script
         |
         +---- Local Pterodactyl Retention
         |
         +---- rclone / Google OAuth
         |             |
         |             v
         |         Google Drive
         |
         +---- Discord Notifications
```

Google Drive uploads, certificate renewal, and other required external integrations operate over outbound connections rather than public inbound ports.

---

## 9. Outcomes and Lessons Learned

This phase of the project provided practical experience with:

- Proxmox VE virtualization and LXC containers.
- Ubuntu Server administration.
- Docker container networking and firewall rule evaluation.
- UFW, iptables, connection tracking, and Proxmox firewall policies.
- Twingate private access and resource-based authorization.
- Nginx reverse proxy configuration and TLS termination.
- Cloudflare DNS management and DNS-01 certificate validation.
- Google OAuth configuration and GitHub Pages hosting.
- Python automation, Pterodactyl APIs, and `rclone`.
- Automated backup retention, recovery checks, and Discord notifications.
- Troubleshooting internal DNS, network routing, and service connectivity.

Several issues were resolved through direct inspection of logs, packet-filtering rules, service listeners, and network connectivity rather than relying solely on application-level configuration.

Notable examples included identifying Docker's interaction with UFW, correcting DNS resolution for Pterodactyl backup downloads, and narrowing Proxmox's automatically trusted management network.

The largest improvement is that the homelab no longer depends on public inbound port forwarding for its active services.

Administrative access is limited to authorized Twingate users and designated management devices, while Minecraft players are granted access only to the necessary game-server port.

## 10. Future Improvements

The current deployment is operational, but additional projects remain:

- Introduce VLAN segmentation to separate management devices, server infrastructure, and everyday household devices.
- Replace or supplement the Comcast gateway with dedicated, VLAN-capable networking hardware.
- Perform a full Google Drive backup restoration test.
- Establish routine operating-system and application patching procedures.
- Expand monitoring and alerting for infrastructure services.
- Perform independent external connectivity and IPv6 exposure testing.
- Revisit Project Zomboid hosting as a separate future project.
- Continue documenting architectural decisions and configuration changes.

**Project status — October 2026:** Core Twingate migration, active Minecraft Java hosting, automated off-site backups, certificate renewal, and principal host firewall hardening completed and tested.
