---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Prometheus itself (service name is <stack>_<service>)
  - job_name: prometheus
    static_configs:
      - targets: ['monitoring_prometheus:9090']

  # Traefik metrics endpoint
  - job_name: traefik
    metrics_path: /metrics
    static_configs:
      - targets: ['10.0.90.20:8080']
  
  - job_name: node_exporter
    dns_sd_configs:
      - names: ['tasks.monitoring_node-exporter']
        type: 'A'
        port: 9100

  - job_name: cadvisor
    dns_sd_configs:
      - names: ['tasks.monitoring_cadvisor']
        type: 'A'
        port: 8080
