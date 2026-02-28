# Sovereign Infrastructure — Deployment Plan

## Context

A professional, sovereign environment on a Hetzner VPS with four core services: identity/MFA (Keycloak), collaboration (Nextcloud), website (Ghost), and email (Mailcow). The current cax21 (8GB RAM) will be upgraded to a **cax31 (16GB RAM, 8 vCPU, 160GB disk)** to comfortably run all services.

**Architecture Principles:** Zero Trust, Assume Breach, Defense in Depth. Each service runs in its own isolated Docker Compose stack with its own secrets, its own internal networks, and a minimal attack surface. A breach in one service must not allow lateral movement to other services.

---

## Decisions

| Choice | Decision | Reason |
|--------|----------|--------|
| Server | cax31 (16GB, 8 vCPU, 160GB) | Mailcow + all services comfortable |
| Mail | Mailcow | Full featured, webmail (SOGo), antispam (Rspamd), antivirus (ClamAV) |
| Reverse proxy | Traefik v2.11 (existing) | Already configured, Let's Encrypt |
| Identity | Keycloak 26.x | SSO/OIDC/MFA for all services |
| Collaboration | Nextcloud 30.x | Files, calendar, contacts |
| Website | Ghost 5.x | CMS/blog |
| **Isolation** | **1 service = 1 compose stack** | **Assume breach: limit blast radius** |

---

## Zero Trust Architecture

### Design Principles

1. **Each service is an autonomous unit** — its own docker-compose.yml, its own secrets, its own internal networks
2. **Only shared resource: the `proxy` network** — only for Traefik HTTP routing
3. **No service can reach another service** — unless explicitly and minimally required (only `net-auth` for OIDC)
4. **Databases are invisible** — each DB sits in an `--internal` network, only reachable by its own application
5. **Compromise of one stack does not leak to other stacks** — separate compose projects cannot see/reach each other's containers
6. **Secrets are isolated per stack** — Keycloak secrets are not readable from Ghost, etc.
7. **Docker socket is shielded** — via read-only proxy with minimal API endpoints
8. **Data Plane / Control Plane separation** — public services via internet, admin interfaces ONLY via Headscale VPN

### Data Plane vs Control Plane (Tailscale)

Strict separation between what the internet may see and what only the administrator may access:

| Service / Path | Access | Reason |
|----------------|--------|--------|
| Ghost website (`example.com`) | **Public** | SEO, readers |
| Ghost admin (`example.com/ghost`) | **Headscale VPN + Keycloak MFA + Ghost auth** | Triple lock |
| Nextcloud (`cloud.example.com`) | **Public** | Access files from anywhere |
| Keycloak login (`auth.example.com/realms/*`) | **Public** | OIDC flow for Nextcloud |
| Keycloak admin (`auth.example.com/admin/*`) | **Headscale VPN ONLY** | Configuration = critical |
| Mailcow webmail (`mail.example.com`) | **Public** | SOGo webmail for users |
| Mailcow admin (`mail.example.com/admin`) | **Headscale VPN ONLY** | Mailbox management = critical |
| Traefik dashboard (`traefik.example.com`) | **Headscale VPN ONLY** | Infrastructure insight = private |
| Mail protocols (25, 587, 993) | **Public** | Mail must work |

**Result:** A port scan on the public IP reveals NO admin interfaces. They "do not exist" for the outside world (403 Forbidden). An attacker must (1) be on the Headscale VPN network, (2) crack Keycloak MFA, and (3) have service-specific credentials.

### Blast Radius Analysis (Assume Breach)

| Compromised service | What can the attacker reach? | What can the attacker NOT reach? |
|---------------------|------------------------------|----------------------------------|
| Ghost + MariaDB | Ghost content, Ghost DB | Keycloak, Nextcloud, Mailcow, all other DBs |
| Nextcloud + PostgreSQL | Nextcloud files, NC DB, Redis cache | Keycloak admin, Ghost, Mailcow, Keycloak DB |
| Keycloak + PostgreSQL | Identity data, OIDC tokens | Nextcloud files, Ghost content, Mailcow, other DBs |
| Mailcow | Email, contacts via mail | Keycloak, Nextcloud files, Ghost, all app DBs |
| Traefik | HTTP routing (can redirect traffic) | No database access, no application data |

### Network Topology

```
                          INTERNET
                             │
                    ┌────────┴────────┐
                    │   UFW Firewall   │
                    │ 22,80,443,25,   │
                    │ 587,993,995,4190│
                    └────────┬────────┘
                             │
     ════════════════════════╪════════════════════════════
     ║  STACK: traefik       │       /opt/traefik/      ║
     ║  ┌────────────────────┴──────────────────────┐   ║
     ║  │              Traefik                      │   ║
     ║  │         (TLS termination)                 │   ║
     ║  └──────────────┬────────────────────────────┘   ║
     ║     net-socket-proxy (internal)                  ║
     ║         │                                        ║
     ║  ┌──────┴──────┐                                 ║
     ║  │socket-proxy │  (read-only Docker API)         ║
     ║  └─────────────┘                                 ║
     ════════════╪══════════════════════════════════════
                 │
          proxy (external network, HTTP only)
           ┌─────┼──────────┬──────────┬──────────┐
           │     │          │          │          │
     ══════╪═════╪════ ═════╪════ ═════╪════ ═════╪════════
     ║ STACK:    │   ║ ║    │   ║ ║    │   ║ ║    │       ║
     ║ keycloak  │   ║ ║ nextcl.│   ║ ║ ghost │   ║ ║ mailcow   ║
     ║     ┌─────┴─┐ ║ ║ ┌──┴──┐║ ║ ┌──┴──┐║ ║ ┌──┴──────┐║
     ║     │Keyclo.│ ║ ║ │Next │║ ║ │Ghost│║ ║ │Mailcow  ║║
     ║     │ :8080 │ ║ ║ │cloud│║ ║ │:2368│║ ║ │nginx    ║║
     ║     └───┬───┘ ║ ║ └─┬─┬─┘║ ║ └──┬──┘║ ║ └────────┘║║
     ║    net-kc-db  ║ ║net-nc │ ║ ║net-gh  ║ ║  Mailcow  ║║
     ║   (internal)  ║ ║-db  net║ ║-db     ║ ║  own      ║║
     ║     │         ║ ║(int)nc ║ ║(int)   ║ ║  network  ║║
     ║  ┌──┴───┐     ║ ║│  -cac║ ║│       ║ ║ ┌────────┐║║
     ║  │PG-KC │     ║ ║│  (in)║ ║│       ║ ║ │Postfix │║║
     ║  └──────┘     ║ ║│   │  ║ ║┌──┴──┐ ║ ║ │Dovecot │║║
     ║               ║ ║┌┴─┐│  ║ ║│Mari │ ║ ║ │Rspamd  │║║
     ║  ┌─────────┐  ║ ║│PG││  ║ ║│aDB  │ ║ ║ │ClamAV  │║║
     ║  │oauth2-  │  ║ ║│NC│┌┴─┐║ ║└─────┘ ║ ║ │SOGo    │║║
     ║  │proxy    │  ║ ║└──┘│Re││║ ║        ║ ║ │MariaDB │║║
     ║  └─────────┘  ║ ║    │dis║ ║        ║ ║ │Redis   │║║
     ║               ║ ║    └──┘║ ║        ║ ║ └────────┘║║
     ══════╪═════════ ═════╪════ ══════════ ══════════════
           │               │
     net-auth (internal) ──┘
     (only Keycloak ↔ Nextcloud OIDC
      + oauth2-proxy)
```

