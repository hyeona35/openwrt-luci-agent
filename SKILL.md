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

Use this skill whenever an AI agent needs to inspect, diagnose, configure, or
manage an OpenWrt router through the LuCI/ubus control plane.

The skill deliberately follows the architecture used by LuCI's JavaScript APIs:

- `LuCI.rpc` for JSON-RPC/ubus transport and method discovery
- `LuCI.session` for authenticated session handling and ACL checks
- `LuCI.uci` for staged UCI configuration changes and rollback-aware apply
- `LuCI.network` for high-level network, device, Wi-Fi and host abstractions
- `LuCI.fs` for controlled filesystem and command operations

When a high-level LuCI abstraction is unavailable to the external agent, use the
corresponding ubus object or UCI configuration directly. Do not scrape LuCI HTML.

---

## 1. Security and trust model

### 1.1 Secrets

Treat all of the following as secrets:

- router passwords
- `ubus_rpc_session` values
- LuCI `sysauth` cookies
- API tokens
- private keys
- Wi-Fi passwords / PSKs
- VPN private keys
- credential hashes
- backup archives containing secrets
- command output containing any of the above

Secrets MUST be held in protected runtime memory, secure environment variables,
or a secret manager. Never place real secrets in prompts, ordinary logs, telemetry,
examples, generated reports, or user-visible tool output.

When showing configuration, redact sensitive values rather than deleting the
whole section. Examples:

```text
key=<REDACTED>
password=<REDACTED>
private_key=<REDACTED>
ubus_rpc_session=<REDACTED>
```

### 1.2 Authorization

An ubus method being discoverable does not imply that the current session may use
it. Always obey rpcd ACLs.

If the router returns permission denied:

1. stop the attempted operation;
2. identify the missing capability at a high level;
3. do not search for an alternate endpoint to bypass ACLs;
4. do not switch to an unaudited root shell merely to defeat the restriction.

### 1.3 Input trust

Treat user-provided values as untrusted data, especially:

- interface names
- UCI section names
- file paths
- service names
- package names
- hostnames
- shell command fragments
- firewall expressions
- nftables expressions
- regex-like filters

Validate or constrain them before interpolation into commands or configuration.
Prefer structured RPC arguments to stringly-typed shell commands.

---

## 2. Primary architecture

### 2.1 Preferred control plane

Use authenticated HTTP ubus JSON-RPC whenever possible:

```text
POST http(s)://<ROUTER>/ubus
Content-Type: application/json
```

Typical JSON-RPC request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "call",
  "params": [
    "<UBUS_RPC_SESSION>",
    "<OBJECT>",
    "<METHOD>",
    { "<ARGUMENT>": "<VALUE>" }
  ]
}
```

The exact HTTP path may depend on the router's LuCI/uhttpd configuration. A
browser-side LuCI deployment commonly uses its configured admin ubus endpoint.
An external agent may use `/ubus` when that endpoint is exposed.

Do not assume that legacy `/cgi-bin/luci/rpc/` endpoints are the modern LuCI API.
Use them only when the installed system explicitly requires a compatibility path.

### 2.2 LuCI abstraction is not the same as one ubus call

`LuCI.network` is a high-level aggregation layer. One network object may combine
UCI state, `network.interface`, `network.device`, `iwinfo`, host hints and other
runtime data.

Therefore:

> Never invent a one-to-one mapping when the LuCI class is actually an aggregation.

Use direct ubus calls only after identifying the underlying information source.

---

## 3. Session lifecycle

### 3.1 Authenticate

Use `session.login` with protected credentials.

Conceptual request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "call",
  "params": [
    "00000000000000000000000000000000",
    "session",
    "login",
    {
      "username": "${OPENWRT_USERNAME}",
      "password": "${OPENWRT_PASSWORD}",
      "timeout": 900
    }
  ]
}
```

Store the returned session identifier only in protected agent state.

### 3.2 Session refresh

On session-expired errors:

```text
refresh credentials -> session.login -> replace in-memory session -> retry once
```

Do not retry indefinitely and do not expose the old token.

### 3.3 Session and ACL checks

For privileged operations, consider checking access before performing the action:

```text
session.access(object, function)
```

Conceptual `session.access` arguments:

```json
{
  "ubus_rpc_session": "<SESSION>",
  "object": "uci",
  "function": "set"
}
```

If authorization is denied, stop.

### 3.4 Session cleanup

