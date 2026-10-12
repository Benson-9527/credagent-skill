# credagent-skill

一个 Skill,让 AI Agent 帮你保护凭证文件(如SSH 密钥、 API Key、 OAuth Cookie、`.env` 密钥等)——使用来自 [aliyun/credagent](https://github.com/aliyun/credagent) 的 **CredAgent**把凭据文件纳管到加密虚拟文件系统中，凡访问凭据的程序具备度量合法后才许可访问。

你只需用自然语言告诉 Agent 要保护什么。Agent 遵循本 Skill 找到凭据、识别应访问它的程序、运行 `credctl` 把密钥移入 FUSE 安全区并验证结果。原文件变成符号链接——只有被明确授权的程序(通过路径校验,可选 SHA256 度量)才能读取,因此 AI Agent、恶意脚本和未授权工具都无法窃取密钥与 Token。

> 本仓库提供的是 **Skill**(引导 Agent 的指令与文档)。凭据保护软件本身是 [CredAgent](https://github.com/aliyun/credagent)——由 Agent 替你安装并操作,你无需手动安装。

## 特性

- **文件级保护** —— 适用于任意凭据文件(SSH 密钥、`~/.aws/credentials`、`~/.aliyun/config.json`、`.env` 等)
- **零信任访问** —— 仅被明确授权的二进制/脚本可打开受保护文件
- **完整性度量** —— 通过 SHA256 校验授权程序
- **打开频率限制** —— 限制单个 PID 打开凭证的次数
- **可写模式** —— 适用于需要更新的凭证(OAuth Token、Cookie)
- **目录树风险审计** —— 发现程序目录树中的导入影子化与世界可写漏洞
- **完整生命周期** —— 保护、查看、更新、释放与删除

## 仓库结构

```
.
├── SKILL.md                 # Agent 遵循的 Skill 指令
└── references/
    ├── installation.md      # Agent 如何安装 CredAgent + FUSE 前置依赖
    ├── manual.md            # credctl / credreq 完整命令参考
    └── troubleshooting.md  # 权限拒绝诊断与修复
```

## 快速上手

你不需要自己敲任何命令——直接和 Agent 对话即可,本 Skill 会引导 Agent 走完整流程。

**示例 1 —— 保护 SSH 密钥:**

```
You: 请帮我保护 ~/.ssh/id_rsa

Agent: 我来帮你保护 SSH 私钥。
  1. 凭据文件:~/.ssh/id_rsa
  2. 授权程序:/usr/bin/ssh(git over ssh 会调用它)
  3. 保护(度量 SHA256,每个 PID 最多打开 2 次):
     $ credctl protect --strategy BASIC \
         --program-permit /usr/bin/ssh --measure true \
         --open-frequency 2 ~/.ssh/id_rsa
     Successfully onboarded ~/.ssh/id_rsa, securely shredded, and
     created symlink to /mnt/credagent/shielded/a23457c0659d.
  4. 验证:
     $ ls -la ~/.ssh/id_rsa
     lrwxrwxrwx ... ~/.ssh/id_rsa -> /mnt/credagent/shielded/a23457c0659d
✅ 完成。只有 /usr/bin/ssh 能读取该私钥;其他程序(含 AI agent 工具)将被拦截。
```

**示例 2 —— 保护 ~ 目录下的所有凭据:**

```
You: 请保护 ~ 目录中的凭据文件

Agent: 我扫描了 ~ 目录,发现以下凭据文件:
  - ~/.ssh/id_rsa          → 授权 /usr/bin/ssh
  - ~/.aws/credentials     → 授权 /usr/bin/aws
  - ~/.aliyun/config.json  → 授权 /usr/bin/aliyun(可写,需刷新 token)
是否全部保护?(y/n)
You: y
Agent: ✅ 已保护 3 个文件。逐项校验通过,授权程序均可正常访问。
```

就这么简单。之后要释放或修改受保护文件,直接开口即可:"释放 ~/.ssh/id_rsa 的保护",或"给 ~/.aws/credentials 增加允许的程序 /usr/bin/python3"。

## 工作原理

1. **识别** 要保护的凭据文件。
2. **找到** 应访问它的授权程序(`which ssh` → `/usr/bin/ssh`)。
3. **保护** —— `credctl protect` 把内容移入安全区,原文件被符号链接替换。
4. **验证** 符号链接并确认授权程序仍能正常工作。
5. **审计** 程序目录树,排查导入影子化 / 世界可写风险(见 [安全风险检查](SKILL.md))。

## 文档

- [Skill 指令](SKILL.md) —— Agent 遵循的工作流程
- [安装指南](references/installation.md)
- [命令参考](references/manual.md)
- [故障排查](references/troubleshooting.md)

## 许可证

Apache-2.0
