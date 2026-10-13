---
name: credagent
description: Protects file-based credentials with CredAgent, a zero-trust FUSE system that encrypts a secret at rest and serves it back at the same path, so only authorized and SHA256-measured programs can open it. Applicable when a user asks to lock down or harden credential files (SSH keys such as `~/.ssh/id_rsa`, cloud CLI configs such as `~/.aws/credentials` or `~/.aliyun/config.json`, API tokens, OAuth cookies, `.env` secrets), restrict access so only specific authorized binaries or scripts can read them, audit which programs may read secrets, or undo/release existing protection. Triggers on mentions of credential protection, secret hardening, shielding credential files, zero-trust file access, CredAgent, credctl, or preventing AI agents and tools from exfiltrating keys and tokens.
version: 0.2.0
license: Apache-2.0
---

# credagent

Protect file-based credentials using CredAgent — a zero-trust FUSE-based protection system.

## Core Workflow

### Step 0: Verify CredAgent Is Installed

CredAgent is separate software and is **not** bundled with this skill. Check it is installed and running before doing anything else:

```bash
command -v credctl && credctl version              # binary present?
credctl list --json >/dev/null 2>&1 && echo OK      # daemon answering?
ls -d /mnt/credagent/shielded 2>/dev/null \
  || ls -d /usr/local/var/credagent/shielded 2>/dev/null   # FUSE mount up?
```

If any check fails, **stop — do not attempt to protect files.** Tell the user CredAgent has to be installed first and walk them through [`references/installation.md`](references/installation.md):

