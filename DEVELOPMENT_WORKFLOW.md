# 开发工作流规范

本文档定义了 MEV Arbitrage Bot 项目的标准开发工作流程，适用于所有编码 Agent 和开发会话。

## 工作流概述

```
检查版本 → 创建功能分支 → 编码开发 → 提交更改 → 创建PR → 合并到主分支 → 推送远端 → 报告状态
```

## 详细步骤

### 1. 开始开发前：版本检查

**目的**：确保在最新代码基础上开发，避免冲突

**操作**：

```powershell
# 切换到主分支
git checkout master

# 拉取最新远端代码
git pull origin master

# 验证当前位置
git log --oneline -1
git status

# 确认输出显示：
# - 在 master 分支
# - 工作区干净
# - 与远端同步
```

**检查点**：
- ✅ 当前分支是 `master`
- ✅ 工作区状态为 `working tree clean`
- ✅ 显示 `Your branch is up to date with 'origin/master'`

### 2. 创建功能分支

**命名规范**：

```
feature/<功能描述>     # 新功能
fix/<问题描述>         # Bug修复
refactor/<重构描述>    # 重构
docs/<文档描述>        # 文档更新
```

**操作**：

```powershell
# 创建并切换到新分支
git checkout -b feature/price-monitor

# 验证分支创建
git branch --show-current
```

**示例分支名**：
- `feature/pool-manager-impl`
- `feature/triangular-arbitrage`
- `fix/gas-estimation-error`
- `refactor/executor-module`

### 3. 执行编码开发

在功能分支上进行所有开发工作：

- 实现新功能
- 修复Bug
- 编写测试
- 更新文档

**期间可以多次提交**：

```powershell
# 阶段性提交
git add <相关文件>
git commit -m "WIP: implement price fetching logic"

# 或提交所有更改
git add .
git commit -m "feat: complete PriceMonitor implementation"
```

### 4. 开发完成：准备合并

**操作**：

```powershell
# 1. 确保所有更改已提交
git status  # 应显示 nothing to commit, working tree clean

# 2. 查看提交历史
git log --oneline master..HEAD

# 3. 切换回主分支
git checkout master

# 4. 再次拉取最新代码（防止开发期间主分支有更新）
git pull origin master

# 5. 合并功能分支到主分支
git merge feature/price-monitor --no-ff -m "Merge feature/price-monitor into master"

# --no-ff: 保留分支历史，创建合并提交
```

### 5. 推送到远端主分支

**重要**：推送到 `origin master`，而不是功能分支的远端

```powershell
# 推送主分支到远端
git push origin master
```

**禁止操作**：
- ❌ `git push origin feature/price-monitor` - 不要推送功能分支
- ❌ 直接在主分支开发
- ❌ 强制推送 `--force`

### 6. 创建 Pull Request（可选记录）

如果需要代码审查或保留变更记录：

```powershell
# 使用 GH CLI 创建 PR（从功能分支到master）
gh pr create --base master --head feature/price-monitor \
  --title "feat: Implement Price Monitor module" \
  --body "
## 变更内容
- 实现价格监听核心逻辑
- 添加多DEX价格抓取
- 完成单元测试

## 测试
- [x] 单元测试通过
- [x] 集成测试通过
- [x] 本地运行验证

## 相关文档
参考 docs/04-implementation-guide.md
"

# 自动合并（如果已推送）
gh pr merge --merge --delete-branch
```

### 7. 清理功能分支

```powershell
# 删除本地功能分支
git branch -d feature/price-monitor

# 如果之前推送过功能分支到远端，删除远端分支
git push origin --delete feature/price-monitor
```

### 8. 报告完成状态

**报告格式**：

```
✅ 开发任务完成

- 功能分支: feature/price-monitor
- 提交数量: 3
- 合并提交: abc1234
- 推送状态: ✅ 成功推送到 origin/master
- PR编号: #1 (如有)
- PR状态: ✅ 已合并 (如有)

主要变更:
- 实现 PriceMonitor 核心模块
- 添加 QuickSwap, Uniswap V3, SushiSwap 价格抓取
- 完成测试覆盖

远端仓库: https://github.com/unnoderes/mev-arbitrage-bot
```

