# credagent-skill

一个通用的 [Agent Skill](https://docs.claude.com/claude-code/skills),帮助用户使用 **CredAgent** —— 一个基于 FUSE 的零信任凭证保护系统 —— 锁定基于文件的凭证(SSH 密钥、云 CLI 配置、API Token、OAuth Cookie、`.env` 密钥等)。

本 Skill 遵循标准的 Agent Skill 格式(带 YAML frontmatter 的 `SKILL.md` 加上 `references/` 目录),可被任何支持该格式的 Agent 工具加载使用,不绑定特定厂商的 Agent。

> **说明:** 凭据保护功能由 [CredAgent](https://github.com/aliyun/credagent) 软件(阿里云开源项目)实现。本仓库仅提供 **Agent Skill** —— 即引导 Agent 操作 `credctl` / `credagent` / `credreq` 命令行工具的指令与文档。请先从其 [GitHub 仓库](https://github.com/aliyun/credagent) 安装 CredAgent 软件。

凭证被保护后,CredAgent 会将其替换为指向 FUSE 安全区的符号链接。只有被明确授权的程序(通过路径校验,可选 SHA256 度量)才能读取密钥 —— 因此 AI Agent、恶意脚本和未授权工具都无法窃取密钥与 Token。

## 特性

- **文件级保护** —— 适用于任意凭证文件(SSH 密钥、`~/.aws/credentials`、`~/.aliyun/config.json`、`.env` 等)
- **零信任访问** —— 仅被明确授权的二进制/脚本可打开受保护文件
- **完整性度量** —— 通过 SHA256 校验授权程序
- **打开频率限制** —— 限制单个 PID 打开凭证的次数
- **可写模式** —— 适用于需要更新的凭证(OAuth Token、Cookie)
- **目录树风险审计** —— 发现程序目录树中的导入影子化与世界可写漏洞
- **完整生命周期** —— 保护、查看、更新、释放与删除

## 仓库结构

```
.
├── SKILL.md                     # Agent 加载的 Skill 指令
└── references/
    ├── installation.md          # 安装 CredAgent + FUSE 前置依赖
    ├── manual.md                # credctl / credreq 完整命令参考
    └── troubleshooting.md       # 权限拒绝诊断与修复
```

## 安装

先从 [GitHub 仓库](https://github.com/aliyun/credagent) 安装 CredAgent 软件,再将本 Skill 加入你的 Agent 工具。参照 [`references/installation.md`](references/installation.md) 部署 CredAgent 守护进程(`credagent`)、CLI 工具(`credctl`)以及 `credreq`。

## 快速上手

```bash
# 保护 SSH 密钥,仅允许 /usr/bin/ssh 访问(度量、每个 PID 最多打开 2 次)
credctl protect --strategy BASIC \
  --program-permit /usr/bin/ssh \
  --measure true \
  --open-frequency 2 \
  ~/.ssh/id_rsa

# 列出所有受保护文件
credctl list --json

# 释放(还原)受保护文件
credctl release <id>
```

完整命令集见 [快速参考](SKILL.md#quick-reference) 与 [`references/manual.md`](references/manual.md)。

## 工作原理

1. **识别** 要保护的凭证文件。
2. **找到** 应访问它的授权程序(`which ssh` → `/usr/bin/ssh`)。
3. **保护** —— 执行 `credctl protect`,文件内容移入安全区,原文件被符号链接替换。
4. **验证** 符号链接并测试授权程序是否正常工作。
5. **审计** 程序目录树,排查导入影子化 / 世界可写风险(见 [安全风险检查](SKILL.md))。

## 文档

- [Skill 指令](SKILL.md) —— Agent 遵循的工作流程
- [安装指南](references/installation.md)
- [命令参考](references/manual.md)
- [故障排查](references/troubleshooting.md)

## 许可证

Apache-2.0
