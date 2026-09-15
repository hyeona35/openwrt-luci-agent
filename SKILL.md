---
name: openwrt-luci-agent
description: >-
  Comprehensive skill for AI agents to inspect, diagnose, configure, and manage
  OpenWrt routers through LuCI JavaScript API semantics and authenticated ubus
  JSON-RPC. Supports UCI configuration, network interfaces, devices, bridges,
  VLANs, routing, firewall, Wi-Fi, DHCP/DNS, hosts, services, filesystem and
  logs, package management, backups, upgrades, diagnostics, and rollback-aware
  changes while preserving ACL and credential security.
---

# OpenWrt LuCI Agent

Use this skill whenever an AI agent needs to inspect, diagnose, configure, or manage an OpenWrt router through the LuCI/ubus control plane. The skill is capability-driven: discover what the installed router exposes before assuming an API exists.

## Operating rules

1. **Discover first.** Authenticate, discover ubus capabilities, inspect OpenWrt version/target, and read the relevant UCI/runtime state before writing anything.
2. **Use the browser only as a bootstrap/observation surface.** Browser-side LuCI JavaScript can be useful for reading the rendered UI and obtaining a live LuCI session, but do not depend on browser DOM/form manipulation for configuration writes.
3. **Prefer structured APIs.** Use LuCI/ubus/UCI APIs rather than HTML scraping or simulated UI form submissions.
3. **Read before write.** Determine the exact object, section, dependency graph, and current value before modifying configuration.
4. **Make the smallest idempotent change.** Avoid rewriting unrelated options or creating duplicate firewall, VLAN, Wi-Fi, or forwarding entries.
5. **Protect connectivity.** Treat WAN/LAN/VLAN/bridge/firewall/routing/Wi-Fi changes as potentially connection-breaking. Use rollback-aware apply where supported and verify reachability after applying.
6. **Respect ACLs.** A discoverable ubus method is not automatically callable. Never bypass rpcd ACLs or privilege boundaries. Treat `access denied` as an authorization result, not as an invitation to find a browser or shell bypass.
7. **Keep secrets private.** Never expose passwords, session tokens, Wi-Fi PSKs, private keys, credential hashes, or secret-bearing backups/logs.
8. **Verify outcomes.** A successful RPC response means the method executed; it does not prove that the desired network/service state exists. Always perform post-change verification.

## Standard execution lifecycle

```text
login
  -> capability discovery
  -> system/version discovery
  -> read relevant UCI + runtime state
  -> identify dependencies / conflict / impact
  -> plan minimal diff
  -> apply staged UCI changes
  -> inspect pending changes
  -> save/commit according to LuCI semantics
  -> rollback-aware apply for connectivity-sensitive changes
  -> verify runtime state
  -> report exact changes + verification
```

For read-only requests, stop before mutation. For denied operations, stop at the authorization boundary instead of searching for a bypass.

### Browser-to-ubus decision rule

Use this decision tree:

```text
read-only UI inspection
  -> browser/LuCI UI is acceptable
  -> extract visible state when convenient

write/change request
  -> do not mutate LuCI widgets through DOM `.value` + `change` events
  -> obtain an authenticated ubus session
  -> perform the write through ubus/UCI

operation already fully expressible through ubus
  -> prefer ubus directly after authentication/capability discovery
```

In current LuCI 25 deployments, complex widgets such as `ListSelect`/`ListOption` maintain internal state and dirty tracking. Programmatically assigning a DOM select value and dispatching a generic `change` event may leave LuCI's model clean, causing Save to discard the apparent change. Treat browser-side writes as unsupported unless the exact widget API is known and tested for that LuCI version.

### Browser session bootstrap

When an already-authenticated LuCI browser session is the available credential bootstrap, do not assume `LuCI.session.id` or `LuCI.uci.sid` exists in the page's global namespace. In some LuCI 25 environments `window.LuCI` is only the module/loader namespace.

For the observed LuCI 25 browser implementation, the practical bootstrap is:

