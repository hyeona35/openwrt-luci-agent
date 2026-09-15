# OpenWrt LuCI Agent — Vlan Bridge Dsa Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 12. VLANs, bridges and switch topology

Treat VLAN/bridge changes as connectivity-sensitive.

### 12.1 Preflight

Inspect:

```text
network config sections
bridge/device sections
bridge-vlan sections if present
CPU/DSA port topology if exposed
current interface-device relations
firewall zone membership
DHCP server bindings
```

### 12.2 DSA awareness

Do not assume legacy `swconfig` semantics on modern DSA targets.

If topology is unclear:

```text
rpc.list() -> inspect available network objects
system board -> identify target
UCI network -> inspect device/bridge-vlan structure
runtime device state -> correlate actual interfaces
```

### 12.3 Safe VLAN workflow

```text
backup relevant UCI
read topology
find all references to target VLAN/device
stage minimal VLAN/bridge change
check dependent interfaces and firewall zones
save
inspect uci.changes
apply with rollback
verify management and tagged/untagged paths
```

Never remove the management VLAN merely because it appears unused in a single
UCI file.

---
