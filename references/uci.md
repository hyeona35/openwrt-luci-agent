# OpenWrt LuCI Agent — Uci Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

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