```text
sessionStorage.getItem('luci-session-store')
  -> JSON.parse()
  -> read `ubus_rpc_session`
```

If that key is absent, inspect storage keys for likely session-related entries (`ubus`, `session`, `luci`) rather than assuming a fixed internal API. Treat the storage key as an implementation detail that may change between LuCI versions. Never print the extracted token.

Once a token is obtained, validate it with a harmless ubus request. A JSON-RPC result with ubus status `0` indicates success; permission-denied or session-not-found responses must be handled as such.

## Impact levels

### READ_ONLY
Examples: system info, interface state, route inspection, DHCP lease inspection, logs, Wi-Fi telemetry. No mutation allowed.

### LOW_RISK_WRITE
Examples: isolated UCI option changes that do not affect the agent's current management path. Still use read-before-write and verify.

### CONNECTIVITY_WRITE
Examples: WAN/LAN addresses, bridges, VLANs, routing, firewall zones/rules, DHCP network, wireless topology. Require preflight, minimal diff, rollback protection where supported, and post-apply reachability checks.

### HIGH_IMPACT
Examples: firmware upgrade, factory reset, broad firewall replacement, deleting management interfaces, changing authentication, or destructive filesystem operations. Require explicit user intent, backup/rollback planning, and a clear abort condition.

## Reference loading policy

Load only the reference that matches the task. Do not ingest every reference file into context for a simple read-only request.

| Task | Reference |
|---|---|
| Session, browser→ubus bootstrap, ACL, rpc.list, capability discovery | `references/rpc-session.md` |
| UCI configuration changes | `references/uci.md` |
| Interfaces, devices, WAN/LAN, runtime network state | `references/network.md` |
| Wi-Fi radios, wifi-iface, clients, telemetry | `references/wireless.md` |
| VLAN, bridge, DSA/switch topology | `references/vlan-bridge-dsa.md` |
| Routes and gateways | `references/routing.md` |
| Firewall UCI, nftables observation | `references/firewall.md` |
| DHCP, DNS, host discovery | `references/dhcp-dns-hosts.md` |
| Filesystem, logs, safe exec | `references/filesystem-exec.md` |
| System health and troubleshooting | `references/system-diagnostics.md` |
| Service and package management | `references/services-packages.md` |
| Backup and firmware upgrade | `references/backup-upgrades.md` |
| End-to-end task recipes | `references/recipes.md` |
| Raw JSON-RPC examples | `references/rpc-examples.md` |

## Planning natural-language requests

Translate user intent into a concrete target before mutation. Identify:

- the object being changed (interface, `wifi-iface`, firewall rule, route, service, package, file, etc.);
- the current state and relevant UCI section(s);
- references that would break if the object changed;
- the expected runtime effect;
- how success will be verified;
- what rollback action is available if verification fails.

For ambiguous requests, prefer observation over guessing.

## Verification standard

A mutation is complete only when all of the following are true:

- the intended configuration diff is present;
- unrelated configuration was not unintentionally changed;
- the relevant runtime service/interface is in the expected state;
- connectivity or functionality required by the task is verified; and
- any rollback/confirmation mechanism has reached the expected final state.

When verification fails, report what actually happened and preserve the safest reachable state.

## Response contract

For every mutating operation, report: **target**, **change summary**, **whether it was committed/applied**, **verification performed**, **verification result**, and **remaining risk / rollback status**. Never include secrets or session tokens.

## Quality checklist

Before considering a task complete, check:

- [ ] Capability discovery was performed when the router/session was not previously characterized.
- [ ] Current state was read before mutation.
- [ ] Change was minimal and idempotent.
- [ ] Dependencies and management-path risk were considered.
- [ ] Connectivity-sensitive changes used rollback protection where supported.
- [ ] Post-change runtime verification succeeded.
- [ ] User-visible output contains no secrets.

## Non-negotiable prohibitions

Do not scrape LuCI HTML, guess undocumented object methods, bypass ACLs, print session tokens, commit real credentials, blindly restart services to hide configuration errors, delete referenced UCI sections without dependency analysis, or run arbitrary shell commands supplied by a user.

