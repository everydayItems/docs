# Git 发布流程规范

> 适用于多环境 Kubernetes 部署的标准化发布流程

## 一、分支策略（Git Flow）

### 核心分支

| 分支名 | 生命周期 | 保护级别 | 说明 |
|--------|----------|----------|------|
| `main` | 永久 | 最高保护 | 生产环境代码，只能通过 PR 合并，必须打 tag |
| `develop` | 永久 | 高保护 | 开发主线，集成所有 feature，对应 Test 环境 |

### 临时分支

| 分支前缀 | 命名规范 | 生命周期 | 说明 |
|----------|----------|----------|------|
| `feature/` | `feature/{JIRA-ID}-{brief-description}` | 短期 | 新功能开发，从 develop 拉取，合并回 develop |
| `bugfix/` | `bugfix/{JIRA-ID}-{brief-description}` | 短期 | 非紧急 Bug 修复，从 develop 拉取 |
| `hotfix/` | `hotfix/{JIRA-ID}-{brief-description}` | 短期 | 生产紧急修复，从 main 拉取，合并回 main 和 develop |
| `release/` | `release/v{major}.{minor}.{patch}` | 短期 | 发布准备分支，从 develop 拉取，仅做版本号和 bug 修复 |

### 分支命名示例

```bash
# 功能分支
feature/PROJ-1234-add-new-feature
feature/PROJ-5678-async-queue

# 修复分支
bugfix/PROJ-2345-fix-data-validation
hotfix/PROJ-9999-fix-memory-leak

# 发布分支
release/v2.5.0
```

---

## 二、环境与分支映射

