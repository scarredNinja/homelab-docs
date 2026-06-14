---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
version: "3.7"

services:
  prometheus:
    image: prom/prometheus:latest
    user: "0:0"
    ports:
      - "9090:9090"
    volumes:
      - /mnt/prometheus/prometheus_data/etc:/etc/prometheus/
      - /mnt/prometheus/prometheus_data/data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.size=256MB"
      - "--web.console.libraries=/usr/share/prometheus/console_libraries"
      - "--web.console.templates=/usr/share/prometheus/consoles"
    environment:
      - TZ=Pacific/Auckland
    logging:
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      restart_policy:
        condition: on-failure

  grafana:
    image: grafana/grafana:latest
    user: "0:0"
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    volumes:
      - /mnt/grafana/grafana_data/data:/var/lib/grafana
      - /mnt/grafana/grafana_data/provisioning:/etc/grafana/provisioning/
    environment:
      - TZ=Pacific/Auckland
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      restart_policy:
        condition: on-failure

  loki:
    image: grafana/loki:latest
    user: "0:0"
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
    # Optional: persist Loki data
    # volumes:
    #   - /mnt/glustermount/data/loki:/loki

  promtail:
    image: grafana/promtail:latest
    user: "0:0"
    command: -config.file=/etc/promtail/config.yml
    environment:
      - TMPDIR=/var/lib/promtail               # keep temp + positions on same mount
    volumes:
      - /mnt/promtail/promtail_data/promtail-config.yaml:/etc/promtail/config.yml:ro
      - /mnt/promtail/promtail_data:/var/lib/promtail
      - /mnt/promtail/promtail_data/traefik_data/logs/:/traefik/logs/:ro
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1

networks:
  monitoring:
    external: true
