# OpenWrt LuCI Agent — Filesystem Exec Reference

This file is a task-specific reference. Load it when the agent needs the APIs or procedures covered below. Follow the safety and transaction rules in the parent `SKILL.md`.

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