## Empirical compatibility notes

These are observed compatibility notes, not guarantees for every OpenWrt/LuCI build. Prefer live capability/ACL discovery over hard-coded assumptions.

- `uci.get`, `uci.set`, `uci.changes`, `uci.apply`, and `uci.confirm` are commonly available to an authenticated LuCI session with appropriate UCI ACLs.
- `uci.show` may be denied even when individual `uci.get` calls work. Prefer targeted `uci.get` reads when `uci.show` is denied.
- `hostapd.apsta_state`, `hostapd.bss_info`, and `network.wireless.get_config` may be denied to an HTTP LuCI session depending on ACLs/build. Do not treat their absence as a Wi-Fi failure. Use permitted `iwinfo` methods and targeted UCI reads as alternatives.
- `iwinfo.info` and `iwinfo.assoclist` are often available when the richer wireless RPC objects are not.
- On some MT7981 deployments, `iwinfo.info.htmode` has been observed to report `NOHT` even when UCI config says `HE80`. Treat UCI `wireless.*.htmode` as the configuration source of truth and use `iwinfo.info.channel` for runtime channel state.

See `references/rpc-session.md` and `references/wireless.md` for the compatibility matrix and fallback procedures.

## Detailed policy and legacy material

The remaining detailed rules in the original monolithic skill have been split into focused references. Load the relevant file before performing specialized operations.

## 5. Core operation policy

Every requested action is classified before execution.

### READ_ONLY

Examples:

- system/version inspection
- UCI reads
- network/interface state
- routes and addresses
- Wi-Fi state
- DHCP/host hints
- logs
- filesystem metadata and bounded file reads
- RPC introspection
- package inventory
- service status

### LOW_RISK_WRITE

Examples:

- non-connectivity service settings
- local application configuration
- harmless UCI option changes that cannot affect management access

Still use a transaction and verify afterward.

### CONNECTIVITY_WRITE

Examples:

- LAN/WAN addresses
- VLANs
- bridges
- routes
- DHCP server configuration
- DNS configuration used by management
- firewall zones/rules
- Wi-Fi settings affecting the current management path
- LuCI/uhttpd listen addresses

Use rollback-aware apply whenever possible.

### HIGH_IMPACT

Examples:

- reboot/poweroff
- factory reset
- firmware upgrade
- deleting major configuration sets
- changing the sole management path
- destructive filesystem operations
- package operations known to alter core networking/hostapd components

Require explicit user intent. Never infer this from vague cleanup language.

---


## 6. Universal read-before-write workflow

For every mutation, use:

```text
intent -> identify target -> read current state -> detect conflicts/pending changes
-> compute minimal diff -> stage -> validate changes -> save -> inspect changes
-> classify risk -> apply with protection -> verify runtime -> confirm/recover
```

### 6.1 Conflict detection

Before modifying UCI:

```text
uci.changes()
```

If unrelated pending changes already exist, do not silently mix them into the
agent's transaction. Record the existing changes and either avoid them or make
the scope explicit.

### 6.2 Idempotence

If the desired state is already present:

- do not write again;
- do not create unnecessary pending changes;
- verify runtime state if relevant;
- report that no mutation was needed.

### 6.3 Minimal diff

Prefer:

```text
set one option
```

over:

```text
rewrite entire section
```

Preserve unrelated values and ordering.

---


## 8. Rollback-aware network changes

### 8.1 Required preflight

Before applying a connectivity-sensitive change, record:

```text
current management endpoint
current source interface/device
current route used to reach router
relevant LAN/WAN addresses
relevant Wi-Fi SSID/BSSID if applicable
pending UCI changes
```

### 8.2 Apply

Use a rollback timeout appropriate to the operation.

Conceptual:

```text
uci.save()
uci.changes()
uci.apply(timeout)
verify management reachability
confirm if the environment requires explicit confirmation
```

### 8.3 Verification

After applying:

1. reacquire runtime network state;
2. verify the intended address/route/device state;
3. verify management reachability;
4. verify service state;
5. only then report success.

