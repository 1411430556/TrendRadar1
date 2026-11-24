---
name: git-pull-safe
description: 安全拉取远端更新到本地仓库。用户说拉取更新、pull、同步远端、fetch 合并时使用。先检查本地未提交改动并用 stash 兜底，禁止擅自 commit、reset、force pull。不适用于首次克隆或切换分支。
---

# 安全拉取更新

按以下顺序执行，每步确认无误再进入下一步。环境为 Windows PowerShell 5.1，不支持 `&&`，必须分条执行或用 `; if ($?) { ... }`。

## 1. 检查工作区状态（必须先做）

```
git status -sb
```

判断：
- 工作区干净（只有 `## master...origin/master`，无 `M`/`??`）→ 进入第 2 步
- 有 `M` 或 `??` → 进入「本地有未提交改动」分支，**不要直接 pull**

## 2. 下载远端最新提交（不动本地文件）

```
git fetch origin
git status -sb
git log --oneline HEAD..origin/master
git diff --stat HEAD...origin/master
```

- 如果 `git status -sb` 没有 `[behind N]`，说明已是最新，直接结束，告知用户无需拉取
- 查看 `git log` 和 `git diff --stat`，确认远端改了哪些文件，评估冲突风险
- 本仓库常见场景：用户在 GitHub 网页直接改 `config/frequency_words.txt`，需特别留意该文件

## 3. 合并

优先快进：

```
git merge --ff-only origin/master
```

- 成功 → 完成，告知用户合并内容摘要和当前状态
- 失败（提示 `Not possible to fast-forward`）→ 说明本地也有新提交、两边分叉。**停下询问用户**选 merge 还是 rebase，不要擅自决定

## 4. 关于 push

- 拉取本身不需要 push。只有本地产生新提交（如合并提交、解决冲突后的提交）且用户明确要求时才 push
- 不要主动 push

## 本地有未提交改动时的处理

遇到 pull 被阻塞（报错 `Your local changes would be overwritten by merge` 或本地有 `M`/`??`），**先用状态和差异定位阻塞点**：

```
git status -sb
git diff                    # 已跟踪文件的改动
git diff -- <file>          # 精确到单文件
```

然后给用户列出以下方案，**等用户选择后再执行**，禁止擅自选 commit 或丢弃：

1. **stash（默认推荐，可逆）** — `git stash push -u -m "拉取前暂存"`，拉取后 `git stash pop`
2. **导出补丁** — `git diff > backup.patch`，拉取后 `git apply backup.patch`
3. **commit** — 仅当用户明确要保留为正式变更时
4. **丢弃** — `git checkout -- .` 和 `git clean -fd`，必须用户明确确认不可逆

选 stash 时，`git stash pop` 若冲突，解决后正常处理，不要丢 stash 记录。

## 冲突解决原则

- 解决冲突前先看冲突片段和两边 diff，理解各自意图
- 配置文件（如 `config.yaml`）注意与已自动合并的英文版（`config.en.yaml`）保持结构一致
- 解决后用对应解析器校验（如 Python `yaml.safe_load`），再 `git add` 标记解决
- commit/push 等高影响写操作前二次确认

## 结束汇报

告诉用户：合并了哪些提交、改了什么文件、当前是否与远端同步、是否需要后续 push。
