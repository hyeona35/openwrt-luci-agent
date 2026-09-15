# OpenWrt LuCI Agent — Wireless Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 11. Wi-Fi management

### 11.1 Distinguish configuration from runtime

For every wireless task, inspect:

```text
wifi-device UCI section
wifi-iface UCI section
radio enabled/disabled state
runtime interface name
SSID
BSSID
channel/frequency
channel width / HT mode
mode (AP/STA/etc.)
encryption
associated network
associated clients
signal/noise when available
```

An SSID in `/etc/config/wireless` is not proof that an AP is currently emitting.

### 11.2 Discover radios

Use:

```text
network.getWifiDevices()
```

or corresponding `uci.sections('wireless', 'wifi-device')` plus runtime status.

### 11.3 Discover Wi-Fi interfaces

Use:

```text
network.getWifiNetworks()
```

and/or per-radio `getWifiNetworks()`.

Resolve the target `wifi-iface` by observed identity, not assumed names such as
`default_radio0`.

### 11.4 Runtime wireless telemetry

When permitted, use `iwinfo` and/or the router's `network.wireless` object.

Because wireless object availability differs across versions and ACLs, first
introspect the object.

A recent OpenWrt installation may expose `network.wireless` methods such as
`status`, but an external HTTP RPC session may not have access to every method by
default. Treat ACL output as authoritative.

### 11.5 Wi-Fi client analysis

When a client is reported, distinguish:

```text
configured device identity
last DHCP lease
current neighbor/ARP state
current Wi-Fi association
current IP address
hostname source
```

Do not equate a stale lease with “currently connected”.

### 11.6 Wi-Fi mutations

Before changing SSID, security, channel or band:

1. identify whether the agent is connected through that radio/SSID;
2. prepare an alternate management path if necessary;
3. stage only the requested options;
4. save and inspect `uci.changes()`;
5. apply with rollback protection;
6. verify runtime AP state;
7. confirm the rollback transaction only after the path is known-good.

Never expose a Wi-Fi PSK in a response.

---

## 11.7 `iwinfo.htmode` compatibility warning

Do not treat `iwinfo.info.htmode` as universally authoritative.

On some MT7981 deployments it has been observed to report `NOHT` even while the configured UCI value is `HE80`. For reliable reporting:

```text
runtime channel/frequency -> iwinfo.info
configured HT/HE width -> uci get wireless.<radio>.htmode
```

Report these as separate facts when they disagree:

- **Configured mode:** UCI value (`HE80`, etc.)
- **Runtime channel:** `iwinfo.info.channel`
- **Runtime htmode:** `iwinfo.info.htmode`, explicitly marked as driver/platform-reported

Do not “correct” UCI configuration merely because `iwinfo` reports `NOHT`.

## 11.8 ACL fallbacks for wireless telemetry

If `network.wireless.get_config`, `hostapd.bss_info`, or other rich wireless methods return access denied:

1. use targeted `uci.get` calls for configured SSID, encryption, network, channel and htmode;
2. use permitted `iwinfo.info` for runtime channel/frequency and basic radio telemetry;
3. use `iwinfo.assoclist` for associated clients when permitted;
4. state explicitly when a value could not be obtained because of ACLs.

Do not interpret `access denied` as evidence that the radio or hostapd is broken.
