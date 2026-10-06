# PenguinAgent Issue Thanks Bot

自动为 [HuHoBot/PenguinAgent](https://github.com/HuHoBot/PenguinAgent) 的所有开启和关闭 issues 添加统一感谢评论。  
如果该 issue 已经存在相同内容，无论作者是谁，都不会重复评论。

## 功能

- 每 15 分钟自动检查一次
- 覆盖 open 和 closed 状态的 issues
- 支持手动触发
- 精确检查是否已存在相同评论
- 使用 GitHub CLI (`gh`) 操作 Issues

## 评论内容

```text
# 感谢您的反馈，我们会评估相关策略，感谢对HuHoBot-PenguinAgent的支持
```

## 部署方式

### 方式一：直接部署在目标仓库

适合你有 `HuHoBot/PenguinAgent` 写权限的情况。

1. 在 `HuHoBot/PenguinAgent` 仓库创建 `.github/workflows/auto-comment.yml`
2. 粘贴以下代码：

```yaml
name: Auto Comment on Issues

on:
  schedule:
    - cron: '*/15 * * * *'
  workflow_dispatch:

permissions:
  issues: write

jobs:
  auto-comment:
    runs-on: ubuntu-latest
    steps:
      - name: Auto comment on issues
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO: ${{ github.repository }}
          COMMENT_BODY: |
            # 感谢您的反馈，我们会评估相关策略，感谢对HuHoBot-PenguinAgent的支持
        run: |
          set -euo pipefail

          echo "Fetching issues from $REPO ..."
          issue_numbers=$(gh issue list \
            --repo "$REPO" \
            --state all \
            --json number \
            --limit 1000 \
            --jq '.[].number')

          if [ -z "$issue_numbers" ]; then
            echo "No issues found."
            exit 0
          fi

          for num in $issue_numbers; do
            echo "Checking issue #$num ..."

            comments=$(gh issue view "$num" \
              --repo "$REPO" \
              --json comments \
              --jq '.comments[].body' 2>/dev/null || echo "")

            if printf '%s' "$comments" | grep -qF "$COMMENT_BODY"; then
              echo "  -> Already has the comment. Skipping."
            else
              echo "  -> Adding comment..."
              gh issue comment "$num" --repo "$REPO" --body "$COMMENT_BODY"
              echo "  -> Done."
            fi
          done
```

3. 进入仓库 **Settings → Actions → General → Workflow permissions**，选择 **Read and write permissions**。
4. 进入 **Actions**，选择 `Auto Comment on Issues`，点击 **Run workflow** 测试。

### 方式二：独立仓库 + PAT

适合你没有目标仓库写权限，或者想单独管理这个机器人。

1. 新建仓库，例如 `penguinagent-issue-thanks-bot`。
2. 创建 Fine-grained PAT：
   - Repository access 选择 `HuHoBot/PenguinAgent`
   - Permissions → Issues 设置为 **Read and write**
3. 在新仓库添加 secret：
   - 名称：`GH_PAT`
   - 值：刚才创建的 PAT
4. 创建 `.github/workflows/auto-comment.yml`，粘贴以下代码：

```yaml
name: Auto Comment on PenguinAgent Issues

on:
  schedule:
    - cron: '*/15 * * * *'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  auto-comment:
    runs-on: ubuntu-latest
    steps:
      - name: Auto comment on issues
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
          REPO: HuHoBot/PenguinAgent
          COMMENT_BODY: |
            # 感谢您的反馈，我们会评估相关策略，感谢对HuHoBot-PenguinAgent的支持
        run: |
          set -euo pipefail

          echo "Fetching issues from $REPO ..."
          issue_numbers=$(gh issue list \
            --repo "$REPO" \
            --state all \
            --json number \
            --limit 1000 \
            --jq '.[].number')

          if [ -z "$issue_numbers" ]; then
            echo "No issues found."
            exit 0
          fi

          for num in $issue_numbers; do
            echo "Checking issue #$num ..."

            comments=$(gh issue view "$num" \
              --repo "$REPO" \
              --json comments \
              --jq '.comments[].body' 2>/dev/null || echo "")

            if printf '%s' "$comments" | grep -qF "$COMMENT_BODY"; then
              echo "  -> Already has the comment. Skipping."
            else
              echo "  -> Adding comment..."
              gh issue comment "$num" --repo "$REPO" --body "$COMMENT_BODY"
              echo "  -> Done."
            fi
          done
```

5. 进入 **Actions**，选择 `Auto Comment on PenguinAgent Issues`，点击 **Run workflow** 测试。

## 权限说明

| 场景 | 权限 |
|---|---|
| 目标仓库内使用 `GITHUB_TOKEN` | `permissions: issues: write` |
| 独立仓库使用 Fine-grained PAT | Repository access 选 `HuHoBot/PenguinAgent`，Issues 设为 `Read and write` |
| 独立仓库使用 Classic PAT | 公共仓库 `public_repo`，私有仓库 `repo` |

## 常见问题

### 为什么没有评论？

- 检查 Actions 是否启用。
- 检查 token 权限是否包含 Issues 写权限。
- 查看 Actions 运行日志。
- 如果 issue 已经存在完全相同内容，会自动跳过。

### 会重复评论吗？

不会。脚本会读取 issue 的所有评论，只要有一条评论包含目标内容，就跳过，不论作者是谁。

### 如何修改检查频率？

修改 `cron` 表达式即可。GitHub Actions 最短间隔为 5 分钟，但实际执行可能会有延迟。

## License

MIT
