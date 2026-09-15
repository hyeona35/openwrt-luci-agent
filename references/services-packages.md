# OpenWrt LuCI Agent — Services Packages Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 20. Service management

Service management is version- and package-dependent.

### 20.1 Discover first

Inspect visible ubus objects such as:

```text
service
rc
```

and use `rpc.list()` before assuming method names.

### 20.2 Common service concepts

An available service API may expose operations such as:

```text
list
get_data
action/init
```

The exact method and argument structure must come from introspection.

### 20.3 Service restart policy

Restarting a service is a write operation because it changes runtime state.

Before restarting:

- identify dependency impact;
- determine whether the service is the management path;
- prefer a targeted reload over a full restart if the service supports it;
- verify service health afterward.

### 20.4 Do not use service restart as a substitute for persistent configuration

If the user asked to change a persistent option, change its UCI source first.

A service restart alone may be lost at reboot.

---


## 21. Package management

### 21.1 Detect package manager

Do not hard-code `opkg`.

OpenWrt 25.12 switched from opkg to apk. Older installations may still use opkg.
Detect the actual installed manager first.

Typical detection:

```text
command -v apk
command -v opkg
```

or package-related ubus/service introspection when available.

### 21.2 Inventory

Read-only operations may include:

```text
apk info
opkg list-installed
```

depending on the detected manager.

### 21.3 Install/remove

Package modification is a write operation and may affect dependencies, flash usage
or core networking.

Before changing packages:

```text
identify firmware version
identify architecture
check free space
check package currently installed
check dependencies
check whether package is part of the base image
```

### 21.4 Upgrade caution

Do not blindly run a global package upgrade on current OpenWrt merely because
updates are available. Full firmware upgrades are the coherent mechanism for
keeping the package set aligned with the release image.

For 25.12-class systems, use attended sysupgrade/owut or another supported
firmware upgrade workflow for coherent system upgrades.

---
