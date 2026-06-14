---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
tags:
  - VLAN
  - Network
  - proxmox
  - DockerSwarm
status: Reference
---


## Phase 1: The "Hardware" Level (Proxmox Host)

Since you have a 4-NIC LACP bond, we need to ensure Proxmox treats the bond as a single high-speed pipe that is "VLAN Aware."

**In Proxmox Web UI:**

1. **Create Bond:** Select all 4 NICs, mode `802.3ad`, Hash Policy `layer2+3`.
    
2. **Create Bridge (`vmbr0`):** Set the bridge ports to `bond0` and check the **VLAN Aware** box.
    
3. **Apply:** No reboot required, but ensure your physical switch is already configured for the LACP LAG.
    

---

## Phase 2: The Manager VM "Virtual Hardware"

When you create your first **Manager VM**, go to the **Hardware** tab and add three network devices:

1. **virtio (net0):** Bridge `vmbr0`, Tag `60` (Infrastructure). ✅
    
2. **virtio (net1):** Bridge `vmbr0`, Tag `80` (DMZ). ✅
    
3. **virtio (net2):** Bridge `vmbr0`, Tag `90` (Management). ✅
    

---

## Phase 3: The OS Level (Inside the VM) - **CHANGE  [[Proxmox Network Configuration]]**

Assuming you are using a modern Linux (Ubuntu/Debian) with **Netplan**, your configuration needs to be very specific about **Gateways**. If you give all three NICs a gateway, the VM will lose its mind.

**File:** `/etc/netplan/00-installer-config.yaml`

YAML

```
network:
  version: 2
  ethernets:
    # net0 - Infrastructure (Primary)
    enp1s0:
      addresses: [10.0.60.10/25]
      nameservers:
        addresses: [10.0.60.1] # Point to pfSense DNS
      routes:
        - to: default
          via: 10.0.60.1
          metric: 100

    # net1 - DMZ (Ingress)
    enp6s1:
      addresses: [10.0.80.10/25]

    # net2 - Management (Admin)
    enp6s2:
      addresses: [10.0.90.10/25]
```

---

## Phase 4: The Docker Swarm Init - ✅

Once the networking is applied (`sudo netplan apply`), initialize your Swarm. You must tell Docker to use the **Infrastructure IP** for cluster management so it stays off the DMZ and Management lines.

Bash

```
docker swarm init --advertise-addr 10.0.60.10
```

---

### Why this works for your first Manager:

- **Safety:** You can SSH into `10.0.90.10` from your Admin machine. Even if you break the Docker network, this link remains.
    
- **Infrastructure:** The VM can talk to your Synology and Prometheus on the `10.0.60.x` network at the full speed of the 4-NIC bond.
    
- **DMZ Readiness:** When you’re ready to deploy Traefik, it’s already got a dedicated "door" (`10.0.80.10`) waiting for external traffic.
    

---

### Notes

- The proxmox setup and bond is already completed.