## 完整示例

```powershell
# ===== 1. 检查版本 =====
git checkout master
git pull origin master
git log --oneline -1

# ===== 2. 创建功能分支 =====
git checkout -b feature/strategy-engine

# ===== 3. 编码开发 =====
# ... 编写代码 ...
git add src/core/Strategy.ts
git commit -m "feat: implement triangular arbitrage strategy"

# ... 继续开发 ...
git add tests/Strategy.test.ts
git commit -m "test: add strategy unit tests"

# ===== 4. 准备合并 =====
git checkout master
git pull origin master
git merge feature/strategy-engine --no-ff

# ===== 5. 推送主分支 =====
git push origin master

# ===== 6. 创建PR（可选）=====
gh pr create --base master --head feature/strategy-engine \
  --title "feat: Implement Strategy Engine" \
  --body "实现套利策略引擎核心逻辑"
  
# 如果需要立即合并PR
gh pr merge --merge --delete-branch

# ===== 7. 清理 =====
git branch -d feature/strategy-engine

# ===== 8. 报告 =====
# 输出任务完成报告
```

## Worktree 工作流（高级）

如果使用 Git Worktree 进行隔离开发：

```powershell
# 1. 创建新的worktree和功能分支
git worktree add ../mev-bot-feature-poolmanager -b feature/pool-manager

# 2. 进入worktree
cd ../mev-bot-feature-poolmanager

# 3. 开发和提交
git add .
git commit -m "feat: implement PoolManager"

# 4. 回到主worktree
cd C:\Users\Administrator\Desktop\mev-arbitrage-bot

# 5. 合并功能分支
git checkout master
git pull origin master
git merge feature/pool-manager --no-ff

# 6. 推送
git push origin master

# 7. 清理worktree
git worktree remove ../mev-bot-feature-poolmanager
git branch -d feature/pool-manager
```

## 提交信息规范

遵循 Conventional Commits 格式：

```
<type>(<scope>): <subject>

<body>

<footer>
```

**类型（type）**：
- `feat`: 新功能
- `fix`: Bug修复
- `docs`: 文档更新
- `style`: 代码格式（不影响功能）
- `refactor`: 重构
- `test`: 测试相关
- `chore`: 构建/工具配置

**示例**：

```
feat(core): implement PriceMonitor module

- Add price fetching from QuickSwap
- Add price fetching from Uniswap V3
- Implement price difference calculation
- Add error handling and retry logic

Closes #5
```

## 常见问题

### Q: 功能分支开发期间，主分支有新提交怎么办？

```powershell
# 在功能分支上
git checkout feature/my-feature

# 拉取主分支最新代码
git fetch origin master

# 变基到最新主分支
git rebase origin/master

# 解决冲突后继续
git add .
git rebase --continue
```

### Q: 合并前发现代码有问题怎么办？

```powershell
# 在功能分支继续修改
git checkout feature/my-feature

# 修复问题
git add .
git commit -m "fix: resolve issue before merge"

# 然后继续合并流程
git checkout master
git merge feature/my-feature --no-ff
```

### Q: 推送失败怎么办？

```powershell
# 检查远端状态
git fetch origin

# 如果远端有新提交
git pull origin master --rebase

# 重新推送
git push origin master
```

## 检查清单

每次开发前：
- [ ] 在主分支上
- [ ] 已拉取最新远端代码
- [ ] 工作区干净
- [ ] 创建了功能分支

每次开发后：
- [ ] 所有更改已提交
- [ ] 功能分支已合并到主分支
- [ ] 已推送到 origin/master
- [ ] 功能分支已清理
- [ ] 向用户报告PR编号和状态

---

**最后更新**: 2026-10-18  
**适用版本**: v1.0+
