# credagent-skill

A portable [Agent Skill](https://docs.claude.com/claude-code/skills) that helps users lock down file-based credentials (SSH keys, cloud CLI configs, API tokens, OAuth cookies, `.env` secrets) using **CredAgent** — a zero-trust, FUSE-based credential protection system.

This skill follows the standard Agent Skill format (a `SKILL.md` with YAML frontmatter plus a `references/` directory) and works with any agent tooling that supports that format — not specific to any one vendor's agent.

> **Note:** The credential protection itself is implemented by the [CredAgent](https://github.com/aliyun/credagent) software (an open-source project from Alibaba/Aliyun). This repository only provides the **Agent Skill** — the instructions and documentation that guide an agent to operate the `credctl` / `credagent` / `credreq` CLIs. Install the CredAgent software from its [GitHub repository](https://github.com/aliyun/credagent) first.

When a credential file is protected, CredAgent replaces it with a symlink into a FUSE enclave. Only explicitly authorized programs (verified by path and optional SHA256 measurement) can read the secret — so AI agents, rogue scripts, and unapproved tools are blocked from exfiltrating keys and tokens.

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
├── SKILL.md                     # Skill instructions loaded by the agent
└── references/
    ├── installation.md          # Install CredAgent + FUSE prerequisites
    ├── manual.md                # Full credctl / credreq command reference
    └── troubleshooting.md       # Permission-denied diagnostics & fixes
```

## Installation

Install the CredAgent software from its [GitHub repository](https://github.com/aliyun/credagent), then add this skill into your agent tooling. Follow [`references/installation.md`](references/installation.md) to set up the CredAgent daemon (`credagent`), the CLI (`credctl`), and `credreq`.

## Quick start

```bash
# Protect an SSH key, permitting only /usr/bin/ssh (measured, 2 opens per PID)
credctl protect --strategy BASIC \
  --program-permit /usr/bin/ssh \
  --measure true \
  --open-frequency 2 \
  ~/.ssh/id_rsa

# List all protected files
credctl list --json

# Release (restore) a protected file
credctl release <id>
```

See the [Quick Reference](SKILL.md#quick-reference) and [`references/manual.md`](references/manual.md) for the full command set.

## How it works

1. **Identify** the credential file to protect.
2. **Find** the authorized program that should access it (`which ssh` → `/usr/bin/ssh`).
3. **Protect** it with `credctl protect` — content moves into the enclave, original is replaced by a symlink.
4. **Verify** the symlink and test the authorized program.
5. **Audit** the program's directory tree for shadowing / world-writable risks (see [Security Risk Check](SKILL.md)).

## Documentation

- [Skill instructions](SKILL.md) — the workflow the agent follows
- [Installation guide](references/installation.md)
- [Command reference](references/manual.md)
- [Troubleshooting](references/troubleshooting.md)

## License

Apache-2.0
