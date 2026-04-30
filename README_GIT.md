# Git 版本管理指南 — SageServe_modify_v1

## 当前状态

| 项目 | 详情 |
|------|------|
| 仓库位置 | `SageServe_modify_v1/` |
| 当前分支 | `main` |
| 远程仓库 | `https://github.com/wangdf15/SageServe_modify_v1.git` |
| 本地领先远程 | 2 个提交 |
| 工作区状态 | 干净（无未提交的更改） |

### 提交历史

```
* 0bb18d5  使项目跑通 记录在changes.md中
* a45b4e1  claude接入
* 635bcbd  initial commit
```

---

## 快速上手

### 查看仓库状态

```bash
# 查看工作区状态（最常用）
git status

# 查看提交历史
git log --oneline -10

# 查看具体更改内容
git diff            # 未暂存的更改
git diff --staged   # 已暂存待提交的更改
```

### 日常工作流（修改 → 暂存 → 提交 → 推送）

```bash
# 1. 查看你改了哪些文件
git status

# 2. 将改动加入暂存区
git add <文件路径>        # 添加单个文件
git add .                 # 添加当前目录所有改动
git add -A                # 添加所有改动（含删除）

# 3. 提交（本地保存快照）
git commit -m "描述你做了什么"

# 4. 推送到 GitHub
git push
```

> **注意**：当前本地有 2 个提交尚未推送，下次 `git push` 会一并将它们推送到 GitHub。

### 同步远程更新

```bash
# 拉取远程最新代码并合并到本地
git pull

# 仅拉取但不自动合并（安全查看远程有什么新东西）
git fetch
git diff origin/main   # 查看远程比本地多了什么
git merge origin/main  # 确认无误后合并
```

---

## .gitignore 说明

当前 `.gitignore` 已忽略以下内容：

| 忽略项 | 原因 |
|--------|------|
| `__pycache__/` | Python 编译缓存 |
| `results/` | 模拟运行结果目录 |
| `traces/*` | 请求轨迹数据 |
| `.DS_Store` | macOS 系统文件 |
| `*.lp` | MILP 求解器生成的文件 |
| `ilp_outputs/*` | ILP 输出文件 |
| `figures/*` | 图表输出 |

**注意**：`notebooks/` 目录下有 4 个大型 CSV 文件（>10MB）。如果这些文件不需要版本管理，建议将 `notebooks/*.csv` 加入 `.gitignore`。如果已经追踪了，需要用 `git rm --cached` 移除。

---

## 分支策略（推荐）

```
main ──── ● ──── ● ──── ● ──── ●  （稳定版，随时可发布）
              \
feature/xxx    ● ──── ●            （功能分支，完成后合并回 main）
```

常用操作：

```bash
# 创建并切换到新分支
git checkout -b feature/新功能名

# 切换回 main 分支
git checkout main

# 将功能分支合并到 main
git merge feature/新功能名

# 删除已完成的功能分支
git branch -d feature/新功能名

# 查看所有分支
git branch -a
```

---

## 处理大文件（>100MB）

GitHub 限制单个文件不能超过 100MB。如果项目中有大文件（如模型权重、大型数据集），推荐使用 **Git LFS**：

```bash
# 安装 Git LFS（首次使用）
git lfs install

# 追踪大文件类型
git lfs track "*.bin"
git lfs track "*.pt"
git lfs track "*.csv"   # 如果 CSV 文件很大

# 提交 .gitattributes（LFS 配置）
git add .gitattributes
git commit -m "配置 Git LFS 追踪大文件"
```

---

## 常用场景速查

### 撤销/回退操作

```bash
# 撤销对某文件的修改（回到上次提交的状态）
git checkout -- <文件名>

# 取消暂存（git add 的反操作）
git reset HEAD <文件名>

# 修改最近一次提交的说明
git commit --amend -m "新的提交信息"

# 回退到上一个提交（保留改动在工作区）
git reset HEAD~1

# 彻底回退到上一个提交（丢弃改动）
git reset --hard HEAD~1
```

### 查看历史

```bash
# 查看每次提交改了什么
git log -p

# 查看某文件的修改历史
git log --oneline -- <文件路径>

# 查看某行代码是谁改的
git blame <文件路径>
```

### 临时保存工作现场

```bash
# 暂存当前改动（没做完但需要切分支时用）
git stash

# 恢复最近一次暂存的改动
git stash pop

# 查看所有暂存
git stash list
```

---

## 最佳实践

1. **提交粒度**：每次提交做一件事，方便以后回溯。功能完成一个点就提交一次。
2. **提交信息**：动宾短语描述，清晰说明「做了什么」。中文或英文都可以，保持一致即可。
3. **推送前先拉取**：`git pull --rebase` 可以让历史更整洁（避免多余的合并提交）。
4. **不要提交大文件**：zip 包、数据集、模型权重等应通过 `.gitignore` 排除，或使用 Git LFS。
5. **定期推送**：本地工作完成后及时 `git push`，避免丢失代码。

---

## 当前待办

- [ ] 本地有 2 个未推送的提交，运行 `git push` 同步到 GitHub
- [ ] 检查 `notebooks/` 下的大型 CSV 文件是否需要加入 `.gitignore`
- [ ] 父目录的 `SageServe_modify_v1.zip` 不在 Git 追踪范围内，手动管理即可
