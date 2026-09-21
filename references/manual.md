# CredAgent Command Reference Manual

Complete reference for CredAgent CLI commands. **v0.2.0**

---

## credctl Commands

### protect — Protect a credential file

Move credential content into the enclave, delete original, and create symlink.

```bash
credctl protect --strategy BASIC --program-permit <path> --measure true|false [options] <file>
```

**Parameters:**
| Flag | Required | Description |
|------|----------|-------------|
| `--strategy`, `-s` | No | Protection strategy: `BASIC`, `ONE_TIME_USE`, `PID_BIND`, `SPEC_VERIFICATION`. Default: `BASIC` |
| `--program-permit`, `-p` | **Yes** | Authorized program path. Can specify multiple times |
| `--measure`, `-m` | **Yes (v0.2.0+)** | Measure binary SHA256: `true` or `false`. No default |
| `--open-frequency`, `-o` | No | Max opens per PID. Default: `0` (unlimited) |
| `--writable`, `-w` | No | Allow writes: `true` or `false`. Default: `false` |
| `--config`, `-c` | No | Config file path |
| `--socket-dir`, `-d` | No | Socket directory. Default: `/run/credagent` |

**Examples:**
```bash
# Basic protection with binary measurement
credctl protect --strategy BASIC --program-permit /usr/bin/ssh --measure true ~/.ssh/id_rsa

# With open frequency limit
credctl protect --strategy BASIC --program-permit /usr/bin/aliyun --measure true --open-frequency 1 ~/.aliyun/config.json

# Allow credential updates (OAuth, cookies)
credctl protect --strategy BASIC --program-permit /usr/bin/aliyun --measure true --writable true ~/.aliyun/config.json

# Multiple authorized programs
credctl protect --strategy BASIC -p /usr/bin/psql -p /usr/bin/pg_dump --measure true ~/.pgpass
```

---

### list — List protected files

```bash
credctl list [--json]
```

**Parameters:**
| Flag | Description |
|------|-------------|
| `--json`, `-j` | Output as JSON (recommended for parsing) |

**Example output (JSON):**
```json
[
  {
    "id": "59524298d125",
    "target_path": "/home/user/.ssh/id_rsa",
    "strategy": "BASIC",
    "permissions": "0600 (uid:1000,gid:1000)",
    "permitted_program": "/usr/bin/ssh",
    "program_sha256": ["47adf415134df7eff017e9557634696ba6b2a09f5a3bb1436d91d99b8a1cd5a6"],
    "open_frequency": "2",
    "writable": false
  }
]
```

---

### inspect — View protected file details

```bash
credctl inspect <id> [--json]
```

**Parameters:**
| Flag | Description |
|------|-------------|
| `--json`, `-j` | Output as JSON |

**Example:**
```bash
credctl inspect 59524298d125 --json
```

---

### release — Restore credential to original location

```bash
credctl release <id>
```

**Note:** Requires interactive admin password input. Deletes symlink and restores original file.

---

### remove — Delete protected credential permanently

```bash
credctl remove <original-path>
```

**Note:** Requires interactive admin password input. Uses original file path, not ID. **Does not restore** — deletes from enclave.

---

### update — Modify protection policy

```bash
credctl update <id> [options]
```

**Parameters:**
| Flag | Description |
|------|-------------|
| `--strategy`, `-s` | New strategy |
| `--program-permit`, `-p` | **Must use with `--measure`**. Replaces entire program list |
| `--measure`, `-m` | **Required with `--program-permit`**. `true` or `false` |
| `--open-frequency`, `-o` | New open frequency limit |
| `--writable`, `-w` | New writable setting |

**Examples:**
```bash
credctl update 59524298d125 --strategy BASIC
credctl update 59524298d125 --program-permit /usr/bin/psql --measure true
credctl update 59524298d125 --open-frequency 10
credctl update 59524298d125 --writable true
```

**Note:** `--program-permit` replaces the entire list. For incremental changes, use `attach`/`detach`.

---

### attach — Add authorized programs (incremental)

```bash
credctl attach -p <path> [-p <path>...] <id>
```

**Example:**
```bash
credctl attach -p /usr/bin/psql -p /usr/bin/pg_dump 59524298d125
```

**Output:**
```
Successfully updated credential 59524298d125
Added (1):   [/usr/bin/psql]
Skipped (1): [/usr/bin/pg_dump] (already exist)
```

---

### detach — Remove authorized programs

```bash
credctl detach -p <path> [-p <path>...] <id>
```

**Example:**
```bash
credctl detach -p /usr/bin/pg_dump 59524298d125
```

**Note:** Program path must exist on disk.

---

### refresh — Update SHA256 of authorized programs

```bash
credctl refresh [-p <path> [-p <path>...]] <id>
```

**Examples:**
```bash
# Refresh specific programs
credctl refresh -p /usr/bin/psql 59524298d125

# Refresh ALL authorized programs (v0.2.0+)
credctl refresh 59524298d125
```

**Use when:** System updates change binary SHA256, causing measure verification failures.

---

### dump — Export credential content

```bash
credctl dump [-w <output-file>] <id>
```

