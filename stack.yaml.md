version: '3.7'

services:
  prometheus1:
    image: prom/prometheus:latest
    hostname: prometheus1
    command: --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --web.enable-lifecycle
    volumes:
      # Static config file
      - /mnt/data/prometheus/prometheus1/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      # Bind Mount 1 for TSDB
      - /mnt/data/prometheus/prometheus1:/prometheus
    deploy:
      labels:
        # Enable Traefik
        - "traefik.enable=true"
        
        # Router Definition: Use a common router name (e.g., 'prom') for both
        - "traefik.http.routers.prom.rule=Host(`prometheus.home.local`)"
        - "traefik.http.routers.prom.entrypoints=web" # Assuming HTTPS on Traefik
        - "traefik.http.routers.prom.service=prom-service@docker" # Specify the service
        
        # Service Definition: Traefik will load balance both containers using this port
        - "traefik.http.services.prom-service.loadbalancer.server.port=9090"
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.hostname == swarm-mgr1
    networks:
      - monitoring
      - traefik-public

  prometheus2:
    image: prom/prometheus:latest
    hostname: prometheus2
    command: --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --web.enable-lifecycle
    volumes:
      - /mnt/data/prometheus/prometheus2/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      # Bind Mount 2 for TSDB
      - /mnt/data/prometheus/prometheus2:/prometheus
    deploy:
      labels:
        # Enable Traefik
        - "traefik.enable=true"
        # Service Definition: Traefik will load balance both containers using this port
        - "traefik.http.services.prom-service.loadbalancer.server.port=9090"
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.hostname == swarm-mgr2
    networks:
      - monitoring
      - traefik-public
      
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=mNUQk5pkvQ5t62 # <-- SET A STRONG PASSWORD!
    volumes:
      # Bind Mount for NFS-backed data
      - /mnt/grafana:/var/lib/grafana 
    deploy:
      labels:
        - "traefik.enable=true"
        
        # Router Definition
        - "traefik.http.routers.grafana.rule=Host(`grafana.home.local`)" # CORRECTED Hostname
        - "traefik.http.routers.grafana.entrypoints=web"
        
        # Service Definition
        - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == manager
    networks:
      - monitoring
      - traefik-public

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    network_mode: host
    deploy:
      mode: global

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    expose:
      - 8080
    deploy:
      mode: global
    networks:
      - monitoring

networks:
  monitoring:
    driver: overlay
    attachable: true
  traefik-public:
    external: true