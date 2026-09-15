# OpenWrt LuCI Agent — Backup Upgrades Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 22. Backup and restore

### 22.1 Before major changes

For substantial network/firewall/VLAN changes, obtain a configuration backup when
permitted.

Possible sources:

```text
sysupgrade configuration backup mechanisms
UCI export
selected /etc/config files
```

Prefer the router's supported backup mechanism when available.

### 22.2 Backup contents

A backup can contain:

- passwords
- SSH keys
- Wi-Fi PSKs
- VPN credentials
- certificates

Therefore treat backup files as secrets.

### 22.3 Restore

Restoring configuration is a high-impact operation.

Before restore:

1. verify target device and release compatibility;
2. preserve a copy of current configuration;
3. inspect which files would be replaced;
4. confirm the management path;
5. apply through a controlled mechanism;
6. verify and be prepared for out-of-band recovery.

Never restore a configuration to another device solely because the model name looks
similar.

---


## 23. Firmware upgrades

Firmware upgrades are HIGH_IMPACT.

### 23.1 Preflight

Collect:

```text
model
board name
target/subtarget
current OpenWrt release
current kernel
architecture
root filesystem/overlay space
available upgrade target
```

### 23.2 Safe strategy

Prefer supported LuCI/Attended Sysupgrade/owut workflows when present.

For a firmware operation:

```text
backup -> validate image identity -> validate architecture -> verify storage
-> schedule/reboot expectation -> perform upgrade -> reconnect -> verify release
```

Never flash an image solely because its filename appears close to the device model.

### 23.3 Special caution

A firmware upgrade is not equivalent to installing packages. Treat them as
separate lifecycle operations.

---