### 环境架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   开发环境   │────▶│  预发布环境  │────▶│   生产环境   │
│   (Local)   │     │    (Test)   │     │   (Prod)    │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                    │
   feature/*          develop               main
```

| 环境 | 分支 | 部署方式 | 域名示例 | 说明 |
|------|------|----------|----------|------|
| **本地开发** | `feature/*` | 手动运行 | `localhost:8000` | 开发者本地环境 |
| **Test** | `develop` | 自动部署（CD） | `staging-api.example.com` | 预发布环境，自动部署最新 develop 代码 |
| **Prod** | `main` | 手动触发（审批） | `api.example.com` | 生产环境，需审批和灰度发布 |

---

## 三、完整发布流程

### 3.1 功能开发（Feature Development）

```bash
# 1. 从最新 develop 创建 feature 分支
git checkout develop
git pull origin develop
git checkout -b feature/PROJ-1234-add-new-feature

# 2. 开发过程中定期同步 develop（避免冲突）
git fetch origin develop
git rebase origin/develop  # 或 git merge origin/develop

# 3. 提交代码（遵循 Conventional Commits）
git add .
git commit -m "feat(export): add user data export feature

- Implement data export service
- Add request/response mapping
- Add unit tests

JIRA: PROJ-1234"

# 4. 推送到远程
git push origin feature/PROJ-1234-add-new-feature
```

### 3.2 代码评审（Code Review）

```bash
# 1. 创建 Pull Request（GitHub/GitLab）
- Base: develop
- Compare: feature/PROJ-1234-add-new-feature
- 标题: feat(export): add user export [PROJ-1234]
- 描述: 包含变更摘要、测试计划、截图

# 2. PR 检查清单（自动化 CI）
✅ 单元测试通过 (go test ./... -v)
✅ 代码格式检查 (go fmt, golangci-lint)
✅ 代码覆盖率 ≥ 80%
✅ 无安全漏洞 (gosec)
✅ Docker 镜像构建成功

# 3. Code Review 要求
- 至少 2 名 Reviewer 批准（其中 1 名必须是 Senior）
- 无未解决的 Comment
- CI 流水线全绿

# 4. 合并到 develop
git checkout develop
git merge --no-ff feature/PROJ-1234-add-new-feature
git push origin develop

# 5. 删除 feature 分支
git branch -d feature/PROJ-1234-add-new-feature
git push origin --delete feature/PROJ-1234-add-new-feature
```

### 3.3 Test 测试（预发布）

```bash
# 1. develop 合并后自动触发 Test 部署（CD Pipeline）
# Jenkins/GitLab CI 自动执行：
- 构建 Docker 镜像（tag: develop-{git-sha})
- 推送到镜像仓库
- 更新 k8s/overlays/staging/ 配置
- kubectl apply -k k8s/overlays/staging
- 运行集成测试（Postman/Newman）

# 2. QA 测试验证
- 功能测试（基于测试用例）
- 回归测试（核心业务流程）
- 性能测试（压测关键接口）

# 3. 测试通过后，进入发布准备
```

### 3.4 创建 Release 分支（发布准备）

```bash
# 1. 从 develop 创建 release 分支
git checkout develop
git pull origin develop
git checkout -b release/v2.5.0

# 2. 更新版本号和 CHANGELOG
# - 修改 version.go 或 package.json
# - 更新 CHANGELOG.md
git commit -m "chore(release): bump version to v2.5.0"

# 3. 仅允许 bugfix 提交（禁止新功能）
# 如发现 bug，直接在 release 分支修复：
git commit -m "fix(validation): correct data validation in release

JIRA: PROJ-5678"

# 4. 推送 release 分支
git push origin release/v2.5.0
```

### 3.5 生产发布（Prod Release）

```bash
# 1. 创建 PR: release/v2.5.0 → main
- Base: main
- Compare: release/v2.5.0
- 标题: [Release] v2.5.0
- 描述: 包含完整 CHANGELOG、回滚计划、监控指标

# 2. 发布前检查清单
✅ Test 环境验证通过
✅ 数据库迁移脚本已测试（sql/migrations/）
✅ 配置文件已同步（k8s/overlays/production/）
✅ 回滚方案已准备（上一版本镜像保留）
✅ 监控告警已配置
✅ 值班人员已就位

# 3. 合并到 main（需要 2 名 Tech Lead 批准）
git checkout main
git pull origin main
git merge --no-ff release/v2.5.0
git push origin main

# 4. 打 Tag（语义化版本）
git tag -a v2.5.0 -m "Release v2.5.0

## 新功能
- [PROJ-1234] 新增用户数据导出功能
- [PROJ-2345] 异步队列性能优化

## Bug 修复
- [PROJ-5678] 修复数据校验错误

## 性能优化
- 优化数据库查询，减少 50% MySQL 读取

## 升级说明
- 需执行 sql/migrations/20260121_add_tables.sql
- 环境变量新增: ASYNQ_POLLING_ENABLED=true
"
git push origin v2.5.0

# 5. 合并回 develop（保持同步）
git checkout develop
git merge --no-ff main
git push origin develop

# 6. 删除 release 分支
git branch -d release/v2.5.0
git push origin --delete release/v2.5.0
```

### 3.6 生产部署（灰度发布）

```bash
# 1. 手动触发生产部署（Jenkins Pipeline）
# 输入参数：
- Tag: v2.5.0
- 灰度策略: Canary (10% → 25% → 50% → 100%)
- 回滚阈值: 错误率 > 1% OR P99 延迟 > 2s

# 2. Canary 发布流程（Kubernetes）- 假设生产环境有 10 个 Pod
# 阶段 1: 部署 1 个新版本 Pod（10% 流量）
kubectl set image deployment/app-server \
  app-server=registry.example.com/app-server:v2.5.0 \
  -n production
kubectl scale deployment/app-server --replicas=1 -n production

# 观察 10 分钟，监控指标：
- 错误率（Prometheus: http_request_errors_total）
- 延迟（P50/P95/P99）
- CPU/内存使用率
- 消息队列堆积

# 阶段 2: 扩展到 25% 流量（2-3 个 Pod）
kubectl scale deployment/app-server --replicas=3 -n production

# 观察 15 分钟

# 阶段 3: 扩展到 50% 流量（5 个 Pod）
kubectl scale deployment/app-server --replicas=5 -n production

# 观察 20 分钟

# 阶段 4: 全量发布（100%，10 个 Pod）
kubectl scale deployment/app-server --replicas=10 -n production

# 3. 数据库迁移（在发布前或 Canary 第一阶段）
mysql -h prod-db.example.com -u admin -p myapp_db < sql/migrations/20260121_add_tables.sql

# 4. 配置更新（ConfigMap/Secret）
kubectl apply -f k8s/overlays/production/configmap-env.yaml
kubectl rollout restart deployment/app-server -n production

# 5. 发布后验证
- 冒烟测试（关键业务流程）
- 检查日志（无严重错误）
- 检查监控（指标正常）
- 检查队列（消息队列正常消费）
```

---

## 四、热修复流程（Hotfix）

### 4.1 紧急修复场景

```bash
# 1. 从 main 创建 hotfix 分支
git checkout main
git pull origin main
git checkout -b hotfix/PROJ-9999-fix-critical-bug

# 2. 快速修复 Bug
git commit -m "fix(critical): fix memory leak in task polling

- Add context timeout to prevent goroutine leak
- Close HTTP connections properly

JIRA: PROJ-9999"

# 3. 推送并创建紧急 PR
git push origin hotfix/PROJ-9999-fix-critical-bug

# PR 目标分支: main
# 审批流程简化: 1 名 Tech Lead 即可（紧急情况）
```

### 4.2 紧急发布

```bash
# 1. 合并到 main
git checkout main
git merge --no-ff hotfix/PROJ-9999-fix-critical-bug
git push origin main

# 2. 打 Patch 版本 Tag
git tag -a v2.5.1 -m "Hotfix v2.5.1: Fix critical memory leak

JIRA: PROJ-9999"
git push origin v2.5.1

# 3. 立即部署到生产（跳过灰度，全量发布）
kubectl set image deployment/app-server \
  app-server=registry.example.com/app-server:v2.5.1 \
  -n production

# 4. 合并回 develop（防止下次发布丢失修复）
git checkout develop
git merge --no-ff hotfix/PROJ-9999-fix-critical-bug
git push origin develop

# 5. 删除 hotfix 分支
git branch -d hotfix/PROJ-9999-fix-critical-bug
git push origin --delete hotfix/PROJ-9999-fix-critical-bug
```

---

## 五、回滚策略

### 5.1 快速回滚（Kubernetes）

```bash
# 方法 1: 回滚到上一个版本（最快）
kubectl rollout undo deployment/app-server -n production

# 方法 2: 回滚到指定版本
kubectl rollout history deployment/app-server -n production
kubectl rollout undo deployment/app-server --to-revision=5 -n production

# 方法 3: 重新部署旧版本 Tag
kubectl set image deployment/app-server \
  app-server=registry.example.com/app-server:v2.4.9 \
  -n production
```

### 5.2 数据库回滚

```bash
# 1. 如果执行了数据库迁移，需要回滚 SQL
mysql -h prod-db.example.com -u admin -p myapp_db < sql/rollback/20260121_rollback_tables.sql

# 2. 数据迁移规范
# 每个 migration 文件必须有对应的 rollback 文件：
sql/migrations/20260121_add_column.sql
sql/rollback/20260121_rollback_add_column.sql
```

### 5.3 回滚决策标准

| 指标 | 阈值 | 动作 |
|------|------|------|
| 错误率 | > 1% | 立即回滚 |
| P99 延迟 | > 2 倍基线 | 观察 5 分钟，无改善则回滚 |
| 核心功能不可用 | 任何核心流程失败 | 立即回滚 |
| 数据一致性问题 | 发现数据损坏 | 立即回滚 + 数据修复 |

---

## 六、提交信息规范（Conventional Commits）

### 6.1 提交格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 6.2 Type 类型

| Type | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(export): add data export` |
| `fix` | Bug 修复 | `fix(validation): correct data check` |
| `perf` | 性能优化 | `perf(cache): reduce MySQL queries by 50%` |
| `refactor` | 重构 | `refactor(relay): extract common logic` |
| `docs` | 文档更新 | `docs(readme): update deployment guide` |
| `test` | 测试相关 | `test(task): add unit tests for polling` |
| `chore` | 构建/工具 | `chore(deps): upgrade gin to v1.10.0` |
| `style` | 代码格式 | `style: run go fmt` |
| `ci` | CI/CD 配置 | `ci(jenkins): add staging pipeline` |
| `revert` | 回退提交 | `revert: revert feat(export)` |

### 6.3 Scope 范围

常用 scope（根据项目模块）：
- `export` - 数据导出模块
- `relay` - 转发层
- `task` - 任务系统
- `validation` - 数据校验
- `cache` - 缓存层
- `queue` - 消息队列
- `k8s` - Kubernetes 配置

### 6.4 完整示例

```bash
git commit -m "feat(task): add async queue processing

- Replace periodic polling with message queue
- Reduce MySQL read operations by 80%
- Add payload-based polling without DB lookup
- Keep legacy polling as fallback

Breaking Change: Requires ASYNQ_POLLING_ENABLED=true
JIRA: PROJ-8888

Co-Authored-By: Developer <dev@example.com>"
```

---

## 七、常见问题（FAQ）

### Q1: 如何在 develop 分支上快速验证某个 feature？

```bash
# 1. 临时合并 feature 到 develop（不推送）
git checkout develop
git merge --no-commit --no-ff feature/PROJ-1234-test

# 2. 本地测试验证
go run main.go

# 3. 如果不满意，撤销合并
git merge --abort

# 4. 如果满意，正式合并
git commit -m "Merge feature/PROJ-1234-test into develop"
git push origin develop
```

### Q2: 如何处理多个 feature 并行开发的冲突？

```bash
# 建议使用 Rebase 工作流保持提交历史整洁
git checkout feature/PROJ-1234-my-feature
git fetch origin develop
git rebase origin/develop  # 而不是 merge

# 解决冲突后
git add .
git rebase --continue
git push --force-with-lease origin feature/PROJ-1234-my-feature
```

### Q3: 发布过程中发现严重 Bug，如何快速回滚？

```bash
# 方法 1: Kubernetes 快速回滚（推荐）
kubectl rollout undo deployment/app-server -n production

# 方法 2: 重新部署上一个稳定版本
kubectl set image deployment/app-server \
  app-server=registry.example.com/app-server:v2.4.9 \
  -n production

# 方法 3: Git 回滚（彻底方案）
git revert v2.5.0  # 创建一个新的 revert commit
git tag -a v2.5.1 -m "Revert v2.5.0 due to critical bug"
git push origin main --tags
```

### Q4: 如何处理遗留的 feature 分支（已过期但未合并）？

```bash
# 1. 检查哪些分支已过期（超过 30 天无更新）
git for-each-ref --sort=-committerdate refs/remotes/origin/ \
  --format='%(committerdate:short) %(refname:short)' | grep feature

# 2. 同步最新 develop
git checkout feature/old-feature
git fetch origin develop
git rebase origin/develop

# 3. 如果冲突太多或功能已废弃，直接删除
git branch -D feature/old-feature
git push origin --delete feature/old-feature
```

### Q5: 如何在生产环境快速验证某个功能是否正常？

```bash
# 1. 使用生产环境的健康检查接口
curl https://api.example.com/health

# 2. 调用关键业务接口（带测试 Token）
curl -X POST https://api.example.com/api/task/submit \
  -H "Authorization: Bearer test-token" \
  -d '{"type":"export","format":"csv"}'

# 3. 检查消息队列状态
curl https://api.example.com/api/v1/queue/stats

# 4. 查看实时日志（Kubernetes）
kubectl logs -f deployment/app-server -n production --tail=100
```

---

## 八、最佳实践

### 8.1 代码提交

**推荐做法**
- 每个 commit 只做一件事（原子性）
- 提交信息遵循 Conventional Commits
- 提交前运行 `go fmt` 和 `go test`
- feature 分支定期 rebase develop（保持同步）

**避免做法**
- 直接在 develop/main 分支提交
- 提交信息含糊不清（如 "fix bug"）
- 一个 commit 包含多个不相关改动
- 提交未测试的代码

### 8.2 分支管理

**推荐做法**
- feature 分支及时合并（< 7 天）
- 合并后立即删除 feature 分支
- 使用 `--no-ff` 保留分支历史
- 定期清理远程已删除分支

**避免做法**
- feature 分支长期存在（> 2 周）
- 多个 feature 相互依赖
- 分支命名不规范（如 "test", "tmp"）

### 8.3 发布管理

**推荐做法**
- 每次发布都打 Tag（语义化版本）
- 灰度发布，分阶段观察
- 保留至少 3 个历史版本镜像
- 发布窗口选择低峰期（凌晨/周末）

**避免做法**
- 跳过 Test 验证直接上生产
- 发布后立即下班（无人值守）
- 多个大功能同时发布（风险叠加）

---

## 附录

### A. Git 命令速查

```bash
# 查看分支状态
git status
git branch -a

# 创建并切换分支
git checkout -b feature/new-feature

# 同步远程分支
git fetch origin
git pull origin develop

# 合并分支（保留历史）
git merge --no-ff feature/new-feature

# 变基（整理提交历史）
git rebase origin/develop

# 查看提交历史
git log --oneline --graph --all

# 撤销本地修改
git checkout -- file.go
git reset --hard HEAD

# 修改最后一次提交
git commit --amend

# 推送 Tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# 删除远程分支
git push origin --delete feature/old-feature
```

### B. 语义化版本规范

```
版本格式: MAJOR.MINOR.PATCH (例如: 2.5.1)

- MAJOR: 不兼容的 API 修改（破坏性变更）
- MINOR: 向后兼容的功能新增
- PATCH: 向后兼容的 Bug 修复

示例:
v1.0.0 - 初始版本
v1.1.0 - 新增数据导出功能（新功能）
v1.1.1 - 修复数据校验 Bug（Bug 修复）
v2.0.0 - 重构 API 接口（破坏性变更）
```

### C. 相关文档

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)

---

**文档版本**: v1.0.0
**最后更新**: 2026-01-28
**维护者**: DevOps Team
