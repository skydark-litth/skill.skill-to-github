# skill-to-github

把一个 **WorkBuddy 用户级 skill 目录**同步（发布 / 更新 / 备份）到 **GitHub 仓库**的技能。使用 git over SSH，二进制安全、支持任意文件大小、可一次提交多个文件。

## 适用场景

当你想把本地某个 skill（通常在 `C:\Users\USER\.workbuddy\skills\<skill-name>\`）：

- 发布 / 上传到 GitHub；
- 更新已同步的仓库（本地改动后对齐远程）；
- 做版本控制或云端备份。

## 核心特性

- **双目标显式确认**：必须由用户明确指定「本地 skill 完整路径」与「`owner/repo`」，并先核验本地 `SKILL.md` 名称，避免读错技能或覆盖错误项目。
- **仓库缺失时引导网页创建**：探测到目标仓库不存在，会给出 https://github.com/new 与精确的名称 / owner / 可见性，用户在网页创建后再继续（SSH 密钥本身不能创建远程仓库，此方式无需额外 token）。
- **README 检查与自动生成**：每次上传 / 更新前检查 `README.md` 是否存在、是否仍匹配当前技能；缺失或过期时经用户同意，由大模型总结技能的要求与功能自动生成 / 更新。
- **git 内置验证**：用 `git status` / `git diff` + `git fsck --full` 校验变更与对象完整性，以推送结果（ref 前进、exit 0）确认，无需再把文件下载回本地比对。
- **自我更新**：遇到技能尚未覆盖的问题，先解决，再把「症状 → 原因 → 处置」补进技能，使下次不再重踩。

## 工作流概览

| 阶段 | 步骤 | 说明 |
|---|---|---|
| 确认目标 | Step 0 | 显式指定本地 skill 路径 + `owner/repo`；核验本地 `SKILL.md` |
| 解析仓库 | Step 1 | `git ls-remote` 探测；不存在则引导网页创建；存在则克隆 |
| 检查 README | Step 2 | 缺失 / 过期则询问是否大模型自动生成 |
| 覆盖内容 | Step 3 | `cp -r "<skill-dir>/." "<workdir>/"`（保留仓库独有文件） |
| git 内置验证 | Step 4 | `git status` / `git diff` / `git fsck --full` |
| 提交推送 | Step 5 | `git add -A` → `commit` → `push`；以推送结果确认 |
| 自我更新 | Step 6 | 记录本次新遇到问题的解决办法 |

## 前置条件（一次性环境）

需要一台已安装 Git for Windows、且能通过 SSH 推送到 GitHub 的机器，验证：

```
ssh -T git@github.com
```

首次配置（用户级免管理员安装、PATH、git 全局、SSH 密钥、GitHub 绑定公钥、提交身份）见仓库内 [`references/setup.md`](./references/setup.md)。

## 安全说明

- 优先使用 SSH，无需在命令或文件中存放任何 token；私钥只保留在本机。
- 不把 `.ssh`、任何凭证文件或技能的本地运行产物纳入版本控制。
- 远程的创建、目标的选择均由用户显式确认，避免误覆盖。

## 版本

- 当前版本：**v2.0.0**（详见 [`SKILL.md`](./SKILL.md) 顶部 `version` 字段）。
