# credagent-skill

A skill that lets an AI agent protect your credential files (SSH keys, API keys, OAuth cookies, `.env` secrets, etc.) — using **CredAgent** from [aliyun/credagent](https://github.com/aliyun/credagent), which brings credential files under management in an encrypted virtual filesystem, where a program is permitted to access a credential only after its measurement is verified as legitimate.

Just tell your agent in plain language what to protect. The agent follows this skill to find the credential, identify the program that should access it, run `credctl` to move the secret into a FUSE enclave, and verify the result. The original file becomes a symlink — only explicitly authorized programs (verified by path and optional SHA256 measurement) can read it, so AI agents, rogue scripts, and unapproved tools are blocked from exfiltrating keys and tokens.

> This repository ships the **skill** (instructions + documentation that guide an agent). The credential-protection software itself is [CredAgent](https://github.com/aliyun/credagent) — the agent installs and operates it for you; you don't need to install it manually.

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
    ├── installation.md      # How the agent installs CredAgent + FUSE prerequisites
    ├── manual.md            # Full credctl / credreq command reference
    └── troubleshooting.md   # Permission-denied diagnostics & fixes
```

## Quick start

You don't run any commands yourself — just talk to your agent. The skill guides the agent through the whole flow.

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

- [Skill instructions](SKILL.md) — the workflow the agent follows
- [Installation guide](references/installation.md)
- [Command reference](references/manual.md)
- [Troubleshooting](references/troubleshooting.md)

## License

Apache-2.0
