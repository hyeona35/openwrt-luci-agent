# OpenWrt LuCI Agent

> An Agent Skill for inspecting, diagnosing, configuring, and safely managing OpenWrt routers through LuCI JavaScript API semantics and authenticated ubus JSON-RPC.

[![OpenWrt](https://img.shields.io/badge/OpenWrt-supported-00b5e2)](https://openwrt.org/)
[![LuCI JS API](https://img.shields.io/badge/LuCI-JavaScript%20API-blue)](https://openwrt.github.io/luci/jsapi/)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-compatible-purple)](https://skills.sh/)

## Overview

`openwrt-luci-agent` is a reusable Agent Skill for AI agents that need to inspect, troubleshoot, configure, and manage OpenWrt routers through the same architectural concepts used by LuCI.

The skill is designed around **ubus JSON-RPC**, **LuCI JavaScript API semantics**, and **UCI transactions** rather than HTML scraping or simulated UI clicks.

It covers both read-only diagnostics and carefully controlled configuration changes, with explicit safety procedures for operations that can disconnect the agent from the router.

## What it can help an agent do

- Inspect router system information and runtime state
- Discover available ubus objects and methods
- Authenticate through LuCI/rpcd sessions and respect ACLs
- Read and modify UCI configuration
- Inspect and configure network interfaces
- Inspect devices, bridges, VLANs, DSA ports, routes, and gateways
- Inspect and configure Wi-Fi radios and `wifi-iface` sections
- Inspect DHCP leases, host information, and DNS-related configuration
- Inspect and manage firewall configuration
- Diagnose WAN, LAN, routing, DHCP, DNS, and connectivity problems
- Inspect services and init/service state where supported by the router
- Read system and kernel logs
- Inspect filesystem metadata and selected files
- Execute tightly controlled diagnostics through the LuCI filesystem/exec APIs
- Manage packages using the router's available package manager
- Perform backup and upgrade workflows with explicit safety checks
- Verify configuration changes after applying them
- Use rollback-aware workflows for high-impact network changes

## Design principles

### No HTML scraping

LuCI is treated as a client of OpenWrt's backend APIs. Agents should communicate with `/ubus` and the associated rpcd/ubus objects instead of scraping LuCI HTML pages or imitating browser form submissions.

### Read before write

Before changing configuration, the agent should inspect the relevant runtime state, UCI configuration, references, and dependencies. Configuration changes should be made only after the intended target is unambiguous.

### Transactional configuration

UCI changes should follow a staged workflow such as:

```text
load → inspect → set/add/delete/order → inspect changes → save → apply → verify
```

For connectivity-sensitive changes, use rollback protection whenever supported and confirm that the new state is actually reachable before considering the operation complete.

### ACL-aware operation

The presence of a ubus object or method does not imply that the current session has permission to call it. Permission errors must be treated as authorization boundaries, not problems to bypass.

### Secret-safe output

Passwords, session tokens, Wi-Fi PSKs, private keys, credential hashes, and backup contents must never be exposed in agent responses or ordinary logs.

## Installation

### Install with the Skills CLI

```bash
npx skills add hyeona35/openwrt-luci-agent
```

Install globally instead of into the current project:

```bash
npx skills add hyeona35/openwrt-luci-agent -g
```

Install only this skill from a multi-skill repository:

```bash
npx skills add hyeona35/openwrt-luci-agent --skill openwrt-luci-agent
```

The Skills CLI supports GitHub repositories, full GitHub URLs, direct skill paths, Git URLs, and local paths. See the Skills CLI documentation for the complete syntax.

## Repository layout

```text
openwrt-luci-agent/
├── SKILL.md
├── README.md
├── LICENSE
├── references/
├── examples/
└── scripts/
```

`SKILL.md` contains the compact agent-facing workflow and safety policy. Detailed API material is split into `references/` and task recipes live under `examples/`, so agents can use progressive disclosure instead of loading the entire OpenWrt manual for every request.

### Reference map

| File | Scope |
|---|---|
| `references/rpc-session.md` | Authentication, ACLs, RPC discovery |
| `references/uci.md` | UCI reads/writes, save/apply, rollback |
| `references/network.md` | Interfaces, devices, WAN/LAN state |
| `references/wireless.md` | Radios, `wifi-iface`, clients, telemetry |
| `references/vlan-bridge-dsa.md` | VLAN/bridge/DSA topology |
| `references/routing.md` | Routes, gateways, routing diagnostics |
| `references/firewall.md` | Firewall UCI and nftables observation |
| `references/dhcp-dns-hosts.md` | DHCP, DNS, host discovery |
| `references/filesystem-exec.md` | Filesystem and controlled command execution |
| `references/system-diagnostics.md` | Logs, health checks, troubleshooting |
| `references/services-packages.md` | Services and package management |
| `references/backup-upgrades.md` | Backups and firmware upgrade safety |
| `references/recipes.md` | End-to-end operational recipes |
| `references/rpc-examples.md` | Raw JSON-RPC building blocks |

### Examples

The `examples/` directory contains focused end-to-end task examples that are useful when evaluating an agent or writing regression tests.

## Requirements

The skill assumes an OpenWrt router with an accessible LuCI/rpcd/ubus control plane. The exact available objects, methods, ACL permissions, package manager, network architecture, and LuCI version vary by OpenWrt release and installed packages.

Agents should discover capabilities at runtime instead of assuming every installation exposes every API described in the documentation.

## Security notes

Do not commit any of the following to this repository:

- router passwords
- `ubus_rpc_session` values
- LuCI authentication cookies
- Wi-Fi passwords or PSKs
- VPN private keys
- API tokens
- private keys
- raw backup archives containing credentials
- logs containing secrets

Use environment variables, a secret manager, or an agent runtime's credential facilities for authentication material.

## Example use cases

### Inspect a router

```text
Inspect my OpenWrt router's WAN, LAN, Wi-Fi, DHCP leases, routes, and system health. Do not change anything.
```

### Diagnose WAN problems

```text
Find out why the router lost WAN connectivity. Check interface state, default routes, DHCP/PPPoE state, DNS, relevant logs, and recent configuration changes without modifying the router.
```

### Safely change Wi-Fi

```text
Change the 2.4 GHz SSID, verify the target radio and wifi-iface section first, apply the change with rollback protection, and verify that the interface comes back correctly.
```

### Create a VLAN

```text
Add a VLAN-backed network and firewall zone. Inspect the existing bridge/DSA/VLAN topology and firewall dependencies first, then make the smallest idempotent configuration change and verify connectivity.
```

## Compatibility

OpenWrt APIs and LuCI abstractions vary across releases. This skill is intentionally designed to be capability-driven:

1. discover supported ubus objects/methods;
2. inspect the current UCI configuration;
3. determine the actual network architecture;
4. perform the smallest safe change;
5. verify the resulting runtime state.

Do not blindly apply examples written for a different OpenWrt release, target, firewall backend, wireless stack, or network topology.

## Documentation

- [OpenWrt](https://openwrt.org/)
- [LuCI JavaScript API](https://openwrt.github.io/luci/jsapi/)
- [OpenWrt ubus](https://openwrt.org/docs/techref/ubus)
- [Agent Skills](https://skills.sh/)

## Contributing

Issues, corrections, compatibility notes, and improvements are welcome. When contributing examples, use fake credentials and redact all secrets.

Please prefer changes that improve portability across OpenWrt releases and make agent behavior safer and more deterministic.

## License

This project is licensed under the Apache License, Version 2.0.

See [LICENSE](./LICENSE) for the full license text.

This project is not affiliated with or endorsed by the OpenWrt Project.


## Validation

For local validation, inspect the structure and run:

```bash
find . -maxdepth 2 -type f | sort

npx skills add . --list
```

When testing against a real router, use a non-production device first. Test read-only discovery before any connectivity-changing workflow.

The skill is deliberately reference-heavy: agents should load only the reference relevant to the current task. This follows the progressive-disclosure pattern described by Vercel's Agent Skills documentation.
