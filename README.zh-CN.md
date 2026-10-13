# credagent-skill

[English](README.md) | **简体中文**

一个 Skill,让 AI Agent 帮你保护凭证文件(如SSH 密钥、 API Key、 OAuth Cookie、`.env` 密钥等)——使用来自 [aliyun/credagent](https://github.com/aliyun/credagent) 的 **CredAgent**把凭据文件纳管到加密虚拟文件系统中，凡访问凭据的程序具备度量合法后才许可访问。

你只需用自然语言告诉 Agent 要保护什么。Agent 遵循本 Skill 找到凭据、识别应访问它的程序、运行 `credctl` 把密钥移入 FUSE 安全区并验证结果。原文件变成符号链接——只有被明确授权的程序(通过路径校验,可选 SHA256 度量)才能读取,因此 AI Agent、恶意脚本和未授权工具都无法窃取密钥与 Token。

> 本仓库提供的是 **Skill**(引导 Agent 的指令与文档)。凭据保护软件本身是 [CredAgent](https://github.com/aliyun/credagent)——需要你先在机器上完成一次性安装(见[安装 CredAgent](#安装-credagent一次性))。装好之后剩下的都交给 Agent:它负责找到凭据、选定授权程序、执行 `credctl` 并验证结果。

## 安装本 Skill

```bash
npx skills add Benson-9527/credagent-skill
```

一条命令、无需编译——它会把本仓库的 `SKILL.md` 和 `references/` 拷进你 Agent 的 skills 目录。适用于任何读取 Agent Skills `SKILL.md` 格式的 Agent（Claude Code、Qoder、Codex CLI、OpenClaw 等）。习惯手动的话，直接把这个文件夹放进 Agent 的 skills 目录也可以。

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
    ├── installation.md      # CredAgent + FUSE 的安装、初始化与验证步骤
    ├── manual.md            # credctl / credreq 完整命令参考
    └── troubleshooting.md  # 权限拒绝诊断与修复
```

## 安装 CredAgent（一次性）

Skill 是通过 `credctl` 操作正在运行的 `credagent` 守护进程，所以必须先在机器上装好 CredAgent。这一步需要 root 权限和 FUSE——Agent 可以帮你执行命令，但需要你授权放行。下面两种方式任选其一，结果一致。

**方式一：下载预编译二进制（最快）**

```bash
# Linux x86_64
curl -LO https://github.com/aliyun/credagent/releases/download/v0.2.0/credagent_0.2.0_linux_amd64.tar.gz
tar -xzf credagent_0.2.0_linux_amd64.tar.gz
cd credagent_0.2.0_linux_amd64
sudo ./INSTALL
```

macOS 使用 `credagent_0.2.0_darwin_universal.tar.gz`，同样是 `./INSTALL`。脚本会安装 FUSE 依赖、把三个二进制拷到 `/usr/local/bin/`、把配置模板放到 `/usr/local/etc/credagent/`，并注册 systemd（Linux）或 launchd（macOS）服务。

**方式二：源码编译安装**（Go 1.21+）

```bash
curl -LO https://github.com/aliyun/credagent/archive/refs/tags/v0.2.0.tar.gz
tar -xzf v0.2.0.tar.gz
cd credagent-0.2.0

CGO_ENABLED=0 go build -o credagent ./cmd/credagent
CGO_ENABLED=0 go build -o credctl   ./cmd/credctl
CGO_ENABLED=0 go build -o credreq   ./cmd/credreq

sudo ./INSTALL
```

macOS 上请用 `CGO_ENABLED=1` 并在本机编译——进程身份校验依赖 `libproc`/`sysctl` 的 cgo 调用，无法从 Linux 交叉编译。

**然后初始化并启动守护进程**（生成主密钥、挂载 FUSE 安全区）：

```bash
sudo /usr/local/bin/credagent run \
  --mount /mnt/credagent/shielded/ \
  --socket-dir /run/credagent \
  --baseline-file /var/lib/credagent/baseline.enc \
  --config /usr/local/etc/credagent/engine/config.json \
  --auto-init
# 等到日志出现 "CredAgent is ready and serving requests"，再按 Ctrl-C

sudo systemctl start credagent && sudo systemctl enable credagent

# --auto-init 生成的是随机管理密码，使用 release / remove / update /
# attach / detach / refresh 之前必须先改掉
sudo /usr/local/bin/credagent init --config /usr/local/etc/credagent/engine/config.json
sudo systemctl restart credagent
```

用 `credctl version` 和 `ls -la /mnt/credagent/shielded/` 确认安装成功。完整细节——FUSE 前置依赖、macOS launchd 配置、验证清单、安装排错——见 [`references/installation.md`](references/installation.md)。

## 快速上手

装好 CredAgent 之后,你不需要自己敲任何命令——直接和 Agent 对话即可,本 Skill 会引导 Agent 走完整流程。

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

- [安装指南](references/installation.md) —— 使用 Skill 前先装好 CredAgent
- [Skill 指令](SKILL.md) —— Agent 遵循的工作流程
- [命令参考](references/manual.md)
- [故障排查](references/troubleshooting.md)

## 许可证

Apache-2.0
