# Soevereine Infrastructuur athide.nl — Deployment Plan

## Context

Ralph wil een zakelijke, soevereine omgeving op een Hetzner VPS met vier kernservices: identity/MFA (Keycloak), samenwerking (Nextcloud), website (Ghost) en e-mail (Mailcow). De huidige cax21 (8GB RAM) wordt geüpgraded naar een **cax31 (16GB RAM, 8 vCPU, 160GB disk)** om alle services comfortabel te draaien.

**Architectuurprincipes:** Zero Trust, Assume Breach, Defense in Depth. Elke service draait in zijn eigen geïsoleerde Docker Compose stack met eigen secrets, eigen interne netwerken, en een minimaal aanvalsoppervlak. Een breach in één service mag geen laterale beweging naar andere services mogelijk maken.

---

## Beslissingen

| Keuze | Besluit | Reden |
|-------|---------|-------|
| Server | cax31 (16GB, 8 vCPU, 160GB) | Mailcow + alle services comfortabel |
| Mail | Mailcow | Volledig features, webmail (SOGo), antispam (Rspamd), antivirus (ClamAV) |
| Reverse proxy | Traefik v2.11 (bestaand) | Al geconfigureerd, Let's Encrypt |
| Identity | Keycloak 26.x | SSO/OIDC/MFA voor alle services |
| Samenwerking | Nextcloud 30.x | Files, agenda, contacten |
| Website | Ghost 5.x | CMS/blog |
| **Isolatie** | **1 service = 1 compose stack** | **Assume breach: blast radius beperken** |

---

## Zero Trust Architectuur

### Ontwerpprincipes

1. **Elke service is een zelfstandige eenheid** — eigen docker-compose.yml, eigen secrets, eigen interne netwerken
2. **Enige gedeelde resource: het `proxy` netwerk** — alleen voor Traefik HTTP routing
3. **Geen service kan een andere service bereiken** — tenzij expliciet en minimaal nodig (alleen `net-auth` voor OIDC)
4. **Databases zijn onzichtbaar** — elke DB zit in een `--internal` netwerk, alleen bereikbaar door zijn eigen applicatie
5. **Compromis van één stack leakt niet naar andere stacks** — aparte compose projecten kunnen elkaars containers niet zien/bereiken
6. **Secrets zijn per-stack geïsoleerd** — Keycloak secrets zijn niet leesbaar vanuit Ghost, etc.
7. **Docker socket is afgeschermd** — via read-only proxy met minimale API endpoints
8. **Data Plane / Control Plane scheiding** — publieke services via internet, admin interfaces ALLEEN via Headscale VPN

### Data Plane vs Control Plane (Tailscale)

Strikte scheiding tussen wat het internet mag zien en wat alleen de beheerder mag bereiken:

| Service / Pad | Toegang | Reden |
|---------------|---------|-------|
| Ghost website (`athide.nl`) | **Publiek** | SEO, lezers |
| Ghost admin (`athide.nl/ghost`) | **Headscale VPN + Keycloak MFA + Ghost auth** | Triple lock |
| Nextcloud (`cloud.athide.nl`) | **Publiek** | Overal bij bestanden |
| Keycloak login (`auth.athide.nl/realms/*`) | **Publiek** | OIDC flow voor Nextcloud |
| Keycloak admin (`auth.athide.nl/admin/*`) | **Headscale VPN ONLY** | Configuratie = kritiek |
| Mailcow webmail (`mail.athide.nl`) | **Publiek** | SOGo webmail voor gebruikers |
| Mailcow admin (`mail.athide.nl/admin`) | **Headscale VPN ONLY** | Mailbox beheer = kritiek |
| Traefik dashboard (`traefik.athide.nl`) | **Headscale VPN ONLY** | Infra-inzicht = privé |
| Mail protocols (25, 587, 993) | **Publiek** | Mail moet werken |

**Resultaat:** Een portscan op het publieke IP toont GEEN admin interfaces. Ze "bestaan niet" voor de buitenwereld (403 Forbidden). Een aanvaller moet (1) op het Headscale VPN netwerk zitten, (2) Keycloak MFA kraken, en (3) service-specifieke credentials hebben.

### Blast Radius Analyse (Assume Breach)

| Gecompromitteerde service | Wat kan aanvaller bereiken? | Wat kan aanvaller NIET bereiken? |
|---------------------------|---------------------------|--------------------------------|
| Ghost + MariaDB | Ghost content, Ghost DB | Keycloak, Nextcloud, Mailcow, alle andere DBs |
| Nextcloud + PostgreSQL | Nextcloud bestanden, NC DB, Redis cache | Keycloak admin, Ghost, Mailcow, Keycloak DB |
| Keycloak + PostgreSQL | Identity data, OIDC tokens | Nextcloud bestanden, Ghost content, Mailcow, andere DBs |
| Mailcow | Email, contacten via mail | Keycloak, Nextcloud bestanden, Ghost, alle app DBs |
| Traefik | HTTP routing (kan verkeer omleiden) | Geen database toegang, geen applicatie data |

### Netwerk Topologie

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
          proxy (extern netwerk, alleen HTTP)
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
     ║   (internal)  ║ ║-db  net║ ║-db     ║ ║  eigen    ║║
     ║     │         ║ ║(int)nc ║ ║(int)   ║ ║  netwerk  ║║
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
     (alleen Keycloak ↔ Nextcloud OIDC
      + oauth2-proxy)
