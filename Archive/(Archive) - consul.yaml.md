---
project_id: Homelab-2025
status: Archived
phase: Archive
tags:
  - archive
---
version: '3.8'

services:
  consul:
    image: hashicorp/consul:latest
    container_name: consul-server
    networks:
      - traefik
    ports:
      - "8500:8500"  # Consul UI
      - "8600:8600"  # DNS
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0 -advertise=10.0.60.30
    volumes:
      - consul-data:/consul/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  webgateway:
    driver: overlay
    external: true
  traefik:
    driver: overlay

volumes:
  consul-data:
      external: true
