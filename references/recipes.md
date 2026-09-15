# OpenWrt LuCI Agent — Recipes Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 32. Common operation recipes

### 32.1 Show router status

```text
session.login
-> system board/info
-> network interface dump
-> WAN selection
-> uptime/load/memory/storage
-> summarize
```

### 32.2 Show Wi-Fi status

```text
load wireless
-> getWifiDevices/getWifiNetworks
-> runtime iwinfo/network.wireless if permitted
-> show radios, SSIDs, channels, modes, associated clients
```

### 32.3 Change Wi-Fi SSID

```text
load wireless
-> resolve exact wifi-iface
-> compare current SSID
-> stage ssid only
-> save
-> changes
-> identify management dependency
-> rollback-aware apply
-> flush network cache
-> verify active SSID
```

### 32.4 Create guest network

```text
inspect existing LAN/guest/VLAN topology
-> create/reuse network interface
-> create/reuse Wi-Fi interface
-> create/reuse DHCP service
-> create firewall zone
-> create forwarding/NAT policy
-> save
-> changes
-> rollback-aware apply
-> verify DHCP + isolation + internet
```

### 32.5 Add a port forward

```text
inspect firewall redirects
-> validate destination address stability
-> validate protocol and ports
-> validate WAN zone
-> add minimal redirect
-> add/verify corresponding allow rule if required by platform config
-> apply with rollback
-> verify generated runtime policy
```

### 32.6 Diagnose no internet

```text
client evidence
-> LAN address
-> router gateway
-> firewall forward
-> WAN interface
-> default route
-> upstream gateway
-> raw IP reachability
-> DNS
```

This ordering avoids blaming DNS when the actual failure is link/routing.

### 32.7 Diagnose intermittent Wi-Fi

```text
client association history
-> radio channel
-> signal/noise
-> channel width
-> retries/errors if available
-> hostapd/iwinfo logs
-> driver/kernel logs
-> DHCP timing
```

### 32.8 Diagnose high CPU

```text
load average
-> top/ps
-> CPU frequency/scaling when available
-> firewall/conntrack
-> package/service processes
-> kernel log
-> thermal sensors if available
```

### 32.9 Diagnose storage exhaustion

```text
df -h
-> mount
-> overlay usage
-> package cache
-> logs
-> large writable paths
-> only then cleanup
```

Cleanup must be targeted and reversible where practical.

---