**Examples:**
```bash
# Output to stdout
credctl dump 59524298d125

# Write to file
credctl dump -w /tmp/cred.json 59524298d125
```

---

### restore — Update credential content

```bash
credctl restore (-f <file> | -s <secret>) <id>
```

**Parameters:**
| Flag | Description |
|------|-------------|
| `--file`, `-f` | Read new content from file |
| `--secret`, `-s` | New content as string (appears in process list) |

**Examples:**
```bash
# From file
credctl restore -f /tmp/new-cred.json 59524298d125

# From string
credctl restore -s 'new-secret-value' 59524298d125

# Pipeline edit
credctl restore -s "$(credctl dump 59524298d125 | sed 's/old/new/g')" 59524298d125
```

---

## credagent Commands

### run — Start the daemon

```bash
credagent run [options]
```

**Parameters:**
| Flag | Description | Default |
|------|-------------|---------|
| `--config`, `-c` | Config file path | `/usr/local/etc/credagent/engine/config.json` |
| `--mount` | FUSE mount point | `/mnt/credagent/shielded` |
| `--socket-dir`, `-d` | Socket directory | `/run/credagent` |
| `--baseline-file`, `-b` | Encrypted baseline storage | `/var/lib/credagent/baseline.enc` |
| `--key-file`, `-k` | Master key file | — |
| `--log-level`, `-l` | Log level | `info` |

---

### init — Initialize system

```bash
credagent init [options]
```

Generates master key and sets admin password.

**Parameters:**
| Flag | Description | Default |
|------|-------------|---------|
| `--config`, `-c` | Config file path | `/usr/local/etc/credagent/engine/config.json` |
| `--key-file`, `-k` | Master key output path | `/usr/local/etc/credagent/master.key` |

---

## Quick Decision Guide

| Goal | Command |
|------|---------|
| Protect new credential | `credctl protect` |
| See what's protected | `credctl list --json` |
| View details | `credctl inspect <id>` |
| Add another program | `credctl attach -p <path> <id>` |
| Remove a program | `credctl detach -p <path> <id>` |
| Binary updated | `credctl refresh -p <path> <id>` |
| Edit content | `credctl dump` → edit → `credctl restore` |
| Temporarily disable | `credctl release <id>` |
| Delete permanently | `credctl remove <path>` |
| Change policy | `credctl update <id> --option value` |
| **Check program security** | `ls -la <path>` + `ls -ld $(dirname <path>)` |

---

## Security Checklist

Before protecting a credential, verify:

### Program Security

```bash
# 1. Check program ownership and permissions
ls -la /path/to/program
ls -ld $(dirname /path/to/program)

# 2. Check for immutable flag
lsattr /path/to/program

# Expected secure state:
# - Owned by root
# - Others cannot write (o-w)
# - Or: immutable flag set (i)
```

### Risk Assessment

| Program Location | Risk Level | Action |
|-----------------|------------|--------|
| `/usr/bin/*`, `/usr/sbin/*` | ✅ LOW | Safe to use |
| `/usr/local/bin/*` (root-owned) | ✅ LOW | Safe to use |
| User home directory scripts | ⚠️ HIGH | Fix permissions or use `chattr +i` |
| Project directory scripts | ⚠️ HIGH | Ensure directory access controls |
| World-writable directories | ❌ CRITICAL | Do not use |

### Python Script Risks

For Python scripts, additional attack vectors exist:

**Import shadowing:** Attacker creates file with same name as imported package.
```bash
# If script.py is in /home/user/project/ and contains:
#   import requests

# Attacker creates /home/user/project/requests.py with:
#   def get(*args): print("stolen creds"); return original_get(*args)
```

**Mitigation:**
- Use absolute imports
- Keep scripts in secure directories
- Set immutable flag: `sudo chattr +i script.py`
- Audit entire directory tree for write permissions

### Fixing Insecure Programs

```bash
# Option 1: Make immutable (best for stable scripts)
sudo chattr +i /path/to/program

# Option 2: Fix ownership (for system-wide tools)
sudo chown root:root /path/to/program
sudo chmod 755 /path/to/program

# Option 3: Move to secure location
sudo mv /home/user/project/script.py /usr/local/bin/script.py
sudo chmod 755 /usr/local/bin/script.py
```

---

## Parameter Reference

### --measure (v0.2.0+)

**Required with `--program-permit`**. No default.

| Value | Use Case |
|-------|----------|
| `true` | Stable binaries (`/usr/bin/ssh`, `/usr/bin/aliyun`) |
| `false` | Scripts, frequently-updated binaries |

### --writable

| Value | Use Case |
|-------|----------|
| `false` (default) | Static credentials (SSH keys, API keys) |
| `true` | Dynamic credentials (OAuth tokens, cookies, session configs) |

### --open-frequency

| Value | Use Case |
|-------|----------|
| `1` | Single-use auth |
| `2` | SSH (git push opens twice) |
| `0` | Unlimited (trusted programs) |

---

## See Also

- **Installation:** [`installation.md`](installation.md)
- **Troubleshooting:** [`troubleshooting.md`](troubleshooting.md)
- **Main skill guide:** [`../SKILL.md`](../SKILL.md)