```

---

## Geïsoleerde Stack Structuur

### Directory Layout op Server

```
/opt/
├── traefik/                        # STACK 1: Reverse Proxy
│   ├── docker-compose.yml
│   ├── traefik.yml
│   ├── dynamic/
│   │   └── middlewares.yml
│   └── acme.json
│
├── keycloak/                       # STACK 2: Identity (BEVEILIGD)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   ├── admin_user
│   │   └── admin_pass
│   └── .env
│
├── nextcloud/                      # STACK 3: Samenwerking (BEVEILIGD)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   ├── admin_user
│   │   ├── admin_pass
│   │   └── redis_pass
│   └── .env
│
├── ghost/                          # STACK 4: Website (BEVEILIGD)
│   ├── docker-compose.yml
│   ├── secrets/                    # chmod 700 root:root
│   │   ├── db_user
│   │   ├── db_pass
│   │   └── db_root_pass
│   └── .env
│
├── mailcow-dockerized/             # STACK 5: E-mail (EIGEN BEHEER)
│   ├── docker-compose.yml          # Mailcow-beheerd
│   ├── mailcow.conf
│   └── ...
│
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── healthcheck.sh
│
└── backups/                        # Lokale backup staging
```

**Waarom geen gedeelde secrets directory:**
- Keycloak secrets zijn niet leesbaar door Ghost containers (verschillende compose projects, verschillende mount paths)
- Bij breach van Ghost kan aanvaller niet bij Keycloak admin credentials
- Elke stack heeft alleen toegang tot zijn eigen `/opt/<stack>/secrets/`

### Gedeelde Externe Netwerken

Slechts **twee** netwerken worden gedeeld tussen stacks. Deze worden vooraf aangemaakt:

| Netwerk | Type | Aangemaakt door | Gebruikt door |
|---------|------|-----------------|---------------|
| `proxy` | bridge, extern | Vooraf (`docker network create`) | Traefik, Keycloak, Nextcloud, Ghost, Mailcow-nginx |
| `net-auth` | bridge, intern, extern | Vooraf (`docker network create --internal`) | Keycloak, Nextcloud, oauth2-proxy |

Alle andere netwerken zijn **stack-intern** (gedefinieerd in de docker-compose.yml van die stack, niet gedeeld).

---

## Fase 0: Server Upgrade & Storage Security

### 0.1 Server Upgrade cax21 → cax31
**Via Hetzner Cloud API:**
1. Server graceful shutdown (`POST /servers/{id}/actions/shutdown`)
2. Server type wijzigen naar `cax31` (`POST /servers/{id}/actions/change_type`, `upgrade_disk: true`)
3. Server opstarten (`POST /servers/{id}/actions/poweron`)
4. Verifieer: 16GB RAM, 8 vCPU, 160GB disk
5. 2GB swap toevoegen als safety net (`vm.swappiness=5`)

### 0.2 ARM64 Image Compatibiliteit Test
Voordat er iets gedeployed wordt: test-pull van **alle** images op ARM64 om compatibiliteit te verifiëren:
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
**Specifiek Mailcow:** Na clone van mailcow-dockerized, `docker compose pull` uitvoeren en verifiëren dat alle Mailcow containers (incl. ClamAV, Rspamd) correct starten op ARM64.

### 0.3 Data-at-Rest Encryptie (LUKS)
**Reden:** Hetzner cloud disks zijn niet standaard versleuteld. Voor een soevereine infrastructuur is data-at-rest encryptie essentieel — een fysiek gestolen of gerecyclede disk mag geen leesbare data bevatten.

**Aanpak:** Apart LUKS-versleuteld volume voor `/opt` (alle applicatiedata):
```bash
# Hetzner volume aanmaken (apart van root disk)
# Via API: POST /volumes {name: "data-encrypted", size: 80, server: 122112572}

# LUKS setup op het volume
cryptsetup luksFormat /dev/disk/by-id/<volume-id>
cryptsetup open /dev/disk/by-id/<volume-id> data-encrypted
mkfs.ext4 /dev/mapper/data-encrypted
mount /dev/mapper/data-encrypted /opt
```

**Reboot-implicatie:** Na een reboot moet het volume handmatig ontgrendeld worden via Hetzner Console of SSH met: `cryptsetup open /dev/disk/by-id/<volume-id> data-encrypted && mount /dev/mapper/data-encrypted /opt`. Services starten pas na mount.

> **Bewuste keuze TEGEN ZFS:** Hetzner Cloud VPS gebruikt network-attached storage, geen lokale disks. ZFS op network storage voegt complexiteit toe zonder de volle voordelen (geen hardware-level snapshots). ZFS ARC cache zou ook onnodig RAM consumeren op een 16GB server. LVM snapshots of onze backup strategie (pg_dump + rsync + GPG) bieden voldoende rollback-mogelijkheden.

---

## Fase 1: Infrastructuur Voorbereiding

### 1.1 Docker Daemon Hardening
**Bestand:** `/etc/docker/daemon.json`
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
> `userns-remap` wordt NIET gebruikt (Mailcow incompatibel). Compensatie: per-container hardening.

### 1.2 Externe Netwerken Aanmaken
```bash
docker network create proxy                           # Bestaand, al aanwezig
docker network create --driver bridge --internal net-auth
```

### 1.3 UFW Firewall (Compleet)

> **KRITIEK: Docker-UFW Trap.** Docker manipuleert `iptables` direct en omzeilt UFW-regels. Een `ports: "5432:5432"` in docker-compose staat open voor de wereld, zelfs als UFW het blokkeert. **Onze mitigatie:** Alleen Traefik en Mailcow mappen poorten naar de host. Alle databases en interne services hebben GEEN `ports:` sectie — alleen `expose:` of puur interne netwerken. Verifieer dit in elke docker-compose.yml.

```bash
# Default deny
ufw default deny incoming
ufw default allow outgoing