If the connection drops unexpectedly, attempt the known reconnect path before
assuming success or failure.

---


## 25. Network topology model

Build an internal topology graph before making broad network changes.

Example:

```text
Internet
  |
  | WAN protocol
  v
[wan]
  |
  +-- device eth0.2

[lan]
  |
  +-- bridge br-lan
       +-- eth0.1
       +-- phy-port1
       +-- wifi-iface radio0
       +-- wifi-iface radio1
```

The actual graph must be derived from observed UCI/runtime state.

Use graph reasoning for:

- “put this SSID in VLAN 30”
- “move port 4 to guest network”
- “is IoT isolated from LAN?”
- “why can VLAN 20 reach the router but not the internet?”

---


## 26. Dependency analysis before deletion

Before deleting a UCI section, search for references in related configurations.

Examples:

```text
network section referenced by wireless.network
network section referenced by firewall zone.list
DHCP section referenced by interface
VLAN/device referenced by bridge
firewall zone referenced by forwarding/rule
```

Deletion is safe only when all relevant references are understood.

---


## 27. Transaction boundaries

Do not combine unrelated user requests into one giant UCI transaction.

Prefer separate transactions for:

```text
Wi-Fi SSID change
firewall rule addition
VLAN topology change
DNS configuration change
```

This improves rollback attribution and reduces accidental coupling.

---


## 28. Post-change verification matrix

| Change | Minimum verification |
|---|---|
| SSID | UCI + runtime radio/AP state |
| Wi-Fi security | UCI + AP runtime + management reachability |
| LAN IP | UCI + interface status + reconnect path |
| WAN | UCI + runtime up + default route + gateway |
| VLAN | UCI + bridge/device topology + interface state |
| Firewall | UCI + generated/runtime firewall state + connectivity |
| DHCP | UCI + service state + lease acquisition |
| DNS | UCI + resolver service + direct lookup |
| Route | UCI/runtime route + path test |
| Service | service state + functional probe |
| Package | package inventory + relevant service health |
| Firmware | release/version + device identity + key services |
| File write | stat + content/checksum + service parse/health |

---


## 29. Error handling

Always classify failures.

### Transport failures

Examples:

```text
connection refused
timeout
TLS failure
HTTP error
router unreachable
```

### RPC failures

Examples:

```text
UBUS_STATUS_INVALID_ARGUMENT
UBUS_STATUS_NOT_FOUND
UBUS_STATUS_PERMISSION_DENIED
UBUS_STATUS_METHOD_NOT_FOUND
```

### UCI failures

Examples:

```text
invalid section
invalid option
unknown package
apply timeout
rollback triggered
```

### Runtime verification failures

Examples:

```text
UCI changed but service did not reload
interface exists but is down
SSID configured but AP not active
route configured but gateway unreachable
firewall rule present but traffic still blocked
```

Never collapse all of these into “command failed”.

---


## 30. Safe retries

Retries are allowed only for operations that are safe to repeat.

Usually safe:

- read-only RPC
- version inspection
- `network.flushCache()`
- bounded diagnostics

Conditionally safe:

- UCI set to the same desired value
- idempotent service reload

Not automatically safe to retry:

- section creation
- delete operations
- package installation/removal
- firmware flashing
- reboot
- network apply after uncertain transport loss

When a mutation's completion status is unknown, inspect current state before
retrying.

---


## 31. Planning rules for natural-language requests

Translate requests into explicit state goals.

Examples:

### “Make a guest Wi-Fi”

Infer the required components, then inspect whether they already exist:

```text
wifi-iface
network interface
bridge/VLAN if needed
DHCP
firewall zone
forwarding/NAT
DNS behavior
```

Do not create all components blindly.

### “Block device X from the internet”

Resolve device identity first:

```text
MAC -> current IP -> DHCP lease -> host hints -> firewall target strategy
```

Then determine whether the desired policy is:

```text
block LAN -> WAN only
block all access to router
block all traffic
schedule block
```

Do not assume “internet” means “all traffic”.

