# OpenWrt LuCI Agent — Firewall Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

## 14. Firewall management

### 14.1 General policy

OpenWrt 25.x uses firewall4/nftables on standard current installations. Do not
assume a legacy firewall3/iptables environment.

Detect the running firewall architecture before using firewall-specific commands.

### 14.2 UCI firewall inventory

Inspect at minimum:

```text
config defaults
config zone
config forwarding
config rule
config redirect
config include
config ipset / set definitions if present
```

Map:

```text
zone -> networks/devices
zone -> input/output/forward policy
forward -> source/destination zones
rule -> protocol/address/port/action
redirect -> NAT/port-forward semantics
```

### 14.3 Safe firewall change

```text
1. identify current management path;
2. identify the matching firewall zone;
3. locate existing equivalent/contradictory rules;
4. add the smallest rule needed;
5. inspect pending UCI changes;
6. apply with rollback protection;
7. verify firewall state and management access.
```

### 14.4 Avoid rule duplication

Before adding a rule, search for an existing rule with equivalent:

```text
source zone
 destination zone
protocol
source/destination address
source/destination port
action
family
```

Prefer modifying an existing rule only when the request clearly concerns it.

### 14.5 Firewall runtime verification

If firewall4 tooling is available and permitted, use read-only inspection such as:

```text
fw4 print
nft list ruleset
```

but prefer structured ubus/UCI sources when they provide the needed information.

Do not interpret a generated ruleset line as a persistent configuration source.

---


## 24. Firewall4 / nftables observation

Use `fw4`/`nft` only for runtime inspection unless the user explicitly asks for
low-level manipulation.

Persistent policy should normally be changed through `/etc/config/firewall` and
then applied by OpenWrt's firewall machinery.

Do not edit generated nftables state as if it were the persistent source of truth.

---
