---
project_type: Smart Home
project_id: SmartHome-2025
status: Planning
tags:
  - SmartHome
  - HomeAssistant
phase: Smart Home
---

# 🏠 Smart Home Hub

Central hub for the smart home project. Goal: unified control via Home Assistant, with energy monitoring, presence detection, and environmental sensing across the house.

---

## 🎯 Goals

- Centralise all smart home devices under Home Assistant
- Map energy usage per room/circuit via smart plugs/switches
- Extend and modernise Z-Wave network
- Add environmental sensors (temperature, humidity, moisture)
- Door and window sensors for security/automations

---

## 📋 Phase Status

```dataview
TABLE status, priority, due_date AS "Due Date"
FROM "10 - Projects"
WHERE project_id = "SmartHome-2025"
  AND contains(file.name, "Phase")
SORT due_date ASC
```

---

## ✅ Open Tasks

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.project_id && String(p.project_id).includes("SmartHome-2025"));

const tasks = pages
  .flatMap(p => p.file.tasks
    .where(t => !t.completed)
    .map(t => ({ task: t, file: p.file }))
  )
  .array();

if (tasks.length === 0) {
  dv.paragraph("No open tasks yet.");
} else {
  dv.taskList(tasks.map(t => t.task), false);
}
```

---

## 🗂️ Setup Tasks

### Home Assistant

- [ ] Install Home Assistant OS on dedicated VM or hardware [priority:: 1]
- [ ] Configure basic settings — timezone, location, user accounts [priority:: 1]
- [ ] Connect to existing Philips Hue bridge via Hue integration [priority:: 2]
- [ ] Set up Traefik reverse proxy for HA at `https://ha.home.purvishome.com` [priority:: 2]
- [ ] Enable Prometheus metrics integration for Grafana monitoring [priority:: 3]

### Z-Wave Network

- [ ] Investigate Yale Home Module options — replace or extend current Z-Wave module [priority:: 2]
- [ ] Audit current Z-Wave mesh coverage — identify dead zones [priority:: 3]
- [ ] Add Z-Wave repeater nodes if coverage gaps found [priority:: 3] #Later

### Sensors & Devices

- [ ] Research door sensors — Z-Wave or Zigbee, battery vs wired [priority:: 3]
- [ ] Research window sensors [priority:: 3]
- [ ] Research plant moisture sensors — look at Zigbee options [priority:: 4] #Later
- [ ] Research temperature + humidity sensors per room [priority:: 3]
- [ ] Research smart switches/plugs with energy monitoring for per-circuit tracking [priority:: 2]

### Energy Monitoring

- [ ] Define which circuits/rooms to monitor first [priority:: 2]
- [ ] Install smart plugs or inline energy monitors on priority circuits [priority:: 3]
- [ ] Build Grafana dashboard for energy usage over time [priority:: 4] #Later

---

## 🏗️ Architecture Notes

- **Hub:** Home Assistant on `worker-controller-01` (VLAN TBD — likely VLAN 60 or dedicated IoT VLAN)
- **IoT VLAN:** Devices on VLAN 10 — isolated from main network, HA bridges across
- **Integrations planned:**
  - Philips Hue (bridge integration)
  - Z-Wave JS (Yale lock + future Z-Wave devices)
  - Zigbee2MQTT (future Zigbee sensors)
  - Prometheus metrics export to existing Grafana stack

---

## 📝 Notes & Research

- Yale Home Module: check compatibility with Z-Wave JS before replacing hardware
- Look at Sonoff Zigbee bridge as a low-cost Zigbee coordinator option
- Consider whether to run Zigbee and Z-Wave in parallel or consolidate to one protocol

---

## 📎 Links

- [[01 Homelab Rebuild - Phase 6 Service Deployment Hub]] — HA deployment is tracked here
- [[VLAN and Subnet Summary Sheet]] — IoT VLAN 10 details
- [[Traefik Routing Architecture]] — HA internal routing
