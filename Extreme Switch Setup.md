---
project_id: Homelab-2025
phase: "Phase 3: Network Config"
tags:
  - Extreme
  - Networking
---

# 🔌 Extreme Switch Configuration - LACP & VLAN Tagging

**Status:** ✅ Applied on 16/08  
**Device:** Extreme Networks Switch  
**Purpose:** Configure LACP trunks for Proxmox and pfSense with full VLAN tagging

---

## 📋 Overview

This configuration creates two LACP link aggregation groups (LAGs):
1. **Proxmox LAG** - Ports 37, 39, 41, 43 (4-port bond)
2. **pfSense LAG** - Ports 1, 3 (2-port bond)

Both LAGs are configured with:
- **802.3ad LACP** with short timeout
- **All VLANs tagged** (10, 20, 30, 40, 50, 60, 70, 75, 80, 85, 90)
- **No untagged/native VLAN** (VLAN-aware configuration)
- **L3_L4 address-based hashing** for load balancing

---

## 🖥️ Proxmox NICs Configuration

**Physical Ports:** 37, 39, 41, 43  
**LAG Name:** LAG 37  
**Bond Mode:** 802.3ad (LACP)

### Configuration Steps

#### 1. Create the LAG for Proxmox
```bash
create port 37-43 lag 37
enable sharing 37 grouping 37,39,41,43 algorithm address-based L3_L4 lacp
configure sharing 37 lacp timeout short
```

#### 2. Remove from Default VLAN
```bash
unconfigure vlan default delete ports 37-43
```

#### 3. Tag All VLANs to the LAG
```bash
configure vlan 10 add ports 37 tagged
configure vlan 20 add ports 37 tagged
configure vlan 30 add ports 37 tagged
configure vlan 40 add ports 37 tagged
configure vlan 50 add ports 37 tagged
configure vlan 60 add ports 37 tagged
configure vlan 70 add ports 37 tagged
configure vlan 75 add ports 37 tagged
configure vlan 80 add ports 37 tagged
configure vlan 85 add ports 37 tagged
configure vlan 90 add ports 37 tagged
```

#### 4. Save Configuration
```bash
save configuration
```

---

## 🔥 pfSense LAG Configuration

**Physical Ports:** 1, 3  
**LAG Name:** LAG 1 (or pfsense-bond)  
**Bond Mode:** 802.3ad (LACP)

### Configuration Steps

#### 1. Create the LAG for pfSense
```bash
create port 1,3 lag 1
enable sharing 1 grouping 1,3 algorithm address-based L3_L4 lacp
configure sharing 1 lacp timeout short
```

#### 2. Remove from Default VLAN
```bash
unconfigure vlan default delete ports 1,3
```

#### 3. Tag All VLANs to the LAG
```bash
configure vlan 10 add ports 1 tagged
configure vlan 20 add ports 1 tagged
configure vlan 30 add ports 1 tagged
configure vlan 40 add ports 1 tagged
configure vlan 50 add ports 1 tagged
configure vlan 60 add ports 1 tagged
configure vlan 70 add ports 1 tagged
configure vlan 75 add ports 1 tagged
configure vlan 80 add ports 1 tagged
configure vlan 90 add ports 1 tagged
```

#### 4. Save Configuration
```bash
save configuration
```

---

## ✅ Verification Commands

After applying the configuration, verify with these commands:
```bash
# Check LAG status
show sharing

# Verify LACP configuration
show lacp lag 37
show lacp lag 1

# Check VLAN assignments
show vlan

# Verify specific VLAN on LAG
show vlan 10
show ports 37 information detail
show ports 1 information detail
```

---

## 🔍 Key Configuration Details

### LACP Settings
- **Timeout:** Short (1 second for faster failover)
- **Algorithm:** Address-based L3_L4 (source/dest IP and port hashing)
- **Mode:** Active LACP on both sides

### VLAN Configuration
- **All VLANs:** Tagged on both LAGs
- **No Native VLAN:** Ensures all traffic is explicitly tagged
- **VLAN-Aware:** Proxmox and pfSense handle VLAN tagging internally

### Port Assignments

| Device   | Physical Ports | LAG ID | VLANs            |
| -------- | -------------- | ------ | ---------------- |
| Proxmox  | 37, 39, 41, 43 | LAG 37 | 10-90 (all tagged) |
| pfSense  | 1, 3           | LAG 1  | 10-90 (all tagged) |

---

## 🚨 Important Notes

1. **Save Configuration:** EXOS switches require explicit `save configuration` - changes are lost on reboot otherwise
2. **Matching Configuration:** Ensure Proxmox bond0 and pfSense LAGG interfaces are configured for 802.3ad
3. **VLAN Awareness:** Both Proxmox and pfSense must be configured as VLAN-aware to handle tagged traffic
4. **No Untagged Traffic:** All VLANs are tagged; no default/native VLAN is used
5. **Link Verification:** Check that all physical links are up before enabling LACP

---

## 🔗 Related Documentation

- [[Proxmox Network Configuration]]
- [[pfSense LAGG Configuration]]
- [[VLAN Architecture Overview]]
- [[Network Troubleshooting Guide]]

---

## 📝 Change Log

| Date       | Change                                    | Applied By |
| ---------- | ----------------------------------------- | ---------- |
| 2024-08-16 | Initial LACP configuration for Proxmox and pfSense | [Your Name] |

---

## 🛠️ Troubleshooting

### LACP Not Coming Up
```bash
# Check if LACP is enabled
show lacp lag 37

# Verify physical link status
show ports 37,39,41,43

# Check for speed/duplex mismatches
show ports 37,39,41,43 configuration
```

### VLANs Not Passing Traffic
```bash
# Verify VLAN membership
show vlan

# Check specific VLAN on ports
show vlan 10

# Verify tagging
show ports 37 information detail | include VLAN
```

### Performance Issues
```bash
# Check LAG load distribution
show sharing 37 statistics

# Verify hashing algorithm
show sharing 37 configuration
```