# OpenWrt LuCI Agent — System Diagnostics Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 18. System inspection

At the beginning of diagnosis, collect when permitted:

```text
system board
OpenWrt release/version
kernel version
architecture/target
hostname
uptime
memory usage
storage usage
load averages
current time/timezone
```

Common sources include ubus `system` methods and read-only files such as:

```text
/etc/openwrt_release
/etc/os-release
/proc/uptime
/proc/meminfo
/proc/loadavg
```

Do not assume all files exist on every release.

### 18.1 Storage health

For persistent systems inspect:

```text
df -h
mount
block info
logread
```

When supported, identify overlay/upper storage separately from read-only base
firmware storage.

A full overlay can cause apparently unrelated service/configuration failures.

---


## 19. Logs and diagnostics

### 19.1 Prefer targeted reads

Do not blindly dump massive logs.

Use bounded extraction:

```text
logread
logread -e <pattern>
logread -l <count>
```

when available, or read a bounded section of an authorized log file.

### 19.2 Correlation rule

A log message by itself is not enough to establish root cause.

Correlate:

```text
log timestamp
interface/device state
route state
DHCP state
Wi-Fi association
CPU/memory/load
recent configuration changes
```

### 19.3 Common diagnostic workflows

#### WAN outage

```text
system -> WAN candidate -> interface status -> device/carrier -> routes
-> DHCP/PPPoE runtime -> DNS -> ping gateway -> ping IP -> DNS lookup
```

#### Wi-Fi client cannot connect

```text
radio enabled -> wifi-iface state -> SSID/security -> channel -> hostapd/iwinfo
-> client association -> DHCP -> firewall zone
```

#### LAN client has no internet

```text
client lease -> LAN address -> default gateway -> firewall forwarding
-> WAN state -> NAT/firewall -> DNS
```

#### Router is slow

```text
load -> CPU -> memory -> process list -> conntrack/firewall if available
-> storage -> kernel/network errors
```

---
