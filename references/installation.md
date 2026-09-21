# CredAgent Installation Guide

This guide covers the complete installation process for CredAgent, a zero-trust credential protection system based on FUSE.

**v0.2.0 Update**: FUSE filesystem is now integrated into the `credagent` daemon. The separate `credmon` component is no longer required.

## Components

CredAgent v0.2.0 consists of three components:

| Component | Role |
|-----------|------|
| **credagent** | Enclave daemon with integrated FUSE filesystem |
| **credctl** | CLI tool for protecting/releasing credentials |
| **credreq** | CLI tool for receiving external credentials |

**Note**: The `credmon` daemon was removed in v0.2.0 — FUSE is now built into credagent.

## Prerequisites

FUSE (Filesystem in Userspace) is required for CredAgent to intercept file access attempts.

### Install FUSE

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install fuse libfuse-dev
echo "user_allow_other" | sudo tee -a /etc/fuse.conf
```

#### CentOS/RHEL
```bash
sudo yum install fuse fuse-devel
# Or for newer versions
sudo dnf install fuse fuse-devel
sudo sh -c "echo 'user_allow_other' >> /etc/fuse.conf"
```

#### macOS
1. **Install macFUSE**
   - Download and install from https://osxfuse.github.io/
   - Or use Homebrew:
   ```bash
   brew install --cask macfuse
   ```

2. **Restart System**
   - After installation, restart your system to load the macFUSE kernel extension

### Verify FUSE Installation

#### Linux
```bash
# Check if FUSE module is loaded
lsmod | grep fuse

# Check FUSE version
fusermount -V
```

#### macOS
```bash
# Check if macFUSE is properly installed
kextstat | grep fuse
```

## Quick Install (Recommended)

Download the latest pre-built release from GitHub and run the `INSTALL` script.

### 1. Download the Package for Your OS

| OS | Download URL |
|-----------|----------------------|
| Linux x86_64 | https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_linux_amd64.tar.gz |
| macOS universal | https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_darwin_universal.tar.gz |

### 2. Extract and Run INSTALL

#### Linux
```bash
curl -LO https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_linux_amd64.tar.gz
tar -xzf credagent_0.2.0_linux_amd64.tar.gz
cd credagent_0.2.0_linux_amd64
sudo ./INSTALL
```

#### macOS
```bash
curl -LO https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_darwin_universal.tar.gz
tar -xzf credagent_0.2.0_darwin_universal.tar.gz
cd credagent_0.2.0_darwin_universal
sudo ./INSTALL
```

The `INSTALL` script will:
- Copy binaries to `/usr/local/bin/`
- Install configuration files to `/usr/local/etc/credagent/`
- Install the systemd (Linux) or launchd (macOS) service file
- Create required directories
- Reload the service manager

## Initialize and Start Services

CredAgent requires a master key and an admin password before it can protect credentials. The recommended first-time setup uses `--auto-init` so no interactive `credagent init` is needed up front.

### Step 1: Auto-Initialize (One-Time)

Run `credagent` once in the foreground with `--auto-init`. This generates the master key and a temporary random admin password, then starts serving requests.

#### Linux
```bash
sudo /usr/local/bin/credagent run \
  --mount /mnt/credagent/shielded/ \
  --socket-dir /run/credagent \
  --baseline-file /var/lib/credagent/baseline.enc \
  --config /usr/local/etc/credagent/engine/config.json \
  --auto-init
```

#### macOS
```bash
sudo /usr/local/bin/credagent run \
  --mount /usr/local/var/credagent/shielded/ \
  --socket-dir /var/run/credagent \
  --baseline-file /usr/local/var/credagent/baseline.enc \
  --config /usr/local/etc/credagent/engine/config.json \
  --auto-init
