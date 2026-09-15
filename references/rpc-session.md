# OpenWrt LuCI Agent — Rpc Session Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

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

## 3.4 Browser → ubus session bootstrap (LuCI 25 compatibility)

When the agent is operating inside an already-authenticated LuCI browser context, do **not** assume that `LuCI.session.id` or `LuCI.uci.sid` is available globally. In some LuCI 25 deployments, `window.LuCI` exists only as the module/loader namespace and the individual components are not exposed there.

For the observed LuCI 25 implementation, use this order:

```text
1. sessionStorage.getItem('luci-session-store')
2. JSON.parse(value)
3. read `ubus_rpc_session`
4. validate with a harmless /ubus JSON-RPC call
```

If the key is absent, inspect storage keys for names containing `ubus`, `session`, or `luci`. This is a compatibility fallback, not a stable public LuCI API. Do not hard-code assumptions about future storage keys.

Never print, log, persist in plaintext, or return the session token to the user.

## 3.5 Session lifetime and rollback timing

OpenWrt's `session.login` timeout is in seconds and defaults to **300 seconds (5 minutes)**. The timeout is automatically reset when the session is used. A longer timeout may be requested at login when appropriate.

Do not confuse the ubus session timeout with the rollback/apply timeout. They are separate mechanisms.

For a connectivity-sensitive `uci.apply({ rollback: true, timeout: N })` operation:

```text
apply
  -> verify immediately
  -> call uci.confirm as soon as the intended state is known-good
```

Do not spend the rollback window on unrelated diagnostics. If the session has expired, re-authenticate and retry the confirmation only if the rollback state is still valid and the exact deployed LuCI/UCI behavior permits it.

## 3.6 Observed LuCI-session ACL compatibility matrix

The following is an **empirical compatibility matrix from LuCI HTTP sessions**, not a universal guarantee. Actual access is controlled by rpcd ACLs and the deployed packages. Use `session.access` or a harmless probe when uncertain.

| Object / method | Observed with LuCI session | Preferred fallback |
|---|---:|---|
| `uci.get` | ✅ | targeted `uci.get` |
| `uci.set` | ✅ | — |
| `uci.changes` | ✅ | — |
| `uci.apply` | ✅ | — |
| `uci.confirm` | ✅ | — |
| `uci.show` | ❌ / may be denied | targeted `uci.get` |
| `hostapd.apsta_state` | ❌ / may be denied | `iwinfo` + UCI |
| `hostapd.bss_info` | ❌ / may be denied | `iwinfo.assoclist` + UCI |
| `network.wireless.get_config` | ❌ / may be denied | UCI `wireless` + permitted runtime APIs |
| `iwinfo.info` | ✅ | UCI for configuration values |
| `iwinfo.assoclist` | ✅ | — |
| `system.board` | ✅ | — |

Never attempt to bypass a denied method by switching to an unauthorized interface, stealing another session, or escalating privileges. OpenWrt documents `session.access` as the way to query whether a session is authorized for a given ubus procedure.
