# Documentation

This directory contains the detailed documentation for the Sovereign Stack project.

## Contents

- [deployment-plan.md](deployment-plan.md) — Full deployment plan with security architecture, network topology, container hardening, and validation checklist.

## Planned Documentation

The following docs will be added as the project is built phase by phase:

- `phase-0-server-setup.md` — Server upgrade, LUKS encryption, ARM64 verification
- `phase-1-infrastructure.md` — Docker daemon, networks, UFW, Headscale, secrets
- `phase-2-traefik.md` — Traefik configuration, socket proxy, middlewares
- `phase-3-keycloak.md` — Keycloak deployment, realm setup, OIDC clients, RBAC
- `phase-4-ghost.md` — Ghost CMS deployment, forward-auth integration
- `phase-5-nextcloud.md` — Nextcloud deployment, OIDC, preview configuration
- `phase-6-mailcow.md` — Mailcow deployment, Traefik integration, DNS records
- `phase-7-monitoring.md` — Falco, Trivy, node-exporter setup
- `phase-8-backup.md` — GPG backup strategy, rclone, Hetzner Storage Box
- `phase-9-validation.md` — Security audit checklist, penetration testing
