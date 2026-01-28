# Git SSH 多账号配置指南

> 适用于 macOS / Linux，解决 GitHub 多账号 SSH 认证冲突问题

## 适用场景

- macOS / Linux 环境
- 使用 SourceTree 或 Terminal 管理 Git 仓库
- GitHub 私有仓库
- 多个 GitHub 账号（如 work / personal）
- 全部使用 SSH 认证

## 目标

- 不串号
- 不再出现 `Repository not found` / `Could not read from remote repository`
- 一次配置，长期稳定

---

## 一、问题根因

在 GitHub + SSH + 多账号环境中，GitHub 通过 SSH Key 识别用户身份。

**问题场景**：
- 一台机器上加载了多个 SSH Key
- 所有仓库都使用 `git@github.com:xxx/yyy.git`

**结果**：
- GitHub 随机/按顺序使用某个 key
- 一旦使用了「没有该私有库权限的 key」

**报错信息**：
```
ERROR: Repository not found.
fatal: Could not read from remote repository.
```

> 实际是**权限不足**，不是仓库不存在

---

## 二、解决方案

**核心思想**：给每个 GitHub 账号一个独立的 SSH Host 别名，让不同仓库强制绑定不同的 SSH Key。

---

## 三、配置步骤

### 步骤 1：为每个账号准备独立 SSH Key

**查看已有 key**：
```bash
ls -la ~/.ssh
```

**推荐命名**：
- 工作账号：`id_ed25519_github_work`
- 个人账号：`id_ed25519_github_personal`

**新建 key（推荐 ed25519）**：
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_work -C "work@example.com"
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_personal -C "personal@example.com"
```

**生成文件**：
```
~/.ssh/id_ed25519_github_work
~/.ssh/id_ed25519_github_work.pub
~/.ssh/id_ed25519_github_personal
~/.ssh/id_ed25519_github_personal.pub
```

### 步骤 2：添加公钥到 GitHub

对每个 GitHub 账号分别操作：

1. GitHub → **Settings** → **SSH and GPG keys** → **New SSH key**
2. 填写：
   - **Title**：随意（如 MacBook-Work）
   - **Key**：粘贴对应的 `.pub` 文件内容
   - **Type**：Authentication Key

> ⚠️ 注意不要加错账号

### 步骤 3：配置 SSH Host 别名（关键）

编辑 SSH 配置文件：
```bash
nano ~/.ssh/config
```

**配置内容**：
```
# 工作账号
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_work
  IdentitiesOnly yes
  UseKeychain yes
  AddKeysToAgent yes

# 个人账号
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_personal
  IdentitiesOnly yes
  UseKeychain yes
  AddKeysToAgent yes
```

**设置权限**：
```bash
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_github_work ~/.ssh/id_ed25519_github_personal
chmod 644 ~/.ssh/*.pub
```

### 步骤 4：验证 SSH 连接

```bash
ssh -T git@github-work
ssh -T git@github-personal
```

**期望输出**：
```
Hi <github-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

> 两个命令应显示**不同的** GitHub 用户名

### 步骤 5：为仓库绑定对应 Host

**查看当前 remote**：
```bash
git remote -v
```

**修改 origin**：
```bash
# 工作仓库
git remote set-url origin git@github-work:ORG_OR_USER/REPO.git

# 个人仓库
git remote set-url origin git@github-personal:ORG_OR_USER/REPO.git
```

从此以后：
- 该仓库永远只会用指定账号的 SSH Key
- SourceTree / Terminal 都不会串号

---

## 四、SourceTree 注意事项

| 项目 | 说明 |
|------|------|
| **Remote URL** | 必须是 SSH 形式：`git@github-work:xxx/yyy.git` |
| **账号登录** | 仅用于 UI 展示（PR、仓库列表），不决定 git fetch 使用哪个 key |
| **真正生效** | `~/.ssh/config` 配置 |

---

## 五、快速排错

遇到 `403` / `Repository not found` 时：

```bash
# 1. 检查当前仓库 remote
git remote -v

# 2. 检查默认 SSH 身份
ssh -T git@github.com

# 3. 检查指定 Host 身份
ssh -T git@github-work
ssh -T git@github-personal
```

**如果 `ssh -T git@github.com` 显示的是另一个账号**：
- 说明发生了 SSH 身份串号
- 按本文方案配置 Host 并修改 remote 即可

---

## 六、总结

多 GitHub 账号 + SSH 的唯一稳定解法：

| 要点 | 说明 |
|------|------|
| 独立 Key | 每个账号一个 SSH Key |
| Host 别名 | 每个账号一个 SSH Host 别名 |
| 绑定仓库 | 每个仓库绑定对应 Host |

**优势**：
- ✅ 稳定
- ✅ 不串号
- ✅ SourceTree / Terminal 通用

---

**文档版本**: v1.0.0
**最后更新**: 2026-01-28
