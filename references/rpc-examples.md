# OpenWrt LuCI Agent — Rpc Examples Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

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