---

## Isolated Stack Structure

### Directory Layout on Server

```
/opt/
├── traefik/                        # STACK 1: Reverse Proxy
│   ├── docker-compose.yml
│   ├── traefik.yml
│   ├── dynamic/
│   │   └── middlewares.yml
│   └── acme.json
│
├── keycloak/                       # STACK 2: Identity (SECURED)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   ├── admin_user
│   │   └── admin_pass
│   └── .env
│
├── nextcloud/                      # STACK 3: Collaboration (SECURED)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   ├── admin_user
│   │   ├── admin_pass
│   │   └── redis_pass
│   └── .env
│
├── ghost/                          # STACK 4: Website (SECURED)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   └── db_root_pass
│   └── .env
│
├── mailcow-dockerized/             # STACK 5: Email (SELF-MANAGED)
│   ├── docker-compose.yml          # Mailcow-managed
│   ├── mailcow.conf
│   └── ...
│
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── healthcheck.sh
│
└── backups/                        # Local backup staging
```

**Why no shared secrets directory:**
- Keycloak secrets are not readable by Ghost containers (different compose projects, different mount paths)
- On breach of Ghost, the attacker cannot access Keycloak admin credentials
- Each stack only has access to its own `/opt/<stack>/secrets/`

### Shared External Networks

Only **two** networks are shared between stacks. These are created in advance:

| Network | Type | Created by | Used by |
|---------|------|------------|---------|
| `proxy` | bridge, external | Pre-created (`docker network create`) | Traefik, Keycloak, Nextcloud, Ghost, Mailcow-nginx |
| `net-auth` | bridge, internal, external | Pre-created (`docker network create --internal`) | Keycloak, Nextcloud, oauth2-proxy |

