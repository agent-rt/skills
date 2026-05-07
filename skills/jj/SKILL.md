---
name: jj
version: "1.0.0"
description: Jujutsu (jj) VCS 实用技能：colocated 工作流、change vs commit 心智模型、日常命令套路、与 GitHub/git remote 协作。
license: MIT
metadata:
  namespace: official
  summary: jj-vcs 日常工作流：colocated 模式、squash/split/absorb、bookmark push、op log 救命操作
  tags: [jj, jujutsu, vcs, git, workflow]
  languages: []
  triggers: [jj, jujutsu, jj-vcs, "jj log", "jj new", "jj squash", "jj rebase", colocated, bookmark]
  requires:
    binaries:
      - { name: jj, version: ">=0.39" }
      - { name: git }
---

# Jujutsu (jj) Workflow

## When to use

User is using jj-vcs for version control instead of (or alongside) git. Apply this skill for any jj-related task: starting a new repo, committing, rebasing, pushing to GitHub, recovering from mistakes.

## Mental model — read this before any operation

jj 没有 git 的"暂存区 (index)"和"分支 (branch)"概念。核心三件事：

1. **Working copy = a commit (called `@`)**：你保存文件就等于在更新 `@` 这个 commit。无 `git add` / `git commit` 区分。
2. **Change ID vs Commit ID**：每个 change 有稳定的 `change_id`（如 `abcd...`），rebase / squash 后 `commit_id` 会变但 `change_id` 不变。日常引用都用 change_id。
3. **Bookmark = 命名指针**：替代 git branch。**不会自动跟随 `@` 移动**——这是最大反直觉点，必须显式 `jj bookmark move` 或 `jj git push -c @`（自动建 bookmark）。

## Decision tree — 用户描述任务时如何选命令

| 用户想做的事 | 命令 |
|---|---|
| 看历史 | `jj log`（默认只显示 stack + trunk 附近，由 `revsets.log` 控制） |
| 看完整历史 | `jj log -r ::` 或 alias `jj ll` |
| 开新工作 | `jj new -m "feat: x"`（在 `@` 之上建子 change） |
| 修改 message | `jj describe`（不会改 commit 身份，change_id 稳定） |
| "git commit --amend" | `jj squash`（把 `@` 合并进 `@-`） |
| "git add -p && commit --amend" | `jj squash -i`（交互选 hunks） |
| 拆 commit | `jj split`（交互选哪些改动留在原 change） |
| 把当前散乱改动自动归并到祖先 | `jj absorb`（神器，比 git rebase -i fixup 强） |
| rebase 到 main | `jj rebase -d main` |
| 撤销刚才任意操作 | `jj undo`（包括 rebase / abandon） |
| 看操作历史 | `jj op log`（比 git reflog 强：可还原任意时刻整个仓库状态） |
| 推到远端 | 见 [Push workflow](#push-workflow) |

## Colocated mode (default since v0.39)

在已有 git 仓库里：
```bash
jj git init --colocate    # 新仓库
jj git clone --colocate <url>   # 克隆
```

`.git` 和 `.jj` 共存。**只读** git 命令（`git log` / `git fetch` / `gh`）继续用，**写入**操作走 jj 避免双模型脑裂。

## Push workflow

jj 不像 git 那样"当前分支"概念。push 必须显式：

```bash
# 推当前 change（自动创建 bookmark，模板由 templates.git_push_bookmark 定义）
jj git push -c @

# 把最近一个 bookmark 拽到当前位置，然后推（典型 alias: jj tug）
jj bookmark move --from 'closest_bookmark(@-)' --to @-
jj git push

# 推所有 bookmark
jj git push --all
```

**`private-commits` 防呆**：配置中 `git.private-commits = "description(glob:'wip:*') | ..."` 会阻止匹配的 commit 被 push，相当于"留个 wip:" 就不会被推出去。

## Conflict resolution

jj 的冲突是**一等公民**——它会保存到 commit 里，不阻塞操作。`jj log` 会标 `conflict`。解决：

```bash
jj resolve            # 启动 merge tool（配置里 merge-editor = vscode）
jj resolve --list     # 看哪些文件冲突
```

注意 jj rebase **不会停下来让你 resolve**，它直接把冲突 commit 出来。这正常，慢慢 resolve 就行。

## 与 GitHub PR 协作

```bash
jj git fetch                            # 拉 remote
jj new main@origin                      # 基于 main 开新 change
# ...edit, jj describe...
jj git push -c @                        # 创建 bookmark 推送
gh pr create                            # 建 PR（在 colocated 仓库里 gh 正常用）
```

更新 PR：直接修改 `@`，再 `jj git push`（force-with-lease 自动加上，因为 change_id 没变 jj 知道是同一个 PR）。

## 救命操作

- 误删/误 abandon：`jj op log` 找到操作 ID → `jj op restore <op_id>`
- 误 rebase：`jj undo`（默认撤销最近一次操作）
- 想知道某 change 经历过什么：`jj evolog -r <change_id>`

## 常见错误模式

1. **以为 bookmark 跟随 `@`**：每次想 push 之前都得想"bookmark 在哪？要不要先 move？"——这就是 `jj tug` alias 存在的原因。
2. **conflicted commit 不处理就推出去**：会推到远端导致协作者困惑。push 前 `jj log` 检查无 `conflict` 标记。
3. **在 detached 状态新建 change 但没建 bookmark 就重启 jj**：change 会被 GC。重要 change 一定建 bookmark 或 push。
4. **混用 `git commit` 和 jj**：colocated 模式下 git commit 后下一次 jj 操作会自动 import，但 message 可能错位。建议只用 jj 写入。

## 用户配置参考（home-manager）

仓库路径自动切换 user.email（替代 git 的 includeIf）：
```nix
"--scope" = [{
  "--when".repositories = [ "~/work/*" ];
  user.email = "work@example.com";
}];
```

push bookmark 命名模板：
```nix
templates.git_push_bookmark = ''"username/push-" ++ change_id.short()'';
```

## Don't

- 不要给用户推荐 `jj branch`（已废弃，改名 `jj bookmark`）
- 不要推荐 `git.push-bookmark-prefix`（已 deprecate，用 `templates.git_push_bookmark`）
- 不要在非 colocated 仓库里建议 git 命令——会失败或产生不一致
- 不要建议用 `jj abandon` 来"删除分支"——abandon 是丢弃 change，bookmark 用 `jj bookmark delete`
