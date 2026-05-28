
portainer:
  image: portainer/portainer-ce:latest
  command: -H unix:///var/run/docker.sock
  networks:
    - traefik-public
  ports: []  # do NOT expose host ports; Traefik handles routing
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - portainer-data:/data
  deploy:
	mode: replicated
	replicas: 2
	 placement:
    constraints:
      - node.role == swarm-mgr-1
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.lab.local`)"
      - "traefik.http.routers.portainer.entrypoints=web"
      - "traefik.http.routers.portainer.service=portainer"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

volumes:
  portainer-data:

networks:
  traefik-public:
    external: true
