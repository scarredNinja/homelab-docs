---
project_id: Homelab-2025
phase: 'Phase 1: Preparation'
tags:
  - Storage
status: Reference
---
## Rack Server:

Hpe dl360p gen8

4x 1Gbps nics

2x CPUS (confirm cores)

128 GB RAM
  
📊  Your Current Storage Layout (HPE DL360p Gen8)

|               |        |            |                   |                      |               |
| ------------- | ------ | ---------- | ----------------- | -------------------- | ------------- |
| Logical Drive | RAID   | Size       | Bays Used         | Media Type           | Purpose Today |
| 01            | RAID 0 | 558 GiB    | 1I Box 1 Bay 1    | 600 GB HDD           | Data LUN      |
| 02            | RAID 5 | 1117 GiB   | 1I Box 1 Bays 2–4 | 1.2 TB & 600 GB HDDs | Data LUN      |
| 03            | RAID 0 | 838 GiB    | 2I Box 1 Bay 6    | 900 GB HDD           | Data LUN      |
| 04            | RAID 5 | 999.10 GiB | 2I Box 1 Bays 5-6 | 2x 1TB SSD           | Available     |
|               |        |            |                   |                      |               |
| Free Bays     |        |            | 2I Box 1 Bays 7,8 | -                    | Available     |

## Managed Switch: 

Extreme Switch x440 48p - EXOS 16.2 installed

Rack Server:

3x 600 GB SAS drives

2x 900 GB SAS Drives

OS is in RAID 5 I believe but I need to confirm.

## Synology:

16.2TB