Destroy the session when the surrounding integration explicitly requires active
session cleanup. Do not persist active sessions longer than necessary.

---

## 4. Capability discovery before execution

An agent must be version- and device-aware.

### 4.1 Recommended bootstrap sequence

For a new router/session, inspect in this order:

```text
1. session.login
2. rpc.list()
3. system board/info/version data
4. visible network.* methods
5. visible iwinfo methods
6. visible luci-rpc methods, when installed
7. visible service/rc methods, when installed
8. visible file methods and allowed paths
9. package manager availability
```

### 4.2 RPC method discovery

LuCI `rpc.list()` maps to a JSON-RPC `list` request.

Conceptual form:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "list",
  "params": [
    "uci",
    "network.interface",
    "network.device",
    "iwinfo",
    "system"
  ]
}
```

Use the returned signatures as the authoritative interface for that router.

### 4.3 Capability matrix

Maintain a runtime capability matrix such as:

```text
capabilities = {
  uci: true,
  network_interface_dump: true,
  network_device_status: true,
  iwinfo: true,
  luci_rpc: false,
  file_read: true,
  file_write: false,
  file_exec: true,
  service_api: false,
  rc_api: true,
  package_manager: "apk"
}
```

Do not claim support for a feature until its required object/method/ACL has been
observed.

---

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

## 7. LuCI.uci — full configuration workflow

### 7.1 Read operations

Use concepts equivalent to:

```text
uci.load(config)
uci.get(config, section)
uci.get(config, section, option)
uci.sections(config, type)
uci.sections(config)
uci.changes()
```

### 7.2 Section addressing

Resolve sections from observed state.

Supported conceptual forms include:

```text
named section: lan
extended anonymous reference: @interface[0]
resolved anonymous ID: cfgXXXXXX
temporary local ID: newXXXXXX
```

Do not guess anonymous section identities based solely on array indexes.

### 7.3 Create

Use a stable name when supported:

```text
uci.add(config, type, name, values)
```

If an existing resource with the same desired identity is found, reuse it instead
of creating duplicates.

### 7.4 Modify

```text
uci.set(config, section, option, value)
```

For list options, preserve their semantics. Do not turn a list into a string
merely because the current response is JSON-serializable.

### 7.5 Delete

Delete only the requested section or option.

```text
uci.delete(config, section, option?)
```

Before section deletion, identify references from other UCI configs.

### 7.6 Reorder

When the order of firewall rules, interfaces, or other ordered sections matters,
use the UCI ordering abstraction rather than reconstructing the whole file.

### 7.7 Save vs apply

These are distinct:

```text
set/add/delete/order
    ↓
save()
    ↓
remote UCI saved / affected configs reloaded
    ↓
changes()
    ↓
apply(timeout)
    ↓
rollback-protected runtime application
    ↓
