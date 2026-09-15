# OpenWrt LuCI Agent — Network Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 9. LuCI.network — network inventory and state

### 9.1 High-level functions

Useful LuCI concepts include:

```text
network.getNetwork(name)
network.getNetworks()
network.getDevice(name)
network.getDevices()
network.getWifiDevice(devname)
network.getWifiDevices()
network.getWifiNetwork(name)
network.getWifiNetworks()
network.getWANNetworks()
network.getWAN6Networks()
network.getHostHints()
network.flushCache()
```

These return high-level objects and aggregated runtime state rather than merely
raw UCI sections.

### 9.2 Logical interfaces

For each logical interface, distinguish:

```text
UCI configuration
runtime up/down state
protocol
L2 device/bridge membership
IPv4 addresses
IPv6 addresses
routes
DNS information
attached physical/wireless devices
```

Do not claim an interface is operational merely because its UCI section exists.

### 9.3 WAN detection

LuCI's WAN helper identifies IPv4 WAN candidates by looking for interfaces with a
default `0.0.0.0/0` route and IPv6 WAN candidates by the IPv6 default route.

Therefore, avoid hard-coding `wan` as the WAN interface when the router's routing
state indicates otherwise.

### 9.4 Devices

A device may be:

- physical Ethernet
- bridge
- VLAN subinterface
- tunnel
- wireless interface
- virtual interface

Inspect device membership rather than inferring topology from names like `eth0` or
`br-lan` alone.

### 9.5 Device state

Useful runtime data includes:

```text
ifname
up/down state
carrier/link state
MAC address
IPv4 addresses
IPv6 addresses
MTU
statistics
associated logical networks
```

Where available, query `network.device.status` rather than parsing `ip` output.

### 9.6 Refresh network cache

After a change that affects runtime state:

```text
network.flushCache()
```

Then re-read high-level objects.

---


## 10. Direct network ubus operations

When the high-level abstraction does not expose enough detail, use low-level
network objects after introspection.

### 10.1 `network.interface`

Common useful methods include:

```text
dump
status
up
down
renew
prepare
add_device
remove_device
notify_proto
set_data
remove
```

The exact available methods and signatures must be taken from `rpc.list()` for the
installed netifd version.

### 10.2 `network.device`

Commonly useful method:

```text
status
```

Use it for runtime L2 state, carrier and statistics when permitted.

### 10.3 Example: diagnose WAN

```text
1. Discover network.interface methods.
2. Call network.interface.dump.
3. Identify default-route candidates.
4. Call status on the selected logical interface if available.
5. Inspect the underlying device.
6. Compare UCI protocol configuration.
7. Inspect DNS and route state.
8. Correlate with recent logs.
```

Do not conclude “ISP is down” solely from a failed DNS query; distinguish DNS,
link, DHCP, routing, and upstream reachability failures.

---