```

Wait for the log line:
```
CredAgent is ready and serving requests
```

Then press `Ctrl-C` to stop the foreground process. The FUSE filesystem will be unmounted cleanly.

> **Important**: `--auto-init` only creates a master key if one does not already exist. It does **not** reset an existing key. If an old `baseline.enc` exists, it is backed up with a timestamp suffix because the new key cannot decrypt it.

### Step 2: Start the Service

#### Linux (systemd)
```bash
sudo systemctl start credagent
sudo systemctl enable credagent
```

#### macOS (launchd)
```bash
sudo launchctl load /Library/LaunchDaemons/com.credagent.enclave.plist
```

### Step 3: Set a Real Admin Password

The `--auto-init` flag creates a random, unrecoverable admin password. You must replace it before performing any management operation such as `release`, `remove`, `update`, `attach`, `detach`, or `refresh`.

```bash
sudo /usr/local/bin/credagent init --config /usr/local/etc/credagent/engine/config.json
```

Follow the prompts:
1. `Would you like to overwrite it? (Y/N):` — choose **N** to keep the existing master key.
2. `Would you like to set/overwrite the password? (Y/N):` — choose **Y**.
3. Enter and confirm your new admin password.

After resetting the password, restart the service so the running daemon picks up the new password hash:

#### Linux
```bash
sudo systemctl restart credagent
```

#### macOS
```bash
sudo launchctl unload /Library/LaunchDaemons/com.credagent.enclave.plist
sudo launchctl load /Library/LaunchDaemons/com.credagent.enclave.plist
```

## Verify Service Status

#### Linux
```bash
# Check service status (v0.2.0+: only credagent)
sudo systemctl status credagent

# View real-time logs
sudo journalctl -u credagent -f
```

#### macOS
```bash
# Check service status
sudo launchctl list | grep credagent

# View real-time logs
tail -f /usr/local/var/log/credagent/credagent.log
```

## Verification Checklist

After installation, verify:

- [ ] FUSE is properly installed and loaded
- [ ] credagent service is running (`status` command shows active)
- [ ] FUSE mount point exists:
  - Linux: `ls -la /mnt/credagent/shielded/`
  - macOS: `ls -la /usr/local/var/credagent/shielded/`
- [ ] Socket file exists:
  - Linux: `ls -la /run/credagent/` (should contain `enclave.sock`)
  - macOS: `ls -la /var/run/credagent/`
- [ ] Admin password has been reset after `--auto-init` (see Step 3 above)

**Note (v0.2.0+)**: No `credmon` service or `monitor.sock` — FUSE is integrated into credagent.

## Troubleshooting Installation

### Services Won't Start

```bash
# Check logs for errors (v0.2.0+: only credagent)
# Linux
sudo journalctl -u credagent -n 50

# macOS
tail -n 50 /usr/local/var/log/credagent/credagent.log
```

### FUSE Mount Missing

```bash
# Restart credagent service (v0.2.0+: FUSE integrated)
# Linux
sudo systemctl restart credagent

# macOS
sudo launchctl unload /Library/LaunchDaemons/com.credagent.enclave.plist
sudo launchctl load /Library/LaunchDaemons/com.credagent.enclave.plist

# Check mount point
# Linux: ls -la /mnt/credagent/shielded/
# macOS: ls -la /usr/local/var/credagent/shielded/
```

### Permission Issues

Ensure config files have correct permissions:
```bash
sudo chmod 600 /usr/local/etc/credagent/engine/config.json
```

---

## Appendix: Build and Install from Source

If the pre-built package fails to run on your system (for example, due to an incompatible glibc or kernel version), you can build CredAgent from source.

### Requirements
- Go 1.21+
- FUSE development headers (see Prerequisites)

### Download and Build

```bash
# Download the v0.2.0 source tarball
curl -LO https://github.com/aliyun/credagent/archive/refs/tags/v0.2.0.tar.gz

# Extract
tar -xzf v0.2.0.tar.gz

cd credagent-0.2.0

# Build all binaries
go build -o credagent ./cmd/credagent
go build -o credctl ./cmd/credctl
go build -o credreq ./cmd/credreq
```

### Install

After building, run the same `INSTALL` script:

```bash
sudo ./INSTALL
```

Then follow the [Initialize and Start Services](#initialize-and-start-services) steps above.

---

**Next Steps:** After installation, see `SKILL.md` for usage patterns on protecting credential files.

**Detailed Documentation:** See `~/project/CredAgent/README.md` for complete system architecture and advanced configuration.