# SSH
ufw allow 22/tcp comment 'SSH Management'

# Web (Traefik)
ufw allow 80/tcp comment 'HTTP (Redirect naar HTTPS)'
ufw allow 443/tcp comment 'HTTPS (Traefik)'

# Mail (Mailcow — direct op host)
ufw allow 25/tcp comment 'SMTP (Mail delivery)'
ufw allow 465/tcp comment 'SMTPS (Submission)'
ufw allow 587/tcp comment 'STARTTLS (Submission)'
ufw allow 993/tcp comment 'IMAPS (Mail access)'
ufw allow 4190/tcp comment 'ManageSieve (Sieve filters)'

# Tailscale
ufw allow 41641/udp comment 'Tailscale Direct Connection'
ufw allow in on tailscale0 comment 'Allow all from Tailscale'

# Activeer
ufw enable
```

### 1.4 Host Hardening (finale review)

**Tijdsynchronisatie (KRITIEK voor TOTP/MFA):**
```bash
# Chrony installeren en verifiëren — TOTP is gevoelig voor tijdsverschillen >30s
apt install chrony
systemctl enable chrony
chronyc tracking  # Verifieer offset < 1 seconde
```

**SSH hardening** (`/etc/ssh/sshd_config` toevoegingen):
```
AllowAgentForwarding no
AllowStreamLocalForwarding no
AllowTcpForwarding no
HostKeyAlgorithms ssh-ed25519
PubkeyAcceptedKeyTypes ssh-ed25519
```

**Logrotatie:** Docker json-file limits staan goed (10m/3 files), maar verifieer ook host-logs:
```bash
# Check /var/log grootte na Falco + Traefik deployment
du -sh /var/log/*
# Configureer logrotate voor /var/log/backup.log
```

### 1.5 Directories & Permissions
```bash
# Elke stack krijgt eigen directory met eigen secrets
for stack in keycloak nextcloud ghost; do
  mkdir -p /opt/$stack/secrets
  chmod 700 /opt/$stack/secrets
  chown root:root /opt/$stack/secrets
done
mkdir -p /opt/traefik/dynamic
mkdir -p /opt/scripts /opt/backups
```

### 1.6 Secrets Genereren (per stack geïsoleerd)
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

# Lock alles af
chmod 600 /opt/*/secrets/*
```

---

## Fase 1.6: Headscale VPN (Self-Hosted Tailscale)

**Waarom Headscale i.p.v. Tailscale:** Tailscale's control plane draait op Tailscale Inc's servers — een externe afhankelijkheid die niet past bij een soevereine architectuur. [Headscale](https://github.com/juanfont/headscale) is een open-source (BSD-3) self-hosted implementatie van de Tailscale control server. Volle controle over coördinatie-infrastructuur, user auth en netwerk-policy. Compatibel met officiële Tailscale clients op alle platformen.

**Installatie op de server:**
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
      - "traefik.http.routers.headscale.rule=Host(`vpn.athide.nl`)"
      - "traefik.http.routers.headscale.entrypoints=websecure"
      - "traefik.http.routers.headscale.tls.certresolver=letsencrypt"
      - "traefik.http.services.headscale.loadbalancer.server.port=8080"
    networks:
      - proxy
networks:
  proxy:
    external: true
```

**Client setup (op je lokale machines):**
```bash
# Installeer Tailscale client (compatibel met Headscale)
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --login-server https://vpn.athide.nl
```

**Na installatie:** Noteer het Tailscale IP (100.x.y.z). Dit IP wordt gebruikt in de `management-whitelist` middleware.

> **Fallback:** SSH op poort 22 blijft altijd beschikbaar als directe fallback, onafhankelijk van Headscale status.

---

## Fase 2: Traefik Stack Uitbreiden (`/opt/traefik/`)

### 2.1 Docker Socket Proxy toevoegen
Traefik verliest directe `/var/run/docker.sock` toegang. Een `tecnativa/docker-socket-proxy` geeft alleen read-only container-info door.

**Toevoegen aan `/opt/traefik/docker-compose.yml`:**
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
    # ... bestaande config aanpassen:
    # VERWIJDER: /var/run/docker.sock volume mount
    # TOEVOEG: --providers.docker.endpoint=tcp://socket-proxy:2375
    networks:
      - proxy
      - socket-proxy

networks:
  proxy:
    external: true
  socket-proxy:
    driver: bridge
    internal: true   # socket-proxy kan internet niet bereiken
```

### 2.2 Traefik Dynamic Config
**Nieuw bestand:** `/opt/traefik/dynamic/middlewares.yml`

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
        regex: "^https://www\\.athide\\.nl/(.*)"
        replacement: "https://athide.nl/${1}"
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
  - "traefik.http.routers.traefik.rule=Host(`traefik.athide.nl`)"
  - "traefik.http.routers.traefik.middlewares=management-whitelist@file,security-headers@file"
  - "traefik.http.routers.traefik.service=api@internal"
```
> **Basic Auth verwijderd.** Dashboard alleen bereikbaar via Tailscale.

### 2.4 Traefik Static Config Update
**Bestand:** `/opt/traefik/traefik.yml`
- Toevoegen: `file` provider voor `/opt/traefik/dynamic/`
- Docker endpoint wijzigen naar `tcp://socket-proxy:2375`

---

## Fase 3: Keycloak Stack (`/opt/keycloak/`)

**Eigen docker-compose.yml met:**

| Container | Image | Netwerken | Memory Limit |
|-----------|-------|-----------|-------------|
| `keycloak` | `quay.io/keycloak/keycloak:26.0` | `proxy`, `net-auth`, `kc-internal` | 1024MB |
| `postgres-keycloak` | `postgres:16-alpine` | `kc-internal` alleen | 512MB (max_connections=50) |
| `oauth2-proxy` | `quay.io/oauth2-proxy/oauth2-proxy:v7.6.0` | `proxy`, `net-auth` | 64MB |

**Interne netwerken (stack-eigen):**
- `kc-internal` (bridge, internal): Keycloak ↔ PostgreSQL

**Externe netwerken (gedeeld):**
- `proxy`: Traefik routing
- `net-auth`: OIDC voor Nextcloud

**Keycloak path-based routing (publiek vs admin):**
```yaml
labels:
  # Publiek: OIDC flows voor Nextcloud en andere clients
  - "traefik.http.routers.keycloak-public.rule=Host(`auth.athide.nl`) && (PathPrefix(`/realms`) || PathPrefix(`/resources`) || PathPrefix(`/js`))"
  - "traefik.http.routers.keycloak-public.middlewares=security-headers@file,rate-limit@file"
  # Admin console: ALLEEN via Tailscale
  - "traefik.http.routers.keycloak-admin.rule=Host(`auth.athide.nl`) && PathPrefix(`/admin`)"
  - "traefik.http.routers.keycloak-admin.middlewares=management-whitelist@file,security-headers@file"
```

**Hardening per container:**
- `keycloak`: read_only, no-new-privileges, cap_drop ALL, tmpfs /tmp, user 1000
- `postgres-keycloak`: read_only, no-new-privileges, cap_drop ALL, tmpfs /tmp + /run/postgresql
- `oauth2-proxy`: read_only, no-new-privileges, cap_drop ALL, user 65534

**oauth2-proxy configuratie (in deze stack):**
```yaml
oauth2-proxy:
  image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
  environment:
    OAUTH2_PROXY_PROVIDER: "keycloak-oidc"
    OAUTH2_PROXY_OIDC_ISSUER_URL: "https://auth.athide.nl/realms/athide"
    OAUTH2_PROXY_CLIENT_ID: "oauth2-proxy"
    OAUTH2_PROXY_CLIENT_SECRET_FILE: "/run/secrets/oauth2_client_secret"
    OAUTH2_PROXY_COOKIE_SECRET_FILE: "/run/secrets/oauth2_cookie_secret"
    OAUTH2_PROXY_COOKIE_DOMAINS: ".athide.nl"  # SSO over alle subdomeinen
    OAUTH2_PROXY_COOKIE_SECURE: "true"
    OAUTH2_PROXY_COOKIE_HTTPONLY: "true"
    OAUTH2_PROXY_COOKIE_SAMESITE: "lax"
    OAUTH2_PROXY_REVERSE_PROXY: "true"
    OAUTH2_PROXY_EMAIL_DOMAINS: "*"            # Filtering via Keycloak rollen
    OAUTH2_PROXY_WHITELIST_DOMAINS: ".athide.nl"
    OAUTH2_PROXY_SET_XAUTHREQUEST: "true"
    OAUTH2_PROXY_HTTP_ADDRESS: "0.0.0.0:4180"
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.oauth2.rule=Host(`auth.athide.nl`) && PathPrefix(`/oauth2`)"
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

**Na deployment:**
1. Realm `athide` aanmaken
2. MFA (TOTP) verplicht instellen
3. OIDC clients aanmaken:
   - `oauth2-proxy` (confidential, redirect URI: `https://auth.athide.nl/oauth2/callback`)
   - `nextcloud` (voor directe OIDC integratie)
4. **RBAC:** Rol `admin_access` aanmaken in Keycloak. Alleen gebruikers met deze rol krijgen toegang tot admin panels via oauth2-proxy (`OAUTH2_PROXY_ALLOWED_ROLES=admin_access`). Dit voorkomt dat reguliere Nextcloud-gebruikers bij Ghost admin of Traefik dashboard kunnen.
5. Brute force detection aan
6. Password policy: min 12 tekens, hoofdletter, cijfer, speciaal

---

## Fase 4: Ghost Stack (`/opt/ghost/`)

**Eigen docker-compose.yml met:**

| Container | Image | Netwerken | Memory Limit |
|-----------|-------|-----------|-------------|
| `ghost` | `ghost:5-alpine` | `proxy`, `ghost-internal` | 384MB |
| `mariadb-ghost` | `mariadb:11-jammy` | `ghost-internal` alleen | 384MB |

**Interne netwerken (stack-eigen):**
- `ghost-internal` (bridge, internal): Ghost ↔ MariaDB

**Externe netwerken (gedeeld):**
- `proxy`: Traefik routing

**Ghost heeft GEEN toegang tot `net-auth`** — Ghost admin wordt beschermd via Traefik forward-auth (oauth2-proxy), niet via directe OIDC integratie in Ghost zelf.

> **Belangrijk (2nd opinion):** Ghost ondersteunt GEEN native Header Authentication of OIDC. Forward-auth via oauth2-proxy is een **extra slot op de deur** (toegangscontrole vóór de Ghost login pagina), maar het is GEEN True SSO. Binnen Ghost is nog steeds een sterk wachtwoord + Ghost-eigen 2FA nodig. De forward-auth voorkomt dat onbevoegden überhaupt de login pagina bereiken.

**Traefik routing (path-based):**
```yaml
labels:
  # Publieke website
  - "traefik.http.routers.ghost.rule=Host(`athide.nl`)"
  - "traefik.http.routers.ghost.middlewares=security-headers@file,rate-limit@file"
  # Admin panel: Tailscale whitelist + Keycloak forward-auth
  - "traefik.http.routers.ghost-admin.rule=Host(`athide.nl`) && PathPrefix(`/ghost`)"
  - "traefik.http.routers.ghost-admin.middlewares=management-whitelist@file,auth-keycloak@file,security-headers@file"
```

**Subdomein:** `athide.nl` + `www.athide.nl` (redirect)

**Hardening:**
- `ghost`: no-new-privileges, cap_drop ALL, tmpfs /tmp
- `mariadb-ghost`: read_only, no-new-privileges, cap_drop ALL, tuned buffer pool

---

## Fase 5: Nextcloud Stack (`/opt/nextcloud/`)

**Eigen docker-compose.yml met:**

| Container | Image | Netwerken | Memory Limit |
|-----------|-------|-----------|-------------|
| `nextcloud` | `nextcloud:30-apache` | `proxy`, `net-auth`, `nc-internal`, `nc-cache` | 1024MB |
| `postgres-nextcloud` | `postgres:16-alpine` | `nc-internal` alleen | 512MB (max_connections=50) |
| `redis` | `redis:7-alpine` | `nc-cache` alleen | 128MB |

**Interne netwerken (stack-eigen):**
- `nc-internal` (bridge, internal): Nextcloud ↔ PostgreSQL
- `nc-cache` (bridge, internal): Nextcloud ↔ Redis

**Externe netwerken (gedeeld):**
- `proxy`: Traefik routing
- `net-auth`: OIDC met Keycloak

**Subdomein:** `cloud.athide.nl`

**Na deployment:**
1. OIDC integratie met Keycloak (`user_oidc` app)
2. MFA via Keycloak (niet Nextcloud-eigen TOTP)
3. Background jobs via system cron
4. **Preview generation beperken** in `config.php` (voorkom CPU 100% bij grote foto-uploads):
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

## Fase 6: Mailcow Stack (`/opt/mailcow-dockerized/`)

### 6.1 Installatie
```bash
cd /opt
git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
./generate_config.sh  # hostname: mail.athide.nl
```

### 6.2 Traefik Integratie
**Aanpassingen aan `mailcow.conf`:**
```
HTTP_PORT=8080
HTTP_BIND=127.0.0.1
HTTPS_PORT=8443
HTTPS_BIND=127.0.0.1
SKIP_LETS_ENCRYPT=y
```

Mailcow-nginx krijgt Traefik labels + verbinding met extern `proxy` netwerk.

> **KRITIEK (2nd opinion): Proxy-on-Proxy probleem.** Mailcow gebruikt intern Nginx als proxy. Traefik staat daar weer voor. Zonder correcte `X-Forwarded-For` / `proxy_protocol` configuratie ziet Rspamd alle mail als afkomstig van Traefik's IP → antispam is dan volledig blind. **Oplossing:** In `mailcow.conf` correct `TRUSTED_NETWORK` instellen op het Docker proxy netwerk subnet, en in Traefik's entrypoint `proxyProtocol` + `forwardedHeaders` configureren voor het Mailcow-nginx backend.

**Mailcow admin UI afschermen:**
```yaml
# In Mailcow's nginx Traefik labels:
labels:
  # Webmail (SOGo) - publiek
  - "traefik.http.routers.mailcow.rule=Host(`mail.athide.nl`) && !PathPrefix(`/admin`)"
  - "traefik.http.routers.mailcow.middlewares=security-headers@file,rate-limit@file"
  # Admin UI - alleen Tailscale
  - "traefik.http.routers.mailcow-admin.rule=Host(`mail.athide.nl`) && PathPrefix(`/admin`)"
  - "traefik.http.routers.mailcow-admin.middlewares=management-whitelist@file,security-headers@file"
```

**Mail-poorten direct op host** (niet via Traefik):
25, 587, 993, 995, 4190

### 6.3 Mailcow Containers (eigen beheer, eigen netwerk)
Postfix, Dovecot, Rspamd, ClamAV, SOGo, MariaDB, Redis, Nginx — allemaal in Mailcow's eigen interne netwerk. **Geen enkele Mailcow container (behalve nginx) raakt het `proxy` netwerk.**

### 6.4 Memory Budget Mailcow: ~2.5GB totaal

### 6.5 DNS Records (bij TransIP)

| Type | Naam | Waarde |
|------|------|--------|
| MX | `athide.nl` | `mail.athide.nl` (prio 10) |
| A | `mail` | `89.167.107.143` |
| TXT | `athide.nl` | `v=spf1 mx a ip4:89.167.107.143 -all` |
| TXT | `dkim._domainkey` | *(gegenereerd door Mailcow)* |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:dmarc@athide.nl` |
| TXT | `_smtp._tls` | `v=TLSRPTv1; rua=mailto:tls@athide.nl` |
| SRV | `_autodiscover._tcp` | `0 0 443 mail.athide.nl` |
| CNAME | `autoconfig` | `mail.athide.nl` |

**DMARC progressie:** `p=none` → `p=quarantine` (week 3) → `p=reject` (maand 3)

> **Hetzner blokkeert poort 25 standaard.** Support ticket indienen als eerste stap (1-3 werkdagen).

---

## Fase 7: Hardening & Monitoring

### 7.1 Container Hardening Baseline (alle stacks)
```yaml
security_opt:
  - no-new-privileges:true
  - seccomp=default                # Docker default seccomp profiel (blokkeert ~44 syscalls)
  - apparmor=docker-default        # AppArmor standaard profiel
cap_drop:
  - ALL
pids_limit: 200
restart: on-failure:5              # Voorkom oneindige restart loops (niet unless-stopped)
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
| traefik | socket-proxy | YES | (geen) | 65534 | default | docker-default |
| keycloak | keycloak | YES | (geen) | 1000 | default | docker-default |
| keycloak | postgres-keycloak | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| keycloak | oauth2-proxy | YES | (geen) | 65534 | default | docker-default |
| nextcloud | nextcloud | **NEE** (\*) | NET_BIND_SERVICE, DAC_OVERRIDE, SETGID, SETUID, CHOWN, FOWNER | www-data | default | docker-default |
| nextcloud | postgres-nextcloud | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| nextcloud | redis | YES | (geen) | 999 | default | docker-default |
| ghost | ghost | **NEE** (\*) | NET_BIND_SERVICE, CHOWN, SETGID, SETUID | 1000 | default | docker-default |
| ghost | mariadb-ghost | YES | DAC_OVERRIDE, SETGID, SETUID, FOWNER, CHOWN | 999 | default | docker-default |
| mailcow | *(eigen beheer)* | | | | | |

(\*) Writable volumes nodig voor uploads/content. Volumes gemount met minimale paden — **nooit** `/etc`, `/var/lib/docker`, of andere host-systeem directories mounten.

### 7.3 Docker Content Trust & Image Verificatie
```bash
# Activeer Docker Content Trust (alleen getekende images pullen)
echo 'export DOCKER_CONTENT_TRUST=1' >> /etc/environment
```
- Alle images worden gepind op specifieke versies (geen `latest` tags)
- Wekelijks `docker pull` + herstart om security patches te ontvangen

### 7.4 Vulnerability Scanning met Trivy
```bash
# Installeer Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan alle draaiende images
docker images --format '{{.Repository}}:{{.Tag}}' | xargs -I {} trivy image --severity HIGH,CRITICAL {}
```
- **Wekelijks geautomatiseerd** via cron
- **Bij elke deployment** voor nieuwe images
- Alerts bij HIGH/CRITICAL findings

### 7.5 Runtime Anomaly Detection met Falco
```yaml
# In /opt/monitoring/docker-compose.yml
falco:
  image: falcosecurity/falco:latest
  privileged: true    # Nodig voor kernel syscall monitoring
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
- Detecteert: shell spawning in containers, gevoelige file reads, onverwachte netwerk connecties, crypto mining
- Alerts via syslog of webhook

### 7.6 CIS Docker Benchmark Audit
```bash
# Docker Bench for Security (regelmatig draaien)
docker run --rm --net host --pid host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc:/etc:ro \
  docker/docker-bench-security
```
- Draaien na elke deployment fase
- Alle findings documenteren en mitigeren
- Target: geen WARN of FAIL op kritieke controles

### 7.7 Bewuste Afwijkingen (gedocumenteerd)

| Best Practice | Status | Reden |
|---------------|--------|-------|
| Rootless Docker | **NIET TOEGEPAST** | Mailcow vereist root Docker daemon; compensatie: per-container user, cap_drop, no-new-privileges |
| `userns-remap` | **NIET TOEGEPAST** | Mailcow incompatibel; compensatie: AppArmor + Seccomp + non-root users |
| Docker Swarm Secrets | **NIET TOEGEPAST** | Geen Swarm; file-based secrets met chmod 600 als alternatief |
| `unless-stopped` restart | **BEWUST VERMEDEN** | `on-failure:5` voorkomt oneindige restart loops bij misconfiguratie |

### 7.8 Backup Strategie (3-2-1 Regel)

**Principe:** 3 kopieën, 2 media, 1 offsite. Encryptie met **asymmetrische GPG** — public key op server, private key ALLEEN op lokale machine. Zelfs bij volledige server-compromise zijn backups niet te ontsleutelen.

**Voorbereiding:**
1. GPG keypair genereren op **lokale machine** (niet de server): `gpg --full-generate-key`
2. Public key exporteren en naar server uploaden: `gpg --export -a "Ralph" > public.key`
3. Op server importeren: `gpg --import public.key`
4. Secrets worden UITGESLOTEN van backups (secrets horen in een password manager, niet in backups)

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

# 1. Database dumps (hot, geen downtime)
docker exec postgres-keycloak pg_dump -U keycloak > "$BACKUP_DIR/keycloak_db_$DATE.sql"
docker exec postgres-nextcloud pg_dump -U nextcloud > "$BACKUP_DIR/nextcloud_db_$DATE.sql"
docker exec mariadb-ghost mariadb-dump --all-databases -u root \
  -p"$(docker exec mariadb-ghost cat /run/secrets/db_root_pass)" > "$BACKUP_DIR/ghost_db_$DATE.sql"

# 2. Config & data (ZONDER secrets)
tar -czf "$BACKUP_DIR/configs_$DATE.tar.gz" \
  /opt/traefik /opt/keycloak /opt/nextcloud /opt/ghost \
  --exclude="*/secrets"