confirm()
```

Modern `LuCI.uci.apply(timeout)` is explicitly rollback-aware. Do not describe
plain `uci.commit` as equivalent to `LuCI.uci.apply()`.

### 7.8 Direct low-level UCI commit

A direct `uci.commit` ubus call is a lower-level mechanism. Use it only when the
operation explicitly requires it and the loss of LuCI's higher-level apply
semantics is understood.

For network-affecting changes, prefer the LuCI-compatible rollback path.

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

## 12. VLANs, bridges and switch topology

Treat VLAN/bridge changes as connectivity-sensitive.

### 12.1 Preflight

Inspect:

```text
network config sections
bridge/device sections
bridge-vlan sections if present
CPU/DSA port topology if exposed
current interface-device relations
firewall zone membership
DHCP server bindings
```

### 12.2 DSA awareness

Do not assume legacy `swconfig` semantics on modern DSA targets.

If topology is unclear:

```text
rpc.list() -> inspect available network objects
system board -> identify target
UCI network -> inspect device/bridge-vlan structure
runtime device state -> correlate actual interfaces
```

### 12.3 Safe VLAN workflow

```text
backup relevant UCI
read topology
find all references to target VLAN/device
stage minimal VLAN/bridge change
check dependent interfaces and firewall zones
save
inspect uci.changes
apply with rollback
verify management and tagged/untagged paths
```

Never remove the management VLAN merely because it appears unused in a single
UCI file.

---

## 13. Routing and gateways

Routing changes can silently break both WAN and inter-VLAN access.

### 13.1 Read state

Prefer runtime network APIs where available. Otherwise use controlled diagnostics:

```text
ip route
ip -6 route
ip rule
```

### 13.2 Diagnose a routing issue

Determine:

```text
source interface
source address
selected route
next hop/gateway
metric
policy-routing table/rule
interface state
```

Do not fix a routing problem by adding another default route without first
understanding existing metrics and policy rules.

### 13.3 Route mutations

Before adding/removing a route:

- identify duplicate routes;
- inspect metrics;
- inspect policy rules;
- determine whether the route is generated by a protocol handler;
- change the UCI source rather than only the current runtime state when persistence
  is desired.

---

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

## 17. LuCI.fs — filesystem and command execution

### 17.1 Available abstractions

Use LuCI-style operations:

```text
fs.list(path)
fs.stat(path)
fs.stat(path) / fs.lstat(path)
fs.read(path)
fs.lines(path)
fs.write(path, data, mode)
fs.remove(path)
fs.exec(command, params, env)
```

`fs.exec()` maps to the rpcd `file.exec` method and is capable of invoking commands
permitted by the caller's ACL.

### 17.2 Path safety

Before accessing a path:

1. resolve/normalize it where appropriate;
2. ensure it is within the intended operational scope;
3. never accept `../` traversal from untrusted data;
4. avoid reading credential files unless the task truly requires them;
5. redact secrets from returned content.

### 17.3 File size limits

The ubus file transport has message/output limits. For large or binary files,
LuCI may use a cgi-io helper path instead of ubus. Follow the installed API and ACL
behavior rather than assuming every file can be fetched through `file.read`.

### 17.4 Write safety

Before `fs.write()` or `fs.remove()`:

```text
read/stat -> determine ownership/mode -> backup when appropriate -> mutate -> verify
```

Never overwrite a critical config file blindly when a supported UCI API exists.

### 17.5 Command execution

Prefer:

```text
fs.exec("ip", ["addr"])
```

over:

```text
fs.exec("sh", ["-c", "ip addr | ... user input ..."])
```

Never interpolate raw user input into a shell expression.

### 17.6 Diagnostic command allowlist

Routine read-only commands may include:

```text
ubus
ip
route
ss
cat
logread
dmesg
uname
uptime
free
df
mount
ls
find (bounded scope)
ps
nslookup
ping
traceroute / tracepath when installed
ethtool when installed
iw / iwinfo when installed
fw4 / nft for read-only inspection when installed
```

The agent must discover availability and use only the minimum command necessary.

---

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

## 24. Firewall4 / nftables observation

Use `fw4`/`nft` only for runtime inspection unless the user explicitly asks for
low-level manipulation.

Persistent policy should normally be changed through `/etc/config/firewall` and
then applied by OpenWrt's firewall machinery.

Do not edit generated nftables state as if it were the persistent source of truth.

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

## 33. Examples of JSON-RPC building blocks

### 33.1 UCI get

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "call",
  "params": [
    "<SESSION>",
    "uci",
    "get",
    {
      "config": "network",
      "section": "lan"
    }
  ]
}
```

### 33.2 UCI set

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "call",
  "params": [
    "<SESSION>",
    "uci",
    "set",
    {
      "config": "network",
      "section": "lan",
      "values": {
        "ipaddr": "192.168.10.1"
      }
    }
  ]
}
```

### 33.3 UCI apply

Conceptually:

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "call",
  "params": [
    "<SESSION>",
    "uci",
    "apply",
    {
      "timeout": 30,
      "rollback": true
    }
  ]
}
```

The exact response and confirmation behavior depend on the deployed LuCI/UCI API.

### 33.4 Network interface dump

```json
{
  "jsonrpc": "2.0",
  "id": 20,
  "method": "call",
  "params": [
    "<SESSION>",
    "network.interface",
    "dump",
    {}
  ]
}
```

### 33.5 File read

```json
{
  "jsonrpc": "2.0",
  "id": 30,
  "method": "call",
  "params": [
    "<SESSION>",
    "file",
    "read",
    {
      "path": "/etc/openwrt_release"
    }
  ]
}
```

### 33.6 Command execution

```json
{
  "jsonrpc": "2.0",
  "id": 31,
  "method": "call",
  "params": [
    "<SESSION>",
    "file",
    "exec",
    {
      "command": "ip",
      "params": ["addr"]
    }
  ]
}
```

Never substitute user-supplied shell syntax into `command` or `params` without
validation.

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