- **Prebuilt binary:** `curl` the [Release](https://github.com/aliyun/credagent/releases) tarball for their OS, extract, `sudo ./INSTALL`
- **From source:** download the source tarball, `CGO_ENABLED=0 go build -o credagent ./cmd/credagent` (same for `credctl`, `credreq`; on macOS use `CGO_ENABLED=1` and build natively), then `sudo ./INSTALL`
- **Then bootstrap once:** `sudo credagent run --mount /mnt/credagent/shielded/ --socket-dir /run/credagent --baseline-file /var/lib/credagent/baseline.enc --config /usr/local/etc/credagent/engine/config.json --auto-init`, Ctrl-C after "ready and serving requests", start the service, and reset the random admin password with `sudo credagent init --config ...`

This needs root and FUSE, so the user must approve the commands — you cannot do it silently on their behalf. Proceed to Step 1 only once `credctl version` prints a version and the daemon answers.

### Step 1: Find the Credential File

Identify which file needs protection:

| Program | Common Credential Path |
|---------|----------------------|
| SSH/Git | `~/.ssh/id_rsa` |
| Claude CLI | `~/.claude/settings.json` |
| Aliyun CLI | `~/.aliyun/config.json` |
| AWS CLI | `~/.aws/credentials` |
| Docker | `~/.docker/config.json` |

**Tip:** Check `~/.config/` or `~/.<program>/` directories, or consult the program's documentation.

### Step 2: Find the Authorized Program

Determine which program should access the credential:

```bash
# Find binary path
which <command>
# Example: which ssh → /usr/bin/ssh
```

**Important:**
- For scripts (Python, Node.js, etc.): use the **script path**, not the interpreter
- For Git SSH access: use `/usr/bin/ssh`, not `git`

### Step 3: Protect the Credential

```bash
credctl protect --strategy BASIC --program-permit <binary-path> --measure true|--false <credential-file>
```

**Key parameters:**
- `--program-permit`: Required. Path to authorized binary
- `--measure`: **Required (v0.2.0+)**. Must be `true` or `false`
  - `true`: Verify binary SHA256 (use for stable binaries like `/usr/bin/ssh`)
  - `false`: Skip integrity check (use for scripts or frequently-updated binaries)
- `--open-frequency`: Optional. Limit opens per PID (e.g., `2` for SSH during git push)
- `--writable`: Optional, default `false`. Set `true` for credentials that need updates (OAuth tokens, cookies)

**Examples:**
```bash
# SSH key - ssh opens twice during git operations
credctl protect --strategy BASIC --program-permit /usr/bin/ssh --measure true --open-frequency 2 ~/.ssh/id_rsa

# API key for script
credctl protect --strategy BASIC --program-permit /path/to/script.py --measure true --open-frequency 1 /path/to/api-key.txt

# OAuth config that needs updates
credctl protect --strategy BASIC --program-permit /usr/bin/aliyun --measure true --writable true ~/.aliyun/config.json
```

### Step 4: Verify Protection

```bash
# Check symlink created
ls -la <credential-file>
# Should show: -> /mnt/credagent/shielded/<id>

# List all protected files
credctl list --json

# Inspect specific file details
credctl inspect <id>
```

**Confirm it works:** Run the authorized program and check logs:
```bash
sudo journalctl -u credagent --since "2 minutes ago" | grep permitted
```

---

### Step 5: Security Risk Check (Important!)

CredAgent protects the **credential file**, but not the **authorized program** or its environment. If the program's directory tree is writable by untrusted users, attackers can inject malicious code without modifying the program itself.

**Check program and entire directory tree:**
```bash
# Check program file
ls -la <program-path>

# Check program directory and ALL contents
ls -laR $(dirname <program-path>)

# Find world-writable files/dirs in the tree
find $(dirname <program-path>) -perm -o+w -ls 2>/dev/null

# Check immutable flag
lsattr <program-path>
```

**Secure configuration:**
- ✅ Program owned by `root` with `o-w` (others cannot write)
- ✅ **Entire directory tree** owned by `root` with `o-w`
- ✅ No world-writable files or subdirectories
- ✅ Or use immutable flag on critical files: `sudo chattr +i <file>`

**Risk scenarios:**

| Scenario | Risk | Attack Vector |
|----------|------|---------------|
| Script in user-writable dir | **CRITICAL** | Inject `package.py` to shadow imports |
| Dir has `o+w` permission | **CRITICAL** | Create/replace any file in tree |
| Subdir is world-writable | **HIGH** | Place malicious deps in `__pycache__/`, `vendor/`, etc. |
| Config file in dir is writable | **HIGH** | Modify behavior via config injection |
| System binary (`/usr/bin/*`) | **LOW** — Entire tree is root-owned | N/A |

**Python import shadowing attack:**
```bash
# Attacker doesn't need to modify script.py!
# Just create malicious file in same directory:

$ ls -la /home/user/project/
-rw-r--r-- 1 user user script.py
-rw-r--r-- 1 user user requests.py  # <-- Attacker created this!

# When script.py runs: import requests
# Python loads attacker's requests.py, not the real package!
```

**Other attack vectors:**
- Replace files in `vendor/`, `node_modules/`, `lib/` subdirectories
- Modify `.env`, `config.json`, or other config files
- Inject code into `__pycache__/` or compiled bytecode
- Create symlinks to redirect file access

**Mitigation:**
```bash
# Option 1: Lock down entire directory tree (recommended)
sudo chown -R root:root /path/to/program-dir/
sudo chmod -R o-w /path/to/program-dir/

# Option 2: Make critical files immutable
sudo chattr +i /path/to/program-dir/script.py
sudo chattr -R +i /path/to/program-dir/lib/  # lock dependencies

# Option 3: Move to secure location
sudo mv /home/user/project/script.py /usr/local/bin/script.py
# Ensure /usr/local/bin/ and all contents are root-owned, o-w
```

**For project scripts requiring edits:**
1. Use version control to detect unauthorized changes
2. Restrict directory access to trusted users only
3. Consider using `--measure false` with explicit risk acceptance
4. Audit directory tree regularly: `find /path -perm -o+w`

---

## Quick Reference

| Task | Command |
|------|---------|
| List protected files | `credctl list --json` |
| View file details | `credctl inspect <id>` |
| Release (restore) | `credctl release <id>` |
| Remove (delete) | `credctl remove <original-path>` |
| Update policy | `credctl update <id> --option value` |
| Add program | `credctl attach -p <path> <id>` |
| Remove program | `credctl detach -p <path> <id>` |
| Refresh SHA256 | `credctl refresh -p <path> <id>` |

For detailed command reference, see [`references/manual.md`](references/manual.md).

---

## Troubleshooting

**Permission denied errors:** See [`references/troubleshooting.md`](references/troubleshooting.md).

**Common issues:**
- `credctl: command not found` → CredAgent isn't installed; see Step 0 and [`references/installation.md`](references/installation.md)
- `connection refused` / socket missing → the `credagent` daemon isn't running; `sudo systemctl restart credagent` (Linux) or reload the launchd plist (macOS)
- Wrong `--program-permit` path → release and re-protect with correct binary
- `--open-frequency` too low → release and re-protect with higher value
- Binary updated after protection → use `credctl refresh` or re-protect
- Need to write to credential → use `--writable true`

---

## Related Documentation

- **Installation (prerequisite — see Step 0):** [`references/installation.md`](references/installation.md)
- **Full command reference:** [`references/manual.md`](references/manual.md)
- **Troubleshooting:** [`references/troubleshooting.md`](references/troubleshooting.md)