tar -czf "$BACKUP_DIR/nextcloud_data_$DATE.tar.gz" \
  /var/lib/docker/volumes/nextcloud_nextcloud-data/_data

# 3. Mailcow eigen backup
export MAILCOW_BACKUP_LOCATION="$BACKUP_DIR/mailcow"
mkdir -p "$MAILCOW_BACKUP_LOCATION"
cd /opt/mailcow-dockerized && ./helper-scripts/backup_and_restore.sh backup all --silent

# 4. GPG encryptie (asymmetrisch)
for file in "$BACKUP_DIR"/*.{sql,gz}; do
  [ -f "$file" ] || continue
  gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$file"
  shred -u "$file"   # Secure delete onversleutelde versie
done

# 5. Offsite sync via rclone (Hetzner Storage Box, SFTP)
rclone sync "$BACKUP_DIR" storagebox:backups/athide/ -v --log-file="$LOG_FILE"

# 6. Opschonen
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete
log "--- Backup Voltooid ---"
```

**Cron:** `0 2 * * * /opt/scripts/backup.sh`

**Offsite:** Hetzner Storage Box (BX11, ~€3.81/mnd) via rclone (SFTP, poort 23).

**Extra vangnet:** Hetzner Cloud Snapshots inschakelen voor de hele VPS — instant rollback bij catastrofale fouten.

**Retentie:** 30 dagen lokaal, 90 dagen offsite.
**Restore-test:** Wekelijks Ghost DB herstellen in tijdelijke Docker omgeving.

**GPG Private Key beheer:** De private key is het meest kritieke bezit. Als deze verloren gaat, zijn ALLE offsite backups onbruikbaar. Bewaar op minimaal 2 fysiek gescheiden locaties (bijv. 2 USB-sticks in verschillende kluizen, of YubiKey + papieren backup).

### 7.9 Monitoring Stack
- **Node-exporter** (64MB) — systeem-metrics (CPU, RAM, disk, netwerk)
- **Falco** (256MB) — runtime anomaly detection, syscall monitoring
- **Trivy** — wekelijkse vulnerability scans (geen permanente container)
- **Healthcheck cron** (elke 5 min) — service availability alerts
- **Externe uptime monitoring** (Hetrixtools of Uptime Kuma) — extern perspectief

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
| mailcow | *totaal* | ~3000MB | ~2500MB |
| monitoring | node-exporter | 64MB | 32MB |
| monitoring | Falco | 256MB | 128MB |
| vpn | Headscale | 128MB | 64MB |
| **TOTAAL** | | **~7,768MB** | **~6,174MB** |
| **Beschikbaar** | | **~15,500MB** | |
| **Headroom** | | **~7,732MB (50%)** | |

Swap: 2GB safety net, `vm.swappiness=5`.

---

## Subdomein Overzicht

| URL | Stack | Service |
|-----|-------|---------|
| `athide.nl` | ghost | Ghost (website) |
| `www.athide.nl` | ghost | Redirect → athide.nl |
| `auth.athide.nl` | keycloak | Keycloak (SSO/MFA) |
| `cloud.athide.nl` | nextcloud | Nextcloud |
| `mail.athide.nl` | mailcow | Mailcow (webmail/admin) |
| `traefik.athide.nl` | traefik | Dashboard (Tailscale only) |
| `vpn.athide.nl` | headscale | Headscale control server |

---

## Deployment Volgorde

```
Fase 0  → Server upgrade cax21 → cax31 + LUKS encryptie + ARM64 image test
Fase 1  → Infra (netwerken, directories, secrets, daemon.json, UFW, Headscale, host hardening)
Fase 2  → Stack: traefik uitbreiden (socket-proxy, middlewares, Tailscale whitelist)
Fase 3  → Stack: keycloak + oauth2-proxy deployen, realm + RBAC configureren
Fase 4  → Stack: ghost deployen (forward-auth op /ghost)
Fase 5  → Stack: nextcloud deployen + OIDC koppeling + preview config
Fase 6  → Stack: mailcow deployen + Traefik integratie + proxy_protocol config
Fase 7  → DNS records (MX, SPF, DKIM, DMARC) bij TransIP
Fase 8  → Monitoring: Falco + node-exporter + Trivy scan
Fase 9  → Backup setup (GPG asymmetrisch + rclone + Hetzner Snapshots)
Fase 10 → CIS Docker Bench audit + hardening review
Fase 11 → Validatie (SSL Labs, nmap, Incognito tests, cross-stack tests, SMTP egress)
```

---

## Validatie Checklist

### TLS & Headers
- [ ] SSL Labs A+ rating op alle subdomeinen
- [ ] securityheaders.com A+ rating

### Mail
- [ ] mail-tester.com score ≥ 9/10
- [ ] MXToolbox: SPF, DKIM, DMARC pass

### Container Hardening
- [ ] Alle containers: `no-new-privileges`, `cap_drop: ALL`
- [ ] Alle containers: Seccomp profiel actief (default)
- [ ] Alle containers: AppArmor profiel actief (docker-default)
- [ ] Alle containers: `restart: on-failure:5` (geen `unless-stopped`)
- [ ] Alle containers: specifieke `user:` (non-root) gedefinieerd
- [ ] Alle containers: memory + pids limits ingesteld
- [ ] Geen container draait met `--privileged` (behalve Falco monitoring)
- [ ] Geen `latest` tags — alle images gepind op specifieke versie

### Netwerk Isolatie (Zero Trust)
- [ ] Geen container heeft directe Docker socket toegang (socket-proxy only)
- [ ] Databases niet bereikbaar vanaf proxy netwerk
- [ ] **Cross-stack test:** vanuit Ghost container, `curl` naar Keycloak/Nextcloud FAALT
- [ ] **Cross-stack test:** vanuit Nextcloud, `curl` naar Ghost MariaDB FAALT
- [ ] **Cross-stack test:** vanuit Keycloak, `curl` naar Nextcloud PostgreSQL FAALT
- [ ] Alle DB-netwerken zijn `--internal` (geen internet toegang)

### Secrets & Data
- [ ] Secrets per stack geïsoleerd (Ghost kan Keycloak secrets niet lezen)
- [ ] Geen wachtwoorden in docker-compose.yml of .env (alleen file references)
- [ ] Docker Content Trust actief (`DOCKER_CONTENT_TRUST=1`)
- [ ] Geen gevoelige host directories gemount (`/etc`, `/var/lib/docker`)

### SMTP Egress Filtering
- [ ] **SMTP sinkhole test:** Vanuit Ghost container, `curl` naar poort 25 op internet FAALT — alleen Mailcow mag SMTP praten
- [ ] **SMTP sinkhole test:** Vanuit Nextcloud container, outbound poort 25 FAALT
- [ ] **SMTP sinkhole test:** Vanuit Keycloak container, outbound poort 25 FAALT

### Admin Isolatie (Tailscale)
- [ ] **Incognito test (VPN UIT):** `https://auth.athide.nl/admin` → 403 Forbidden
- [ ] **Incognito test (VPN UIT):** `https://mail.athide.nl/admin` → 403 Forbidden
- [ ] **Incognito test (VPN UIT):** `https://traefik.athide.nl` → 403 Forbidden
- [ ] **Incognito test (VPN AAN):** `https://athide.nl/ghost` → Keycloak login → MFA → Ghost admin
- [ ] oauth2-proxy RBAC: gebruiker ZONDER `admin_access` rol → geweigerd bij Ghost admin

### Scanning & Audit
- [ ] Trivy scan: geen CRITICAL vulnerabilities in draaiende images
- [ ] Docker Bench for Security: geen FAIL op kritieke controles
- [ ] Falco draait en detecteert test-anomalie (shell spawn in container)
- [ ] **Nmap scan** (vanaf externe machine, niet op Tailscale): alleen poorten 22, 25, 80, 443, 465, 587, 993, 4190 open. Alle andere (8080, 5432, 3306, etc.) CLOSED/FILTERED
- [ ] Geen enkele docker-compose.yml bevat `ports:` voor database containers

### Backup & Recovery
- [ ] Backup GPG encryptie: backup bestand is niet leesbaar zonder private key
- [ ] Backup EXCLUDEERT `/opt/*/secrets/` directories
- [ ] Restore test: Ghost DB herstellen in tijdelijke Docker omgeving succesvol
- [ ] Offsite backup naar Hetzner Storage Box werkend via rclone

### Host Hardening
- [ ] Fail2ban actief op SSH
- [ ] UFW: alleen poorten 22, 80, 443, 25, 465, 587, 993, 4190, 41641/udp open
- [ ] UFW: Tailscale interface (`tailscale0`) trusted
- [ ] SSH: geen root login, geen password auth
- [ ] LUKS encryptie actief op data volume (`/opt`)

---

## Risico's en Mitigatie

| Risico | Ernst | Mitigatie |
|--------|-------|-----------|
| Traefik Docker socket toegang | MEDIUM | Socket-proxy (read-only, alleen containers endpoint) |
| `proxy` netwerk als laterale route | MEDIUM | Containers exposeren alleen HTTP poort; Seccomp + AppArmor beperken syscalls; geen shell/exec |
| `net-auth` als laterale route | LOW | Alleen OIDC protocol; Keycloak en Nextcloud enige deelnemers |
| Single server = SPOF | HIGH | Offsite encrypted backups, gedocumenteerde restore, wekelijkse restore-test |
| Hetzner poort 25 geblokkeerd | MEDIUM | Support ticket als eerste actie (fase 6) |
| ClamAV hoog geheugengebruik | LOW | cax31 heeft 8GB headroom |
| Mailcow eigen container beheer | LOW | Battle-tested, eigen update/netwerk; Falco monitort runtime gedrag |
| Supply chain: compromised images | MEDIUM | Docker Content Trust, pinned versies, Trivy scanning, geen `latest` tags |
| Container escape via kernel exploit | LOW | AppArmor + Seccomp + no-new-privileges + non-root users; regelmatig kernel updates |
| Falco draait privileged | LOW | Nodig voor syscall monitoring; compensatie: Falco is read-only, geen netwerk connecties |
| Rootless Docker niet mogelijk | MEDIUM | Mailcow vereist root daemon; compensatie: per-container hardening op alle andere lagen |

### Bronnen Security Best Practices
- [OneUptime: Docker Security Best Practices (2026)](https://oneuptime.com/blog/post/2026-02-02-docker-security-best-practices/view)
- [BetterStack: Docker Security Best Practices](https://betterstack.com/community/guides/scaling-docker/docker-security-best-practices/)
- [Cloud Native Now: Docker Security in 2025](https://cloudnativenow.com/editorial-calendar/best-of-2025/docker-security-in-2025-best-practices-to-protect-your-containers-from-cyberthreats-2/)
