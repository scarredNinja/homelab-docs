---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - traefik
  - VLAN
---
## Tasks
- [x] Setup reverse proxy #Networking  [priority:: 2]
	- [x] Current issue is on accessing the dashboard when separating out between internal and external contexts. need to investigate ✅ 2025-11-08
	- [x] Look at HTTPS redirect and auth #Networking
		- [x] Setup traefik ✅ 2025-11-18
		- [x] Setup pfsense - [[Traefik Implementation - External Services]] ✅ 2026-02-14
		- [x] Setup portainer ✅ 2026-02-14
			- Current error: 404 page not found
		- [x] Setup pihole - [[Traefik Implementation - External Services]]  [priority:: 2] #Monitoring #DockerSwarm
		- [x] Setup proxmox - [[Traefik Implementation - External Services]] ✅ 2026-02-14

## Links
- https://doc.traefik.io/traefik/v1.7/user-guide/examples/
- https://doc.traefik.io/traefik/v1.7/user-guide/swarm-mode/
- https://doc.traefik.io/traefik/v1.7/user-guide/docker-and-lets-encrypt/
- https://doc.traefik.io/traefik/v1.7/user-guide/kv-config/
- https://doc.traefik.io/traefik/v1.7/user-guide/cluster/
- https://doc.traefik.io/traefik/v1.7/user-guide/cluster-docker-consul/
- https://www.youtube.com/watch?v=pU7JvIrthxg
- https://doc.traefik.io/traefik/setup/swarm/

https://www.reddit.com/r/selfhosted/s/riudoZm8BB

~~Setup an internal proxy and a separate external proxy to handle external requests.~~ 
 - Not at the moment, look at one proxy and test from there

## To Note

Look at Jim's garage video or another exaample

Setup a separate reverse prozy for internal and external use and setup a Dmz for exertal.

https://www.reddit.com/r/selfhosted/s/XQpEeSrRY9

Look at how tails ale can fit in etc for external access.

Also for the reverse proxies, see if I couple use my rp without issu

https://www.jeffgeerling.com/blog/2021/my-6-node-1u-raspberry-pi-rack-mount-cluster

---

Once internal is working then look at external but only when internal app services are up

- For external access, needs to authenticate, look at autheliia