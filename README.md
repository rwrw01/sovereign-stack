# Sovereign Stack

Enterprise-grade, self-hosted sovereign infrastructure stack with zero-trust architecture, designed for business use on a single Hetzner VPS.

## What is this?

A complete blueprint and implementation guide for deploying a fully sovereign digital workspace:

- **Identity & MFA** — [Keycloak](https://www.keycloak.org/) for SSO, OIDC, and mandatory TOTP
- **Collaboration** — [Nextcloud](https://nextcloud.com/) for files, calendar, contacts
- **Website/CMS** — [Ghost](https://ghost.org/) for content management
- **Email** — [Mailcow](https://mailcow.email/) for full-featured email with antispam/antivirus
- **VPN** — [Headscale](https://github.com/juanfont/headscale) (self-hosted Tailscale) for management plane isolation
- **Reverse Proxy** — [Traefik](https://traefik.io/) with Let's Encrypt, security headers, rate limiting

## Architecture Principles

| Principle | Implementation |
|-----------|---------------|
| **Zero Trust** | Every service in its own isolated Docker Compose stack |
| **Assume Breach** | Compromise of one service cannot reach others |
| **Defense in Depth** | Firewall + network segmentation + container hardening + Seccomp + AppArmor |
| **Data/Control Plane Separation** | Public services via internet, admin interfaces via Headscale VPN only |
| **Sovereignty** | Self-hosted VPN (Headscale), LUKS disk encryption, GPG-encrypted offsite backups |

## Stack Overview

```
Internet → UFW → Traefik (TLS) → Isolated Docker Stacks
                                    ├── Keycloak + PostgreSQL + oauth2-proxy
                                    ├── Nextcloud + PostgreSQL + Redis
                                    ├── Ghost + MariaDB
                                    ├── Mailcow (own ecosystem)
                                    └── Headscale (VPN control server)
```

Each stack has:
- Own `docker-compose.yml` and secrets directory
- Own internal `--internal` Docker networks (databases invisible from outside)
- Shared only via `proxy` network (HTTP routing through Traefik)
- Container hardening: `no-new-privileges`, `cap_drop: ALL`, Seccomp, AppArmor, memory limits

## Target Server

- **Hetzner cax31**: ARM64, 8 vCPU, 16GB RAM, 160GB disk
- **OS**: Ubuntu 22.04
- **Memory usage**: ~7.8GB of 16GB (50% headroom)

## Security Features

- **Network segmentation**: 7+ isolated Docker networks, databases on `--internal` networks
- **Docker socket protection**: Tecnativa socket-proxy (read-only, containers endpoint only)
- **Admin isolation**: Keycloak admin, Mailcow admin, Traefik dashboard only via Headscale VPN
- **Forward authentication**: Ghost admin behind oauth2-proxy + Keycloak MFA + Tailscale whitelist
- **RBAC**: Keycloak `admin_access` role required for admin panel access
- **Container hardening**: Seccomp, AppArmor, non-root users, read-only rootfs, PID limits
- **Image security**: Docker Content Trust, pinned versions, weekly Trivy scans
- **Runtime monitoring**: Falco for anomaly detection (shell spawn, crypto mining, etc.)
- **Mail security**: SPF, DKIM, DMARC with progressive enforcement
- **Backup encryption**: Asymmetric GPG (public key on server, private key offline only)
- **Disk encryption**: LUKS on data volume
- **Audit**: CIS Docker Benchmark, nmap validation, cross-stack isolation tests

## Documentation

See the [docs/](docs/) directory for the full deployment plan and phase-by-phase guides.

- [Full Deployment Plan](docs/deployment-plan.md) — Complete architecture, security matrix, and validation checklist

## Packages & Licenses

All packages used in this stack are open source. The following table lists each component, its license, and its role:

| Package | Version | License | Role |
|---------|---------|---------|------|
| [Traefik](https://github.com/traefik/traefik) | v2.11 | MIT | Reverse proxy, TLS termination, rate limiting |
| [Keycloak](https://github.com/keycloak/keycloak) | 26.0 | Apache-2.0 | Identity provider, SSO, MFA (TOTP) |
| [Nextcloud](https://github.com/nextcloud/server) | 30.x | AGPL-3.0 | Collaboration (files, calendar, contacts) |
| [Ghost](https://github.com/TryGhost/Ghost) | 5.x | MIT | Website / CMS |
| [Mailcow](https://github.com/mailcow/mailcow-dockerized) | latest | GPL-3.0 | Email server (Postfix, Dovecot, SOGo, Rspamd, ClamAV) |
| [Headscale](https://github.com/juanfont/headscale) | 0.26 | BSD-3-Clause | Self-hosted Tailscale control server (VPN) |
| [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) | v7.6.0 | MIT | Forward authentication proxy for Keycloak SSO |
| [PostgreSQL](https://www.postgresql.org/) | 16-alpine | PostgreSQL License | Database for Keycloak and Nextcloud |
| [MariaDB](https://mariadb.org/) | 11-jammy | GPL-2.0 | Database for Ghost |
| [Redis](https://github.com/redis/redis) | 7-alpine | BSD-3-Clause | Cache for Nextcloud |
| [Docker Socket Proxy](https://github.com/Tecnativa/docker-socket-proxy) | latest | Apache-2.0 | Read-only Docker API proxy for Traefik |
| [Falco](https://github.com/falcosecurity/falco) | latest | Apache-2.0 | Runtime security / anomaly detection |
| [Trivy](https://github.com/aquasecurity/trivy) | latest | Apache-2.0 | Container vulnerability scanning |
| [Node Exporter](https://github.com/prometheus/node_exporter) | latest | Apache-2.0 | System metrics collection |
| [rclone](https://github.com/rclone/rclone) | latest | MIT | Offsite backup sync to Hetzner Storage Box |

### Mailcow Sub-Components

Mailcow bundles the following open-source components in its stack:

| Component | License | Role |
|-----------|---------|------|
| [Postfix](http://www.postfix.org/) | IBM Public License / Eclipse Public License 2.0 | SMTP server |
| [Dovecot](https://dovecot.org/) | MIT / LGPL-2.1 | IMAP/POP3 server |
| [SOGo](https://www.sogo.nu/) | GPL-2.0 | Webmail / CalDAV / CardDAV |
| [Rspamd](https://rspamd.com/) | Apache-2.0 | Spam filtering |
| [ClamAV](https://www.clamav.net/) | GPL-2.0 | Antivirus scanning |
| [Unbound](https://nlnetlabs.nl/projects/unbound/) | BSD-3-Clause | DNS resolver |
| [Nginx](https://nginx.org/) | BSD-2-Clause | Internal reverse proxy |

## Contributing

Contributions are welcome! Please open an issue or PR.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

**Copyright (c) 2026 Ralph Wagter / athide.nl**

---

> *"Expose only what provides value to the world; hide the rest."*
