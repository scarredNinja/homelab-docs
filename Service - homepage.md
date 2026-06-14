---
type: swarm-service
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - DockerSwarm
  - Service
  - Dashboard
service_name: Homepage
vm: worker-monitoring-01
swarm_constraint: node.labels.zone == monitoring
vlan: 60
service_status: running
stack_file: proxmox-swarm/stacks/stack-homepage.yml
comment: stack_file is repo-relative (Portainer Git deployment)
port: 3000
external_access: false
traefik_entrypoint: websecure
url_internal: 'https://dashboard.home.purvishome.com'
zfs_dataset: rpool/docker-data/homepage
mount_path: /mnt/docker-data/homepage
last_updated: '2026-06-08T00:00:00.000Z'
status: Completed
---


# Homepage

Self-hosted application dashboard (gethomepage.dev). Pinned to `worker-monitoring-01`. Provides a unified view of all homelab services with status widgets.

## Deployment

-  Deployed to Swarm cluster and running stably on `worker-monitoring-01` (zone=monitoring).
- Wildcard TLS Traefik routing confirmed live at `https://dashboard.home.purvishome.com` (VLAN 60).
- **2026-06-08 Update:** Integrated physical Proxmox VE host resources monitoring, primary/backup Synology NAS status (`diskstation` widgets with Swarm secrets), and Prometheus-based RX/TX bandwidth rate metrics for `vmbr1` and `enp5s0` in a new `Hardware` section at the top of the dashboard. Configured a custom anime-style Japanese bridge background image and applied modern glassmorphism styling via `custom.css`. Deployed on feature branch `feature/homepage-proxmox-integration`.
- **2026-06-02 Update:** Active widgets synchronization completed. Corrected Portainer environment ID to 5, bypassed self-signed TLS globally (`NODE_TLS_REJECT_UNAUTHORIZED: "0"`), configured Uptime Kuma native status widget, and securely integrated Grafana credentials (`grafana_admin_user` and `grafana_admin_password`) via Docker Secrets for the active Grafana widget.
- **2026-05-28 Update:** Migrated Pi-hole widget credentials to Docker Secrets (`pihole_app_password`). Secret mounted at `/run/secrets/pihole_app_password`. Widgets still show API error due to path-resolution constraints. Plan is to substitute via env vars in the next session.
- Stack: `proxmox-swarm/stacks/stack-homepage.yml`
- Config files: `/mnt/docker-data/homepage/` (virtiofs from Proxmox host)
- Networks: `traefik-public`, `monitoring`
- Middleware: `internal-only@file`

## Config Files

| File | Purpose |
|------|---------|
| `settings.yaml` | Theme, layout, general settings |
| `services.yaml` | Service tiles and groupings |
| `widgets.yaml` | Top-level info widgets (date, stats) |
| `bookmarks.yaml` | Quick-access bookmarks |
| `docker.yaml` | Docker integration (optional) |

## Planned Service Layout

| Group | Services |
|-------|---------|
| Monitoring | Grafana, Prometheus, Uptime Kuma |
| Management | Portainer, Proxmox, pfSense, Traefik |
| Media | Plex, Tautulli |
| Downloads | Sonarr, Radarr, Prowlarr, Transmission |
| Home | Home Assistant, UniFi |

---

##  Deployment Config

### Docker Swarm Stack (`stack-homepage.yml`)