All other networks are **stack-internal** (defined in that stack's docker-compose.yml, not shared).

---

## Phase 0: Server Upgrade & Storage Security

### 0.1 Server Upgrade cax21 → cax31
**Via Hetzner Cloud API:**
1. Graceful server shutdown (`POST /servers/{id}/actions/shutdown`)
2. Change server type to `cax31` (`POST /servers/{id}/actions/change_type`, `upgrade_disk: true`)
3. Start server (`POST /servers/{id}/actions/poweron`)
4. Verify: 16GB RAM, 8 vCPU, 160GB disk
5. Add 2GB swap as safety net (`vm.swappiness=5`)

### 0.2 ARM64 Image Compatibility Test
Before deploying anything: test-pull of **all** images on ARM64 to verify compatibility:
```bash
for img in \
  "quay.io/keycloak/keycloak:26.0" \
  "postgres:16-alpine" \
  "nextcloud:30-apache" \
  "redis:7-alpine" \
  "ghost:5-alpine" \
  "mariadb:11-jammy" \
  "quay.io/oauth2-proxy/oauth2-proxy:v7.6.0" \
  "tecnativa/docker-socket-proxy:latest" \
  "traefik:v2.11"; do
  echo "Testing: $img"
  docker pull "$img" && docker run --rm "$img" echo "OK: $img" || echo "FAIL: $img"
done
```
**Specifically Mailcow:** After cloning mailcow-dockerized, run `docker compose pull` and verify that all Mailcow containers (including ClamAV, Rspamd) start correctly on ARM64.

### 0.3 Data-at-Rest Encryption (LUKS)
**Reason:** Hetzner cloud disks are not encrypted by default. For a sovereign infrastructure, data-at-rest encryption is essential — a physically stolen or recycled disk must not contain readable data.

**Approach:** Separate LUKS-encrypted volume for `/opt` (all application data):
```bash
# Create Hetzner volume (separate from root disk)
# Via API: POST /volumes {name: "data-encrypted", size: 80, server: <REDACTED-SERVER-ID>}

# LUKS setup on the volume
cryptsetup luksFormat /dev/disk/by-id/<volume-id>
cryptsetup open /dev/disk/by-id/<volume-id> data-encrypted
mkfs.ext4 /dev/mapper/data-encrypted
mount /dev/mapper/data-encrypted /opt
```

**Reboot implication:** After a reboot, the volume must be manually unlocked via Hetzner Console or SSH with: `cryptsetup open /dev/disk/by-id/<volume-id> data-encrypted && mount /dev/mapper/data-encrypted /opt`. Services only start after mount.

> **Deliberate choice AGAINST ZFS:** Hetzner Cloud VPS uses network-attached storage, not local disks. ZFS on network storage adds complexity without the full benefits (no hardware-level snapshots). ZFS ARC cache would also unnecessarily consume RAM on a 16GB server. LVM snapshots or our backup strategy (pg_dump + rsync + GPG) provide sufficient rollback capabilities.

---

## Phase 1: Infrastructure Preparation

### 1.1 Docker Daemon Hardening
**File:** `/etc/docker/daemon.json`
```json
{
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "storage-driver": "overlay2"
}
```
> `userns-remap` is NOT used (Mailcow incompatible). Compensation: per-container hardening.

### 1.2 Create External Networks
```bash
docker network create proxy                           # Existing, already present
docker network create --driver bridge --internal net-auth
```

### 1.3 UFW Firewall (Complete)

> **CRITICAL: Docker-UFW Trap.** Docker manipulates `iptables` directly and bypasses UFW rules. A `ports: "5432:5432"` in docker-compose is open to the world, even if UFW blocks it. **Our mitigation:** Only Traefik and Mailcow map ports to the host. All databases and internal services have NO `ports:` section — only `expose:` or purely internal networks. Verify this in every docker-compose.yml.

```bash
# Default deny
ufw default deny incoming
ufw default allow outgoing

# SSH
ufw allow 22/tcp comment 'SSH Management'

# Web (Traefik)
ufw allow 80/tcp comment 'HTTP (Redirect to HTTPS)'
ufw allow 443/tcp comment 'HTTPS (Traefik)'

# Mail (Mailcow — direct on host)
ufw allow 25/tcp comment 'SMTP (Mail delivery)'
ufw allow 465/tcp comment 'SMTPS (Submission)'
ufw allow 587/tcp comment 'STARTTLS (Submission)'
ufw allow 993/tcp comment 'IMAPS (Mail access)'
ufw allow 4190/tcp comment 'ManageSieve (Sieve filters)'

# Tailscale
ufw allow 41641/udp comment 'Tailscale Direct Connection'
ufw allow in on tailscale0 comment 'Allow all from Tailscale'

# Enable
ufw enable
```

### 1.4 Host Hardening (final review)

**Time synchronization (CRITICAL for TOTP/MFA):**
```bash
# Install and verify Chrony — TOTP is sensitive to time differences >30s
apt install chrony
systemctl enable chrony
chronyc tracking  # Verify offset < 1 second
```

**SSH hardening** (`/etc/ssh/sshd_config` additions):
```
AllowAgentForwarding no
AllowStreamLocalForwarding no
AllowTcpForwarding no
HostKeyAlgorithms ssh-ed25519
PubkeyAcceptedKeyTypes ssh-ed25519
```

**Log rotation:** Docker json-file limits are configured correctly (10m/3 files), but also verify host logs:
```bash
# Check /var/log size after Falco + Traefik deployment
du -sh /var/log/*
# Configure logrotate for /var/log/backup.log
```

### 1.5 Directories & Permissions
```bash
# Each stack gets its own directory with its own secrets
for stack in keycloak nextcloud ghost; do
  mkdir -p /opt/$stack/secrets
  chmod 700 /opt/$stack/secrets
  chown root:root /opt/$stack/secrets
done
mkdir -p /opt/traefik/dynamic
mkdir -p /opt/scripts /opt/backups
```

### 1.6 Generate Secrets (isolated per stack)
```bash
# Keycloak secrets
openssl rand -base64 32 > /opt/keycloak/secrets/db_pass
echo "keycloak" > /opt/keycloak/secrets/db_user
openssl rand -base64 32 > /opt/keycloak/secrets/admin_pass
echo "admin" > /opt/keycloak/secrets/admin_user

# Nextcloud secrets
openssl rand -base64 32 > /opt/nextcloud/secrets/db_pass
echo "nextcloud" > /opt/nextcloud/secrets/db_user
openssl rand -base64 32 > /opt/nextcloud/secrets/admin_pass
echo "admin" > /opt/nextcloud/secrets/admin_user
openssl rand -base64 32 > /opt/nextcloud/secrets/redis_pass

# Ghost secrets
openssl rand -base64 32 > /opt/ghost/secrets/db_pass
echo "ghost" > /opt/ghost/secrets/db_user
openssl rand -base64 32 > /opt/ghost/secrets/db_root_pass

# Lock everything down
chmod 600 /opt/*/secrets/*
```

---

## Phase 1.6: Headscale VPN (Self-Hosted Tailscale)

**Why Headscale instead of Tailscale:** Tailscale's control plane runs on Tailscale Inc's servers — an external dependency that does not fit a sovereign architecture. [Headscale](https://github.com/juanfont/headscale) is an open-source (BSD-3) self-hosted implementation of the Tailscale control server. Full control over coordination infrastructure, user auth, and network policy. Compatible with official Tailscale clients on all platforms.

**Installation on the server:**
```yaml
# /opt/headscale/docker-compose.yml
services:
  headscale:
    image: headscale/headscale:0.26
    container_name: headscale
    command: serve
    volumes:
      - ./config:/etc/headscale:ro
      - ./lib:/var/lib/headscale
    tmpfs:
      - /var/run/headscale
    ports:
      - "8080:8080"        # API + gRPC
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          memory: 128M
    restart: on-failure:5
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.headscale.rule=Host(`vpn.example.com`)"
      - "traefik.http.routers.headscale.entrypoints=websecure"
      - "traefik.http.routers.headscale.tls.certresolver=letsencrypt"
      - "traefik.http.services.headscale.loadbalancer.server.port=8080"
    networks:
      - proxy
networks:
  proxy:
    external: true
```

**Client setup (on your local machines):**
```bash
# Install Tailscale client (compatible with Headscale)
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --login-server https://vpn.example.com
```

**After installation:** Note the Tailscale IP (100.x.y.z). This IP is used in the `management-whitelist` middleware.

> **Fallback:** SSH on port 22 remains always available as a direct fallback, independent of Headscale status.

---

## Phase 2: Extend Traefik Stack (`/opt/traefik/`)

### 2.1 Add Docker Socket Proxy
Traefik loses direct `/var/run/docker.sock` access. A `tecnativa/docker-socket-proxy` only passes through read-only container info.

**Add to `/opt/traefik/docker-compose.yml`:**
```yaml
services:
  socket-proxy:
    image: tecnativa/docker-socket-proxy:latest
    environment:
      CONTAINERS: 1
      SERVICES: 0
      TASKS: 0
      NETWORKS: 0
      VOLUMES: 0
      IMAGES: 0
      INFO: 0
      POST: 0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - socket-proxy
    deploy:
      resources:
        limits:
          memory: 32M
    security_opt:
      - no-new-privileges:true
    read_only: true
    restart: unless-stopped

  traefik:
    # ... modify existing config:
    # REMOVE: /var/run/docker.sock volume mount
    # ADD: --providers.docker.endpoint=tcp://socket-proxy:2375
    networks:
      - proxy
      - socket-proxy

networks:
  proxy:
    external: true
  socket-proxy:
    driver: bridge
    internal: true   # socket-proxy cannot reach the internet
```

### 2.2 Traefik Dynamic Config
**New file:** `/opt/traefik/dynamic/middlewares.yml`

```yaml
http:
  middlewares:
    # === Security Headers ===
    security-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        frameDeny: true
        customFrameOptionsValue: "SAMEORIGIN"
        referrerPolicy: "strict-origin-when-cross-origin"
        permissionsPolicy: "camera=(), microphone=(), geolocation=(), payment=()"

    # === Rate Limiting (3 tiers) ===
    rate-limit:
      rateLimit:
        average: 100
        burst: 200
        period: 1m
    rate-limit-strict:
      rateLimit:
        average: 20
        burst: 50
        period: 1m
    rate-limit-login:
      rateLimit:
        average: 5
        burst: 10
        period: 1m

    # === Tailscale Management Whitelist ===
    management-whitelist:
      ipWhiteList:
        sourceRange:
          - "100.64.0.0/10"   # Tailscale CGNAT range
          - "127.0.0.1/32"    # Localhost

    # === Keycloak Forward Auth (via oauth2-proxy) ===
    auth-keycloak:
      forwardAuth:
        address: "http://oauth2-proxy:4180"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-Auth-Request-User"
          - "X-Auth-Request-Email"
          - "X-Auth-Request-Access-Token"

    # === WWW Redirect ===
    www-redirect:
      redirectRegex:
        regex: "^https://www\\.example\\.com/(.*)"
        replacement: "https://example.com/${1}"
        permanent: true

    # === Nextcloud DAV Redirects ===
    nextcloud-redirects:
      redirectRegex:
        regex: "https://(.*)/.well-known/(?:card|cal)dav"
        replacement: "https://${1}/remote.php/dav"
        permanent: true
```

### 2.3 Traefik Dashboard → Tailscale Only
```yaml
labels:
  - "traefik.http.routers.traefik.rule=Host(`traefik.example.com`)"
  - "traefik.http.routers.traefik.middlewares=management-whitelist@file,security-headers@file"
  - "traefik.http.routers.traefik.service=api@internal"
```
> **Basic Auth removed.** Dashboard only reachable via Tailscale.

### 2.4 Traefik Static Config Update
**File:** `/opt/traefik/traefik.yml`
- Add: `file` provider for `/opt/traefik/dynamic/`
- Change Docker endpoint to `tcp://socket-proxy:2375`

---

## Phase 3: Keycloak Stack (`/opt/keycloak/`)

**Dedicated docker-compose.yml with:**

| Container | Image | Networks | Memory Limit |
|-----------|-------|----------|-------------|
| `keycloak` | `quay.io/keycloak/keycloak:26.0` | `proxy`, `net-auth`, `kc-internal` | 1024MB |
| `postgres-keycloak` | `postgres:16-alpine` | `kc-internal` only | 512MB (max_connections=50) |
| `oauth2-proxy` | `quay.io/oauth2-proxy/oauth2-proxy:v7.6.0` | `proxy`, `net-auth` | 64MB |

**Internal networks (stack-owned):**
- `kc-internal` (bridge, internal): Keycloak ↔ PostgreSQL

**External networks (shared):**
- `proxy`: Traefik routing
- `net-auth`: OIDC for Nextcloud

**Keycloak path-based routing (public vs admin):**
```yaml
labels:
  # Public: OIDC flows for Nextcloud and other clients
  - "traefik.http.routers.keycloak-public.rule=Host(`auth.example.com`) && (PathPrefix(`/realms`) || PathPrefix(`/resources`) || PathPrefix(`/js`))"
  - "traefik.http.routers.keycloak-public.middlewares=security-headers@file,rate-limit@file"
  # Admin console: ONLY via Tailscale
  - "traefik.http.routers.keycloak-admin.rule=Host(`auth.example.com`) && PathPrefix(`/admin`)"
  - "traefik.http.routers.keycloak-admin.middlewares=management-whitelist@file,security-headers@file"
```

**Hardening per container:**
- `keycloak`: read_only, no-new-privileges, cap_drop ALL, tmpfs /tmp, user 1000
- `postgres-keycloak`: read_only, no-new-privileges, cap_drop ALL, tmpfs /tmp + /run/postgresql
- `oauth2-proxy`: read_only, no-new-privileges, cap_drop ALL, user 65534

**oauth2-proxy configuration (in this stack):**
```yaml
oauth2-proxy:
  image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
  environment:
    OAUTH2_PROXY_PROVIDER: "keycloak-oidc"
    OAUTH2_PROXY_OIDC_ISSUER_URL: "https://auth.example.com/realms/main"
    OAUTH2_PROXY_CLIENT_ID: "oauth2-proxy"
    OAUTH2_PROXY_CLIENT_SECRET_FILE: "/run/secrets/oauth2_client_secret"
    OAUTH2_PROXY_COOKIE_SECRET_FILE: "/run/secrets/oauth2_cookie_secret"
    OAUTH2_PROXY_COOKIE_DOMAINS: ".example.com"  # SSO across all subdomains
    OAUTH2_PROXY_COOKIE_SECURE: "true"
    OAUTH2_PROXY_COOKIE_HTTPONLY: "true"
    OAUTH2_PROXY_COOKIE_SAMESITE: "lax"
    OAUTH2_PROXY_REVERSE_PROXY: "true"
    OAUTH2_PROXY_EMAIL_DOMAINS: "*"            # Filtering via Keycloak roles
    OAUTH2_PROXY_WHITELIST_DOMAINS: ".example.com"
    OAUTH2_PROXY_SET_XAUTHREQUEST: "true"
    OAUTH2_PROXY_HTTP_ADDRESS: "0.0.0.0:4180"
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.oauth2.rule=Host(`auth.example.com`) && PathPrefix(`/oauth2`)"
    - "traefik.http.routers.oauth2.entrypoints=websecure"
    - "traefik.http.routers.oauth2.tls.certresolver=letsencrypt"
    - "traefik.http.services.oauth2.loadbalancer.server.port=4180"
  networks:
    - proxy
    - net-auth
  secrets:
    - oauth2_client_secret
    - oauth2_cookie_secret
  security_opt:
    - no-new-privileges:true
  read_only: true
  cap_drop:
    - ALL
  user: "65534:65534"
  tmpfs:
    - /tmp
  deploy:
    resources:
      limits:
        memory: 64M
  restart: on-failure:5
```

**After deployment:**
1. Create realm `main`
2. Set MFA (TOTP) as mandatory
3. Create OIDC clients:
   - `oauth2-proxy` (confidential, redirect URI: `https://auth.example.com/oauth2/callback`)
   - `nextcloud` (for direct OIDC integration)
4. **RBAC:** Create role `admin_access` in Keycloak. Only users with this role gain access to admin panels via oauth2-proxy (`OAUTH2_PROXY_ALLOWED_ROLES=admin_access`). This prevents regular Nextcloud users from accessing Ghost admin or the Traefik dashboard.
5. Enable brute force detection
6. Password policy: min 12 characters, uppercase, digit, special character

---

## Phase 4: Ghost Stack (`/opt/ghost/`)

**Dedicated docker-compose.yml with:**

| Container | Image | Networks | Memory Limit |
|-----------|-------|----------|-------------|
| `ghost` | `ghost:5-alpine` | `proxy`, `ghost-internal` | 384MB |
| `mariadb-ghost` | `mariadb:11-jammy` | `ghost-internal` only | 384MB |

**Internal networks (stack-owned):**
- `ghost-internal` (bridge, internal): Ghost ↔ MariaDB

**External networks (shared):**
- `proxy`: Traefik routing

**Ghost has NO access to `net-auth`** — Ghost admin is protected via Traefik forward-auth (oauth2-proxy), not via direct OIDC integration in Ghost itself.

> **Important (2nd opinion):** Ghost does NOT support native Header Authentication or OIDC. Forward-auth via oauth2-proxy is an **additional lock on the door** (access control before the Ghost login page), but it is NOT True SSO. Within Ghost, a strong password + Ghost's own 2FA is still required. The forward-auth prevents unauthorized users from even reaching the login page.

**Traefik routing (path-based):**
```yaml
labels:
  # Public website
  - "traefik.http.routers.ghost.rule=Host(`example.com`)"
  - "traefik.http.routers.ghost.middlewares=security-headers@file,rate-limit@file"
  # Admin panel: Tailscale whitelist + Keycloak forward-auth
  - "traefik.http.routers.ghost-admin.rule=Host(`example.com`) && PathPrefix(`/ghost`)"
  - "traefik.http.routers.ghost-admin.middlewares=management-whitelist@file,auth-keycloak@file,security-headers@file"
```

**Subdomain:** `example.com` + `www.example.com` (redirect)

**Hardening:**
- `ghost`: no-new-privileges, cap_drop ALL, tmpfs /tmp
- `mariadb-ghost`: read_only, no-new-privileges, cap_drop ALL, tuned buffer pool

---

## Phase 5: Nextcloud Stack (`/opt/nextcloud/`)

**Dedicated docker-compose.yml with:**

| Container | Image | Networks | Memory Limit |
|-----------|-------|----------|-------------|
| `nextcloud` | `nextcloud:30-apache` | `proxy`, `net-auth`, `nc-internal`, `nc-cache` | 1024MB |
| `postgres-nextcloud` | `postgres:16-alpine` | `nc-internal` only | 512MB (max_connections=50) |
| `redis` | `redis:7-alpine` | `nc-cache` only | 128MB |

**Internal networks (stack-owned):**
- `nc-internal` (bridge, internal): Nextcloud ↔ PostgreSQL
- `nc-cache` (bridge, internal): Nextcloud ↔ Redis

**External networks (shared):**
- `proxy`: Traefik routing
- `net-auth`: OIDC with Keycloak

**Subdomain:** `cloud.example.com`

**After deployment:**
1. OIDC integration with Keycloak (`user_oidc` app)
2. MFA via Keycloak (not Nextcloud's own TOTP)
3. Background jobs via system cron
4. **Limit preview generation** in `config.php` (prevent CPU 100% on large photo uploads):
   ```php
   'enable_previews' => true,
   'enabledPreviewProviders' => [
     'OC\Preview\PNG', 'OC\Preview\JPEG', 'OC\Preview\GIF',
     'OC\Preview\PDF', 'OC\Preview\MP3', 'OC\Preview\TXT',
   ],
   'preview_max_x' => 1024,
   'preview_max_y' => 1024,
   ```

---

## Phase 6: Mailcow Stack (`/opt/mailcow-dockerized/`)

### 6.1 Installation
```bash
cd /opt
git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
./generate_config.sh  # hostname: mail.example.com
```

### 6.2 Traefik Integration
**Modifications to `mailcow.conf`:**
```
HTTP_PORT=8080
HTTP_BIND=127.0.0.1
HTTPS_PORT=8443
HTTPS_BIND=127.0.0.1
SKIP_LETS_ENCRYPT=y
```

Mailcow-nginx gets Traefik labels + connection to external `proxy` network.

> **CRITICAL (2nd opinion): Proxy-on-Proxy problem.** Mailcow uses Nginx internally as a proxy. Traefik sits in front of that. Without correct `X-Forwarded-For` / `proxy_protocol` configuration, Rspamd sees all mail as originating from Traefik's IP → antispam is then completely blind. **Solution:** In `mailcow.conf`, correctly set `TRUSTED_NETWORK` to the Docker proxy network subnet, and in Traefik's entrypoint configure `proxyProtocol` + `forwardedHeaders` for the Mailcow-nginx backend.

**Restrict Mailcow admin UI:**
```yaml
# In Mailcow's nginx Traefik labels:
labels:
  # Webmail (SOGo) - public
  - "traefik.http.routers.mailcow.rule=Host(`mail.example.com`) && !PathPrefix(`/admin`)"
  - "traefik.http.routers.mailcow.middlewares=security-headers@file,rate-limit@file"
  # Admin UI - Tailscale only
  - "traefik.http.routers.mailcow-admin.rule=Host(`mail.example.com`) && PathPrefix(`/admin`)"
  - "traefik.http.routers.mailcow-admin.middlewares=management-whitelist@file,security-headers@file"
```

**Mail ports direct on host** (not via Traefik):
25, 587, 993, 995, 4190

### 6.3 Mailcow Containers (self-managed, own network)
Postfix, Dovecot, Rspamd, ClamAV, SOGo, MariaDB, Redis, Nginx — all in Mailcow's own internal network. **No Mailcow container (except nginx) touches the `proxy` network.**

### 6.4 Mailcow Memory Budget: ~2.5GB total

### 6.5 DNS Records (at DNS provider)

| Type | Name | Value |
|------|------|-------|
| MX | `example.com` | `mail.example.com` (priority 10) |
| A | `mail` | `<REDACTED-IP>` |
| TXT | `example.com` | `v=spf1 mx a ip4:<REDACTED-IP> -all` |
| TXT | `dkim._domainkey` | *(generated by Mailcow)* |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:dmarc@example.com` |
| TXT | `_smtp._tls` | `v=TLSRPTv1; rua=mailto:tls@example.com` |
| SRV | `_autodiscover._tcp` | `0 0 443 mail.example.com` |
| CNAME | `autoconfig` | `mail.example.com` |

**DMARC progression:** `p=none` → `p=quarantine` (week 3) → `p=reject` (month 3)

> **Hetzner blocks port 25 by default.** Submit a support ticket as the first step (1-3 business days).

---

## Phase 7: Hardening & Monitoring

### 7.1 Container Hardening Baseline (all stacks)
```yaml
security_opt:
  - no-new-privileges:true
  - seccomp=default                # Docker default seccomp profile (blocks ~44 syscalls)
  - apparmor=docker-default        # AppArmor default profile
cap_drop:
  - ALL
pids_limit: 200
restart: on-failure:5              # Prevent infinite restart loops (not unless-stopped)
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

### 7.2 Per-Container Security Matrix

| Stack | Container | read_only | cap_add | user | seccomp | apparmor |
|-------|-----------|-----------|---------|------|---------|----------|
| traefik | traefik | YES | NET_BIND_SERVICE | 65534 | default | docker-default |
| traefik | socket-proxy | YES | (none) | 65534 | default | docker-default |
| keycloak | keycloak | YES | (none) | 1000 | default | docker-default |
| keycloak | postgres-keycloak | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| keycloak | oauth2-proxy | YES | (none) | 65534 | default | docker-default |
| nextcloud | nextcloud | **NO** (\*) | NET_BIND_SERVICE, DAC_OVERRIDE, SETGID, SETUID, CHOWN, FOWNER | www-data | default | docker-default |
| nextcloud | postgres-nextcloud | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| nextcloud | redis | YES | (none) | 999 | default | docker-default |
| ghost | ghost | **NO** (\*) | NET_BIND_SERVICE, CHOWN, SETGID, SETUID | 1000 | default | docker-default |
| ghost | mariadb-ghost | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| mailcow | *(self-managed)* | | | | | |

(\*) Writable volumes needed for uploads/content. Volumes mounted with minimal paths — **never** mount `/etc`, `/var/lib/docker`, or other host system directories.

### 7.3 Docker Content Trust & Image Verification
```bash
# Enable Docker Content Trust (only pull signed images)
echo 'export DOCKER_CONTENT_TRUST=1' >> /etc/environment
```
- All images are pinned to specific versions (no `latest` tags)
- Weekly `docker pull` + restart to receive security patches

### 7.4 Vulnerability Scanning with Trivy
```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan all running images
docker images --format '{{.Repository}}:{{.Tag}}' | xargs -I {} trivy image --severity HIGH,CRITICAL {}
```
- **Weekly automated** via cron
- **At every deployment** for new images
- Alerts on HIGH/CRITICAL findings

### 7.5 Runtime Anomaly Detection with Falco
```yaml
# In /opt/monitoring/docker-compose.yml
falco:
  image: falcosecurity/falco:latest
  privileged: true    # Required for kernel syscall monitoring
  volumes:
    - /var/run/docker.sock:/host/var/run/docker.sock:ro
    - /proc:/host/proc:ro
    - /dev:/host/dev:ro
    - falco-config:/etc/falco
  deploy:
    resources:
      limits:
        memory: 256M
```
- Detects: shell spawning in containers, sensitive file reads, unexpected network connections, crypto mining
- Alerts via syslog or webhook

### 7.6 CIS Docker Benchmark Audit
```bash
# Docker Bench for Security (run regularly)
docker run --rm --net host --pid host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc:/etc:ro \
  docker/docker-bench-security
```
- Run after each deployment phase
- Document and mitigate all findings
- Target: no WARN or FAIL on critical controls

### 7.7 Deliberate Deviations (documented)

| Best Practice | Status | Reason |
|---------------|--------|--------|
| Rootless Docker | **NOT APPLIED** | Mailcow requires root Docker daemon; compensation: per-container user, cap_drop, no-new-privileges |
| `userns-remap` | **NOT APPLIED** | Mailcow incompatible; compensation: AppArmor + Seccomp + non-root users |
| Docker Swarm Secrets | **NOT APPLIED** | No Swarm; file-based secrets with chmod 600 as alternative |
| `unless-stopped` restart | **DELIBERATELY AVOIDED** | `on-failure:5` prevents infinite restart loops on misconfiguration |

### 7.8 Backup Strategy (3-2-1 Rule)

**Principle:** 3 copies, 2 media, 1 offsite. Encryption with **asymmetric GPG** — public key on server, private key ONLY on local machine. Even with full server compromise, backups cannot be decrypted.

**Preparation:**
1. Generate GPG keypair on **local machine** (not the server): `gpg --full-generate-key`
2. Export public key and upload to server: `gpg --export -a "<KEY-OWNER>" > public.key`
3. Import on server: `gpg --import public.key`
4. Secrets are EXCLUDED from backups (secrets belong in a password manager, not in backups)

**Backup script** (`/opt/scripts/backup.sh`):
```bash
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/opt/backups/staging"
DATE=$(date +%Y-%m-%d_%H%M%S)
GPG_RECIPIENT="<KEY-ID>"  # Public key ID
RETENTION_DAYS=30
LOG_FILE="/var/log/backup.log"
mkdir -p "$BACKUP_DIR"

log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"; }
log "--- Start Backup ---"

# 1. Database dumps (hot, no downtime)
docker exec postgres-keycloak pg_dump -U keycloak > "$BACKUP_DIR/keycloak_db_$DATE.sql"
docker exec postgres-nextcloud pg_dump -U nextcloud > "$BACKUP_DIR/nextcloud_db_$DATE.sql"
docker exec mariadb-ghost mariadb-dump --all-databases -u root \
  -p"$(docker exec mariadb-ghost cat /run/secrets/db_root_pass)" > "$BACKUP_DIR/ghost_db_$DATE.sql"

# 2. Config & data (WITHOUT secrets)
tar -czf "$BACKUP_DIR/configs_$DATE.tar.gz" \
  /opt/traefik /opt/keycloak /opt/nextcloud /opt/ghost \
  --exclude="*/secrets"
tar -czf "$BACKUP_DIR/nextcloud_data_$DATE.tar.gz" \
  /var/lib/docker/volumes/nextcloud_nextcloud-data/_data

# 3. Mailcow own backup
export MAILCOW_BACKUP_LOCATION="$BACKUP_DIR/mailcow"
mkdir -p "$MAILCOW_BACKUP_LOCATION"
cd /opt/mailcow-dockerized && ./helper-scripts/backup_and_restore.sh backup all --silent

# 4. GPG encryption (asymmetric)
for file in "$BACKUP_DIR"/*.{sql,gz}; do
  [ -f "$file" ] || continue
  gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$file"
  shred -u "$file"   # Secure delete unencrypted version
done

# 5. Offsite sync via rclone (Hetzner Storage Box, SFTP)
rclone sync "$BACKUP_DIR" storagebox:backups/infra/ -v --log-file="$LOG_FILE"

# 6. Cleanup
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete
log "--- Backup Complete ---"
```

**Cron:** `0 2 * * * /opt/scripts/backup.sh`

**Offsite:** Hetzner Storage Box (BX11, ~€3.81/month) via rclone (SFTP, port 23).

**Additional safety net:** Enable Hetzner Cloud Snapshots for the entire VPS — instant rollback on catastrophic failures.

**Retention:** 30 days local, 90 days offsite.

**Restore test:** Weekly restore Ghost DB in a temporary Docker environment.

**GPG Private Key management:** The private key is the most critical asset. If lost, ALL offsite backups are unusable. Store in at least 2 physically separated locations (e.g., 2 USB sticks in different safes, or YubiKey + paper backup).

### 7.9 Monitoring Stack
- **Node-exporter** (64MB) — system metrics (CPU, RAM, disk, network)
- **Falco** (256MB) — runtime anomaly detection, syscall monitoring
- **Trivy** — weekly vulnerability scans (no permanent container)
- **Healthcheck cron** (every 5 min) — service availability alerts
- **External uptime monitoring** (Hetrixtools or Uptime Kuma) — external perspective

---

## Memory Budget — cax31 (16GB)

| Stack | Component | Limit | Reservation |
|-------|-----------|-------|-------------|
| *OS* | kernel + Docker daemon | — | ~650MB |
| traefik | Traefik | 256MB | 128MB |
| traefik | socket-proxy | 32MB | 16MB |
| keycloak | Keycloak | 1024MB | 768MB |
| keycloak | PostgreSQL | 512MB | 256MB |
| keycloak | oauth2-proxy | 64MB | 32MB |
| nextcloud | Nextcloud | 1024MB | 768MB |
| nextcloud | PostgreSQL | 512MB | 256MB |
| nextcloud | Redis | 128MB | 64MB |
| ghost | Ghost | 384MB | 256MB |
| ghost | MariaDB | 384MB | 256MB |
| mailcow | *total* | ~3000MB | ~2500MB |
| monitoring | node-exporter | 64MB | 32MB |
| monitoring | Falco | 256MB | 128MB |
| vpn | Headscale | 128MB | 64MB |
| **TOTAL** | | **~7,768MB** | **~6,174MB** |
| **Available** | | **~15,500MB** | |
| **Headroom** | | **~7,732MB (50%)** | |

Swap: 2GB safety net, `vm.swappiness=5`.

---

## Subdomain Overview

| URL | Stack | Service |
|-----|-------|---------|
| `example.com` | ghost | Ghost (website) |
| `www.example.com` | ghost | Redirect → example.com |
| `auth.example.com` | keycloak | Keycloak (SSO/MFA) |
| `cloud.example.com` | nextcloud | Nextcloud |
| `mail.example.com` | mailcow | Mailcow (webmail/admin) |
| `traefik.example.com` | traefik | Dashboard (Tailscale only) |
| `vpn.example.com` | headscale | Headscale control server |

---

## Deployment Order

```
Phase 0  → Server upgrade cax21 → cax31 + LUKS encryption + ARM64 image test
Phase 1  → Infra (networks, directories, secrets, daemon.json, UFW, Headscale, host hardening)
Phase 2  → Stack: extend traefik (socket-proxy, middlewares, Tailscale whitelist)
Phase 3  → Stack: deploy keycloak + oauth2-proxy, configure realm + RBAC
Phase 4  → Stack: deploy ghost (forward-auth on /ghost)
Phase 5  → Stack: deploy nextcloud + OIDC integration + preview config
Phase 6  → Stack: deploy mailcow + Traefik integration + proxy_protocol config
Phase 7  → DNS records (MX, SPF, DKIM, DMARC) at DNS provider
Phase 8  → Monitoring: Falco + node-exporter + Trivy scan
Phase 9  → Backup setup (GPG asymmetric + rclone + Hetzner Snapshots)
Phase 10 → CIS Docker Bench audit + hardening review
Phase 11 → Validation (SSL Labs, nmap, incognito tests, cross-stack tests, SMTP egress)
```

---

## Validation Checklist

### TLS & Headers
- [ ] SSL Labs A+ rating on all subdomains
- [ ] securityheaders.com A+ rating

### Mail
- [ ] mail-tester.com score ≥ 9/10
- [ ] MXToolbox: SPF, DKIM, DMARC pass

### Container Hardening
- [ ] All containers: `no-new-privileges`, `cap_drop: ALL`
- [ ] All containers: Seccomp profile active (default)
- [ ] All containers: AppArmor profile active (docker-default)
- [ ] All containers: `restart: on-failure:5` (no `unless-stopped`)
- [ ] All containers: specific `user:` (non-root) defined
- [ ] All containers: memory + pids limits configured
- [ ] No container runs with `--privileged` (except Falco monitoring)
- [ ] No `latest` tags — all images pinned to specific version

### Network Isolation (Zero Trust)
- [ ] No container has direct Docker socket access (socket-proxy only)
- [ ] Databases not reachable from proxy network
- [ ] **Cross-stack test:** from Ghost container, `curl` to Keycloak/Nextcloud FAILS
- [ ] **Cross-stack test:** from Nextcloud, `curl` to Ghost MariaDB FAILS
- [ ] **Cross-stack test:** from Keycloak, `curl` to Nextcloud PostgreSQL FAILS
- [ ] All DB networks are `--internal` (no internet access)

### Secrets & Data
- [ ] Secrets isolated per stack (Ghost cannot read Keycloak secrets)
- [ ] No passwords in docker-compose.yml or .env (only file references)
- [ ] Docker Content Trust active (`DOCKER_CONTENT_TRUST=1`)
- [ ] No sensitive host directories mounted (`/etc`, `/var/lib/docker`)

### SMTP Egress Filtering
- [ ] **SMTP sinkhole test:** From Ghost container, `curl` to port 25 on internet FAILS — only Mailcow may speak SMTP
- [ ] **SMTP sinkhole test:** From Nextcloud container, outbound port 25 FAILS
- [ ] **SMTP sinkhole test:** From Keycloak container, outbound port 25 FAILS

### Admin Isolation (Tailscale)
- [ ] **Incognito test (VPN OFF):** `https://auth.example.com/admin` → 403 Forbidden
- [ ] **Incognito test (VPN OFF):** `https://mail.example.com/admin` → 403 Forbidden
- [ ] **Incognito test (VPN OFF):** `https://traefik.example.com` → 403 Forbidden
- [ ] **Incognito test (VPN ON):** `https://example.com/ghost` → Keycloak login → MFA → Ghost admin
- [ ] oauth2-proxy RBAC: user WITHOUT `admin_access` role → denied at Ghost admin

### Scanning & Audit
- [ ] Trivy scan: no CRITICAL vulnerabilities in running images
- [ ] Docker Bench for Security: no FAIL on critical controls
- [ ] Falco running and detects test anomaly (shell spawn in container)
- [ ] **Nmap scan** (from external machine, not on Tailscale): only ports 22, 25, 80, 443, 465, 587, 993, 4190 open. All others (8080, 5432, 3306, etc.) CLOSED/FILTERED
- [ ] No docker-compose.yml contains `ports:` for database containers

### Backup & Recovery
- [ ] Backup GPG encryption: backup file is not readable without private key
- [ ] Backup EXCLUDES `/opt/*/secrets/` directories
- [ ] Restore test: Ghost DB restore in temporary Docker environment successful
- [ ] Offsite backup to Hetzner Storage Box working via rclone

### Host Hardening
- [ ] Fail2ban active on SSH
- [ ] UFW: only ports 22, 80, 443, 25, 465, 587, 993, 4190, 41641/udp open
- [ ] UFW: Tailscale interface (`tailscale0`) trusted
- [ ] SSH: no root login, no password auth
- [ ] LUKS encryption active on data volume (`/opt`)

---

## Risks and Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| Traefik Docker socket access | MEDIUM | Socket-proxy (read-only, only containers endpoint) |
| `proxy` network as lateral route | MEDIUM | Containers expose only HTTP port; Seccomp + AppArmor restrict syscalls; no shell/exec |
| `net-auth` as lateral route | LOW | Only OIDC protocol; Keycloak and Nextcloud sole participants |
| Single server = SPOF | HIGH | Offsite encrypted backups, documented restore, weekly restore test |
| Hetzner port 25 blocked | MEDIUM | Support ticket as first action (phase 6) |
| ClamAV high memory usage | LOW | cax31 has 8GB headroom |
| Mailcow own container management | LOW | Battle-tested, own update/network; Falco monitors runtime behavior |
| Supply chain: compromised images | MEDIUM | Docker Content Trust, pinned versions, Trivy scanning, no `latest` tags |
| Container escape via kernel exploit | LOW | AppArmor + Seccomp + no-new-privileges + non-root users; regular kernel updates |
| Falco runs privileged | LOW | Required for syscall monitoring; compensation: Falco is read-only, no network connections |
| Rootless Docker not possible | MEDIUM | Mailcow requires root daemon; compensation: per-container hardening on all other layers |

### Security Best Practices References
- [OneUptime: Docker Security Best Practices (2026)](https://oneuptime.com/blog/post/2026-02-02-docker-security-best-practices/view)
- [BetterStack: Docker Security Best Practices](https://betterstack.com/community/guides/scaling-docker/docker-security-best-practices/)
- [Cloud Native Now: Docker Security in 2025](https://cloudnativenow.com/editorial-calendar/best-of-2025/docker-security-in-2025-best-practices-to-protect-your-containers-from-cyberthreats-2/)
| Single server