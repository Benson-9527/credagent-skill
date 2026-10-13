# credagent-skill

**English** | [简体中文](README.zh-CN.md)

A skill that lets an AI agent protect your credential files (SSH keys, API keys, OAuth cookies, `.env` secrets, etc.) — using **CredAgent** from [aliyun/credagent](https://github.com/aliyun/credagent), which brings credential files under management in an encrypted virtual filesystem, where a program is permitted to access a credential only after its measurement is verified as legitimate.

Just tell your agent in plain language what to protect. The agent follows this skill to find the credential, identify the program that should access it, run `credctl` to move the secret into a FUSE enclave, and verify the result. The original file becomes a symlink — only explicitly authorized programs (verified by path and optional SHA256 measurement) can read it, so AI agents, rogue scripts, and unapproved tools are blocked from exfiltrating keys and tokens.

> This repository ships the **skill** (instructions + documentation that guide an agent). The credential-protection software itself is [CredAgent](https://github.com/aliyun/credagent) — install it once on the machine first (see [Install CredAgent](#install-credagent-one-time)). After that the agent drives everything: it finds the credential, picks the authorized program, runs `credctl`, and verifies the result.

## Install the skill

```bash
npx skills add Benson-9527/credagent-skill
```

One command, no build step — it copies this repo's `SKILL.md` and `references/` into your agent's skills directory. Works with any agent that reads the Agent Skills `SKILL.md` format (Claude Code, Qoder, Codex CLI, OpenClaw, …). If you prefer doing it by hand, drop this folder into your agent's skills directory instead.

## Features

- **File-level protection** for any credential file (SSH keys, `~/.aws/credentials`, `~/.aliyun/config.json`, `.env`, etc.)
- **Zero-trust access** — only explicitly permitted binaries/scripts can open protected files
- **Integrity measurement** — verify authorized binaries by SHA256
- **Open-frequency limiting** — cap how many times a PID may open a credential
- **Writable mode** for credentials that need updates (OAuth tokens, cookies)
- **Directory-tree risk auditing** — surface import-shadowing and world-writable vulnerabilities in the program's tree
- **Full lifecycle** — protect, inspect, update, release, and remove

## Repository layout

```
.
├── SKILL.md                 # Skill instructions the agent follows
└── references/
    ├── installation.md      # CredAgent + FUSE install, init, and verification steps
    ├── manual.md            # Full credctl / credreq command reference
    └── troubleshooting.md   # Permission-denied diagnostics & fixes
```

## Install CredAgent (one-time)

The skill drives `credctl` against a running `credagent` daemon, so CredAgent must be installed on the machine first. This needs root and FUSE — an agent can run the commands, but you have to approve them. Pick either path; both end at the same place.

**Option A — prebuilt binaries (fastest)**

```bash
# Linux x86_64
curl -LO https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_linux_amd64.tar.gz
tar -xzf credagent_0.2.0_linux_amd64.tar.gz
cd credagent_0.2.0_linux_amd64
sudo ./INSTALL
```

macOS uses `credagent_0.2.0_darwin_universal.tar.gz` and the same `./INSTALL`. The script installs FUSE deps, copies the three binaries to `/usr/local/bin/`, drops the config templates in `/usr/local/etc/credagent/`, and registers the systemd (Linux) or launchd (macOS) service.

**Option B — build from source** (Go 1.21+)

```bash
curl -LO https://github.com/aliyun/credagent/archive/refs/tags/v0.2.0.tar.gz
tar -xzf v0.2.0.tar.gz
cd credagent-0.2.0

CGO_ENABLED=0 go build -o credagent ./cmd/credagent
CGO_ENABLED=0 go build -o credctl   ./cmd/credctl
CGO_ENABLED=0 go build -o credreq   ./cmd/credreq

sudo ./INSTALL
```

On macOS use `CGO_ENABLED=1` and build natively — process attestation there calls `libproc`/`sysctl` through cgo, so cross-compiling from Linux does not work.

**Then bootstrap the daemon** (generates the master key and mounts the FUSE enclave):

```bash
sudo /usr/local/bin/credagent run \
  --mount /mnt/credagent/shielded/ \
  --socket-dir /run/credagent \
  --baseline-file /var/lib/credagent/baseline.enc \
  --config /usr/local/etc/credagent/engine/config.json \
  --auto-init
# Wait for "CredAgent is ready and serving requests", then Ctrl-C

sudo systemctl start credagent && sudo systemctl enable credagent

# --auto-init generated a random admin password; replace it before using
# release / remove / update / attach / detach / refresh
sudo /usr/local/bin/credagent init --config /usr/local/etc/credagent/engine/config.json
sudo systemctl restart credagent
```

Confirm with `credctl version` and `ls -la /mnt/credagent/shielded/`. Full details — FUSE prerequisites, launchd on macOS, the verification checklist, and install troubleshooting — are in [`references/installation.md`](references/installation.md).

## Quick start

With CredAgent installed, you don't run any commands yourself — just talk to your agent. The skill guides it through the whole flow.

**Example 1 — protect an SSH key:**

```
You: Please protect ~/.ssh/id_rsa

Agent: I'll protect your SSH private key.
  1. Credential file: ~/.ssh/id_rsa
  2. Authorized program: /usr/bin/ssh (used by git over ssh)
  3. Protecting (SHA256 measured, max 2 opens per PID):
     $ credctl protect --strategy BASIC \
         --program-permit /usr/bin/ssh --measure true \
         --open-frequency 2 ~/.ssh/id_rsa
     Successfully onboarded ~/.ssh/id_rsa, securely shredded, and
     created symlink to /mnt/credagent/shielded/a23457c0659d.
  4. Verifying:
     $ ls -la ~/.ssh/id_rsa
     lrwxrwxrwx ... ~/.ssh/id_rsa -> /mnt/credagent/shielded/a23457c0659d
✅ Done. Only /usr/bin/ssh can read this private key; other programs
   (including AI agent tools) will be blocked.
```

**Example 2 — protect all credentials in your home directory:**

```
You: Please protect the credential files in ~

Agent: I scanned ~ and found these credential files:
  - ~/.ssh/id_rsa          → authorize /usr/bin/ssh
  - ~/.aws/credentials     → authorize /usr/bin/aws
  - ~/.aliyun/config.json  → authorize /usr/bin/aliyun (writable, token refresh)
Protect all of them? (y/n)
You: y
Agent: ✅ Protected 3 files. Each verified; authorized programs can access normally.
```

That's it. To release or modify a protected file later, just ask: "Release the protection on ~/.ssh/id_rsa" or "Allow /usr/bin/python3 to read ~/.aws/credentials".

## How it works

1. **Identify** the credential file to protect.
2. **Find** the authorized program that should access it (`which ssh` → `/usr/bin/ssh`).
3. **Protect** — `credctl protect` moves the content into the enclave and replaces the original with a symlink.
4. **Verify** the symlink and confirm the authorized program still works.
5. **Audit** the program's directory tree for shadowing / world-writable risks (see [Security Risk Check](SKILL.md)).

## Documentation

- [Installation guide](references/installation.md) — install CredAgent before using the skill
- [Skill instructions](SKILL.md) — the workflow the agent follows
- [Command reference](references/manual.md)
- [Troubleshooting](references/troubleshooting.md)

## License

Apache-2.0
