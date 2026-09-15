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