```yaml
version: "3.8"

# Homepage dashboard  worker-monitoring-01
#
# Placement: node.labels.zone == monitoring  (worker-monitoring-01 only)
#
# Pre-requisites (run once on manager):
#   docker node update --label-add zone=monitoring <worker-monitoring-01-node-id>
#   docker network create --driver overlay --attachable traefik-public  # if not already present
#
# Provisioning pre-requisites on worker-monitoring-01:
#   - /mnt/docker-data/homepage  (virtiofs share)
#       Contents: settings.yaml, services.yaml, widgets.yaml, bookmarks.yaml,
#                 docker.yaml, kubernetes.yaml, custom.css, custom.js
#       Source files live under proxmox-swarm/stacks/homepage/ in this repo.
#
# Network notes:
#   - traefik-public: external Traefik ingress.
#   - monitoring:     external overlay shared with prometheus/grafana so the
#                     resources widget can reach prometheus directly.
#
# Deploy via Portainer: Stacks -> Add stack -> paste this file -> name "homepage"

services:
  homepage:
    image: ghcr.io/gethomepage/homepage:v1.4.5

    environment:
      # Required since Homepage v0.9+ (host header allow-list).
      HOMEPAGE_ALLOWED_HOSTS: "dashboard.home.purvishome.com"
      HOMEPAGE_FILE_PIHOLE_KEY: /run/secrets/pihole_app_password
      HOMEPAGE_FILE_SONARR_KEY: /run/secrets/sonarr_apikey_v2
      HOMEPAGE_FILE_RADARR_KEY: /run/secrets/radarr_apikey_v2
      HOMEPAGE_FILE_PROWLARR_KEY: /run/secrets/prowlarr_apikey
      HOMEPAGE_FILE_PLEX_KEY: /run/secrets/plex_token
      HOMEPAGE_FILE_TAUTULLI_KEY: /run/secrets/tautulli_apikey
      HOMEPAGE_FILE_SEERR_KEY: /run/secrets/seerr_apikey
      HOMEPAGE_FILE_HA_TOKEN: /run/secrets/ha_prometheus_token
      HOMEPAGE_FILE_UNIFI_USER: /run/secrets/unifi_username
      HOMEPAGE_FILE_UNIFI_PASS: /run/secrets/unifi_password
      HOMEPAGE_FILE_PROXMOX_TOKEN: /run/secrets/proxmox_token
      HOMEPAGE_FILE_PORTAINER_KEY: /run/secrets/portainer_key
      HOMEPAGE_FILE_GRAFANA_USER: /run/secrets/grafana_admin_user
      HOMEPAGE_FILE_GRAFANA_PASS: /run/secrets/grafana_admin_password
      HOMEPAGE_FILE_SYNOLOGY_USER: /run/secrets/synology_username
      HOMEPAGE_FILE_SYNOLOGY_PASS: /run/secrets/synology_password
      NODE_TLS_REJECT_UNAUTHORIZED: "0"
    secrets:
      - source: pihole_app_password
        target: pihole_app_password
      - source: sonarr_apikey_v2
        target: sonarr_apikey_v2
      - source: radarr_apikey_v2
        target: radarr_apikey_v2
      - source: prowlarr_apikey
        target: prowlarr_apikey
      - source: plex_token
        target: plex_token
      - source: tautulli_apikey
        target: tautulli_apikey
      - source: seerr_apikey
        target: seerr_apikey
      - source: ha_prometheus_token
        target: ha_prometheus_token
      - source: unifi_username
        target: unifi_username
      - source: unifi_password
        target: unifi_password
      - source: proxmox_token
        target: proxmox_token
      - source: portainer_key
        target: portainer_key
      - source: grafana_admin_user
        target: grafana_admin_user
      - source: grafana_admin_password
        target: grafana_admin_password
      - source: synology_username
        target: synology_username
      - source: synology_password
        target: synology_password

    volumes:
      - /mnt/docker-data/homepage:/app/config
      - /mnt/docker-data/homepage/images:/app/public/images:ro

    # Override the image's built-in healthcheck  at low CPU limits the
    # Next.js cold start exceeds the default start_period and Swarm
    # kills the task with "unhealthy container" (exit 137) before it
    # finishes booting.
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3000/api/healthcheck >/dev/null 2>&1 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

    networks:
      - traefik-public
      - monitoring

    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.labels.zone == monitoring
      restart_policy:
        condition: on-failure
        delay: 10s
      update_config:
        parallelism: 1
        delay: 10s
        order: stop-first
        failure_action: rollback
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.1"
          memory: 128M
      labels:
        - "traefik.enable=true"
        # Service is attached to multiple overlays; tell Traefik which one
        # to dial when load-balancing.
        - "traefik.swarm.network=traefik-public"
        - "traefik.http.routers.homepage.rule=Host(`dashboard.home.purvishome.com`)"
        - "traefik.http.routers.homepage.entrypoints=websecure"
        - "traefik.http.routers.homepage.tls=true"
        - "traefik.http.routers.homepage.tls.certresolver=cloudflare"
        - "traefik.http.routers.homepage.middlewares=internal-only@file"
        - "traefik.http.services.homepage.loadbalancer.server.port=3000"

networks:
  traefik-public:
    external: true
  # monitoring is a pre-created external overlay (not owned by any stack).
  # The resources widget uses this to reach prometheus directly.
  monitoring:
    external: true

secrets:
  pihole_app_password:
    external: true
  sonarr_apikey_v2:
    external: true
  radarr_apikey_v2:
    external: true
  prowlarr_apikey:
    external: true
  plex_token:
    external: true
  tautulli_apikey:
    external: true
  seerr_apikey:
    external: true
  ha_prometheus_token:
    external: true
  unifi_username:
    external: true
  unifi_password:
    external: true
  proxmox_token:
    external: true
  portainer_key:
    external: true
  grafana_admin_user:
    external: true
  grafana_admin_password:
    external: true
  synology_username:
    external: true
  synology_password:
    external: true
```

## Related

- [[Docker Swarm Infrastructure Runbook]]
- [[VM - monitoring-01]]
- [[Service - uptime-kuma]]
- [[Service - grafana]]
