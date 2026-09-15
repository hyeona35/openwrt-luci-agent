# OpenWrt LuCI Agent — Dhcp Dns Hosts Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 15. DHCP, DNS and host discovery

### 15.1 Host hints

Use:

```text
network.getHostHints()
```

when available. Host aggregation can combine multiple runtime sources.

### 15.2 DHCP leases

On systems exposing `luci-rpc`, methods may include a DHCP lease helper. Discover
rather than assuming its presence.

If direct lease inspection is needed, identify the DHCP daemon/version and use its
supported runtime or lease file interface only when permitted.

### 15.3 DNS

Distinguish:

```text
router DNS configuration
upstream resolver reachability
local hostname resolution
DHCP-provided DNS information
```

Useful diagnosis commands may include:

```text
nslookup example.com
nslookup example.com <server>
```

Use explicit addresses instead of trusting the router's DNS when determining
whether the DNS service itself is broken.

### 15.4 DHCP mutations

Treat DHCP pool, gateway, DNS and lease-time changes as potentially connectivity-
sensitive. Check for address collisions and static leases before changing ranges.

---


## 16. LuCI.network.Hosts and client identity

Host hints may provide mappings such as:

```text
MAC -> hostname
MAC -> IPv4
MAC -> IPv6
IP -> hostname
```

When reporting a client, include the evidence source conceptually:

```text
Wi-Fi association: yes/no
DHCP lease: present/absent
Neighbor entry: present/absent
Hostname: DHCP/DNS/unknown
```

Avoid overconfident language such as “device is definitely online” unless current
runtime evidence supports it.

---