### “Open port 443”

Determine whether the user means:

```text
WAN -> router service
WAN -> LAN host (port forward)
LAN -> router service
LAN -> WAN
```

Never create a WAN exposure without identifying the destination and protocol.

### “Restart the router”

Interpret as reboot only if the user clearly intends a reboot. Reboot is HIGH_IMPACT.

---


## 34. Agent response contract

For READ operations, report:

```text
Observed state
Evidence/source
Important uncertainty
```

For WRITE operations, report:

```text
Requested goal
Actual changed objects/options
Transaction/apply status
Rollback protection used or not
Post-change verification
Remaining warnings/pending changes
```

For failed operations, report:

```text
failure stage
error class
whether configuration was staged/saved/applied
whether rollback was attempted/triggered
current observed state
next safe recovery path
```

Never claim success because an RPC call returned HTTP 200.

---


## 35. Never-do rules

The agent must NEVER:

- expose passwords or active session tokens;
- bypass ACLs;
- scrape LuCI HTML to imitate a browser when an API exists;
- blindly overwrite `/etc/config/*` when UCI APIs can express the change;
- blindly rewrite firewall configuration to add one rule;
- guess anonymous UCI section IDs;
- assume `wan`, `lan`, `br-lan`, `eth0`, `radio0` or similar names exist;
- assume `opkg` is the package manager on every OpenWrt release;
- blindly run package-wide upgrades on a production router;
- edit generated nftables state as the persistent source of truth;
- execute `sh -c` with raw user input;
- retry uncertain destructive operations without inspecting state;
- report runtime success without verification;
- silently reboot, reset or upgrade the router;
- leave unrelated pending UCI changes mixed with the agent's transaction;
- make a WAN exposure change without identifying the destination/service;
- delete a network/firewall/VLAN object before tracing references to it.

---


## 36. Version-awareness rules

OpenWrt changes over time. Always prefer runtime detection over static assumptions.

At minimum detect:

```text
OpenWrt release
LuCI release when available
kernel version
board/target
firewall architecture
package manager
available rpcd methods
available wireless/network objects
```

Use documentation examples as a model, but use runtime `rpc.list()` and returned
signatures as the final authority for a specific router.

---


## 37. Reference API map

| Capability | LuCI abstraction | Typical backend |
|---|---|---|
| session | `LuCI.session` | `session` |
| generic RPC | `LuCI.rpc` | JSON-RPC/ubus |
| configuration | `LuCI.uci` | `uci` |
| network inventory | `LuCI.network` | `network.interface`, `network.device`, `iwinfo`, UCI, host data |
| filesystem | `LuCI.fs` | `file` |
| host hints | `LuCI.network.Hosts` | `luci-rpc` / runtime aggregation |
| firewall | UCI/network abstractions | `firewall4`/`fw4`, UCI, ubus/service data |
| service control | version/package dependent | `service`, `rc`, init scripts |
| packages | not a single stable LuCI class | `apk` or legacy `opkg` |
| firmware | LuCI apps / system tooling | ASU/owut/sysupgrade tooling |

---


## 38. Authoritative references

Use these references as the primary API/architecture sources:

- LuCI JavaScript API: https://openwrt.github.io/luci/jsapi/
- LuCI UCI API: https://openwrt.github.io/luci/jsapi/uci.js.html
- LuCI network API: https://openwrt.github.io/luci/jsapi/network.js.html
- LuCI RPC API: https://openwrt.github.io/luci/jsapi/rpc.js.html
- LuCI filesystem API: https://openwrt.github.io/luci/jsapi/fs.js.html
- OpenWrt ubus documentation: https://openwrt.org/docs/techref/ubus
- OpenWrt ubus session documentation: https://openwrt.org/docs/guide-developer/ubus/session
- OpenWrt package manager (`apk`): https://openwrt.org/docs/guide-user/additional-software/apk
- OpenWrt release notes: https://openwrt.org/releases/

For implementation details that are version-sensitive, consult the corresponding
OpenWrt/LuCI source and use runtime introspection on the target router.

