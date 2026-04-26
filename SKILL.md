---
name: github-ops
description: |
  GitHub 仓库全生命周期运维管理技能。
  涵盖：仓库审计、代码推送、分支管理、PR 管理、CI/CD 配置、Issue 管理、
  Release 管理、开发者引流、日常运维、错误处理。
  当用户提到"GitHub 运维"、"仓库管理"、"项目运维"、"引流"、"开源运营"时使用。
homepage: https://github.com/xiejianjun000
metadata:
  openclaw:
    emoji: '🔧'
    requires: { env: ['GITHUB_TOKEN'] }
    primaryEnv: 'GITHUB_TOKEN'
  security:
    credentials_usage: |
      Requires GitHub Personal Access Token (PAT) with repo scope.
      Token is ONLY sent to api.github.com as Authorization header.
      Never log, echo, or commit the token.
    allowed_domains:
      - api.github.com
---

# GitHub 仓库运维管理 Skill

> 基于 OpenTaiji 生态项目实战经验提炼（2026-04-25~26）

## 概述

本技能覆盖 GitHub 仓库的**全生命周期运维管理**，包括日常维护、项目完善、开发者引流三个层次。

### 适用场景

| 场景 | 触发词 |
|------|--------|
| 仓库审计 | "看看仓库状态"、"项目健康检查" |
| 代码推送 | "推送代码"、"更新仓库" |
| 分支管理 | "合并 PR"、"清理分支" |
| 项目完善 | "完善项目"、"优化仓库" |
| 开发者引流 | "增加曝光"、"引流"、"招贡献者" |
| 日常运维 | "每日运维"、"项目日报" |

---

## 前置条件

### 1. 凭证配置

```bash
# 方式 A — 环境变量（推荐）
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"

# 方式 B — 配置文件
mkdir -p ~/.config/github
echo "your_token" > ~/.config/github/token
```

**Token 权限要求**：`repo`（完整仓库权限）

### 2. 基础配置

```bash
# 加载凭证（每次操作前执行）
GITHUB_TOKEN="${GITHUB_TOKEN:-$(cat ~/.config/github/token 2>/dev/null)}"
if [ -z "$GITHUB_TOKEN" ]; then
  echo "❌ 缺少 GitHub Token"
  exit 1
fi

# API 基础配置
GH_API="https://api.github.com"
GH_AUTH="-H \"Authorization: token $GITHUB_TOKEN\" -H \"Accept: application/vnd.github+json\""
```

### 3. SSL 问题解决（云服务器环境）

云服务器可能无法直接连接 GitHub API，使用 `--resolve` 绕过：

```bash
# 获取 GitHub API IP
nslookup api.github.com 2>/dev/null | grep "Address:" | tail -1 | awk '{print $2}'
# 通常为 140.82.121.6

# 使用 resolve 绕过 SSL 拦截
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/:owner/:repo"
```

---

## 核心操作

### 一、仓库审计（必做第一步）

在任何运维操作前，先全面了解仓库状态。

#### 1.1 列出所有仓库

```bash
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/user/repos?affiliation=owner&per_page=50" | python3 -c "
import sys, json
repos = json.loads(sys.stdin.read())
for r in repos:
    vis = '🔒 私有' if r.get('private') else '🌍 公开'
    print(f'{vis} {r[\"name\"]} | {r.get(\"language\",\"N/A\")} | {r[\"size\"]} KB | {r[\"updated_at\"][:10]}')
"
```

#### 1.2 检查单个仓库详情

```bash
REPO="owner/repo"
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$REPO" | python3 -c "
import sys, json
d = json.loads(sys.stdin.read())
print(f'名称: {d.get(\"full_name\")}')
print(f'描述: {d.get(\"description\", \"无\")}')
print(f'语言: {d.get(\"language\", \"N/A\")}')
print(f'Stars: {d.get(\"stargazers_count\")}')
print(f'大小: {d.get(\"size\")} KB')
print(f'分支: {d.get(\"default_branch\")}')
print(f'更新时间: {d.get(\"updated_at\")}')
print(f'首页: {d.get(\"homepage\", \"无\")}')
"
```

#### 1.3 检查文件结构

```bash
REPO="owner/repo"
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$REPO/git/trees/main?recursive=1" | python3 -c "
import sys, json
d = json.loads(sys.stdin.read())
tree = d.get('tree', [])
files = [t for t in tree if t['type'] == 'blob']
dirs = [t for t in tree if t['type'] == 'tree']
print(f'文件数: {len(files)}')
print(f'目录数: {len(dirs)}')
print(f'总大小: {sum(t.get(\"size\",0) for t in files)} bytes')
# 显示顶层结构
top = sorted(set(t['path'].split('/')[0] for t in files))
print(f'顶层: {top}')
"
```

#### 1.4 检查 CI 状态

```bash
REPO="owner/repo"
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$REPO/actions/runs?per_page=3" | python3 -c "
import sys, json
d = json.loads(sys.stdin.read())
for r in d.get('workflow_runs', []):
    icon = '✅' if r.get('conclusion')=='success' else '❌' if r.get('conclusion')=='failure' else '🔄'
    print(f'{icon} {r[\"name\"]} → {r.get(\"conclusion\",\"运行中\")} ({r[\"created_at\"][:16]})')
"
```

#### 1.5 检查 PR 和 Issue

```bash
REPO="owner/repo"
# PRs
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$REPO/pulls?state=open" | python3 -c "
import sys, json
prs = json.loads(sys.stdin.read())
print(f'Open PRs: {len(prs)}')
for p in prs: print(f'  #{p[\"number\"]}: {p[\"title\"][:60]}')
"

# Issues
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$REPO/issues?state=open&per_page=10" | python3 -c "
import sys, json
issues = json.loads(sys.stdin.read())
real = [i for i in issues if 'pull_request' not in i]
print(f'Open Issues: {len(real)}')
for i in real: print(f'  #{i[\"number\"]}: {i[\"title\"][:60]}')
"
```

### 二、代码推送

#### 2.1 克隆仓库

```bash
OWNER="xiejianjun000"
REPO="repo-name"
cd /tmp
rm -rf "$REPO-ops" 2>/dev/null
git clone "https://$OWNER:$GITHUB_TOKEN@github.com/$OWNER/$REPO.git" "$REPO-ops"
cd "$REPO-ops"
git config user.name "Your Name"
git config user.email "your@email.com"
```

#### 2.2 修改并推送

```bash
# 修改文件...
git add -A
git commit -m "type: description"
git push origin main
```

#### ⚠️ 重要警告：GitHub Secret Scanning

**绝对禁止**在 commit 中包含 Token 的文件。如果脚本中包含 Token，GitHub 会拒绝推送：

```bash
# ❌ 错误：脚本中包含 Token
echo "curl -H 'Authorization: token ghp_xxx' ..." > script.sh
git add script.sh
git commit -m "add script"
git push  # ❌ 会被 GitHub Secret Scanning 拒绝！

# ✅ 正确：使用环境变量
echo 'curl -H "Authorization: token $GITHUB_TOKEN" ...' > script.sh
git add script.sh
git commit -m "add script"
git push  # ✅ 安全
```

### 三、分支管理

#### 3.1 创建分支

```bash
git checkout -b feature/branch-name
# 修改...
git push origin feature/branch-name
```

#### 3.2 创建 PR

```bash
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/pulls" \
  -d '{
    "title": "PR 标题",
    "body": "PR 描述",
    "head": "feature/branch-name",
    "base": "main"
  }'
```

#### 3.3 合并 PR

```bash
# 方式 A：通过 API 合并（需要审核权限）
PR_NUMBER=1
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/pulls/$PR_NUMBER/merge" \
  -d '{"commit_title":"合并说明","merge_method":"merge"}'

# 方式 B：如果 main 有保护规则，需要先创建 PR 再合并
# 保护规则要求：至少 1 个审核 + 状态检查通过
```

#### 3.4 清理过期分支

```bash
# 列出远程分支
git branch -r

# 删除远程分支
git push origin --delete branch-name

# 批量清理（Dependabot 分支等）
for branch in dependabot/* fix/* release/*; do
  git push origin --delete "$branch" 2>/dev/null
done
```

### 四、CI/CD 配置

#### 4.1 添加工作流

```bash
mkdir -p .github/workflows
cat > .github/workflows/ci.yml << 'EOF'
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22' }
      - run: npm ci
      - run: npm test
EOF
```

#### 4.2 修复 CI 失败

常见 CI 失败原因及修复：

| 错误 | 原因 | 修复 |
|------|------|------|
| TypeScript 编译失败 | rootDir 配置错误 | 修改 tsconfig.json |
| 测试超时 | 测试运行时间过长 | 添加 --forceExit |
| 依赖安装失败 | package-lock.json 不匹配 | 删除后重新 npm install |
| Jest 参数过时 | --testPathPattern → --testPathPatterns | 更新参数名 |

### 五、项目完善（开发者引流基础）

#### 5.1 优化仓库描述

```bash
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -X PATCH \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO" \
  -d '{"description":"项目描述，包含关键词","homepage":"https://project-website.com"}'
```

#### 5.2 添加 Topics（搜索标签）

```bash
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/topics" \
  -d '{"names":["keyword1","keyword2","keyword3","typescript","open-source"]}'
```

**推荐标签策略**：
- 技术栈：`typescript`, `javascript`, `python`
- 领域：`ai-hallucination`, `agent-framework`, `anti-hallucination`
- 特性：`open-source`, `mit-license`, `cli`
- 独特：项目特有词

#### 5.3 添加 Issue Labels

```bash
# 批量创建标签
for label in 'good first issue:green' 'bug:d73a4a' 'enhancement:a2eeef' 'documentation:0075ca' 'help wanted:008672' 'question:d87634'; do
  name="${label%%:*}"
  color="${label##*:}"
  curl -sk --resolve api.github.com:443:140.82.121.6 -X POST \
    -H "Authorization: token $GITHUB_TOKEN" \
    -H "Accept: application/vnd.github+json" \
    "$GH_API/repos/$OWNER/$REPO/labels" \
    -d "{\"name\":\"$name\",\"color\":\"$color\"}" 2>/dev/null
done
```

#### 5.4 创建 Good First Issues

```bash
# 创建新手友好的 Issue
curl -sk --resolve api.github.com:443:140.82.121.6 -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/issues" \
  -d '{
    "title": "🌟 任务标题（Good First Issue）",
    "body": "## 任务描述\n\n详细说明...\n\n## 完成标准\n\n- [ ] 步骤1\n- [ ] 步骤2\n\n## 资源\n\n- 参考文件：`path/to/file`\n\n## 需要帮助？\n\n在 Issue 下方留言！",
    "labels": ["good first issue"]
  }'
```

**Good First Issue 设计原则**：
1. 任务明确，有具体的完成标准
2. 提供参考文件路径
3. 预留留言入口，方便新手提问
4. 难度适中，不需要深入理解整个项目

#### 5.5 添加 CONTRIBUTING.md

```bash
cat > CONTRIBUTING.md << 'EOF'
# Contributing to 项目名称

感谢你的关注和贡献！

## 快速开始

```bash
git clone https://github.com/owner/repo.git
cd repo
npm install
npm test  # 运行测试
```

## 开发流程

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/name`)
3. 提交更改 (`git commit -m 'feat: add feature'`)
4. 推送分支 (`git push origin feature/name`)
5. 提交 Pull Request

## 代码规范

- 使用 ESLint 进行代码检查
- 提交信息遵循 Conventional Commits 格式

## Good First Issues

查看带有 `good first issue` 标签的 Issues。

## 需要帮助？

在 Issue 中留言，我们会尽快回复！
EOF
```

#### 5.6 添加 Release Tag

```bash
# 本地打标签
git tag -a v1.0.0 -m "Release v1.0.0 - 版本说明"

# 推送标签
git push origin v1.0.0

# 或通过 API 创建 Release
curl -sk --resolve api.github.com:443:140.82.121.6 -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/releases" \
  -d '{
    "tag_name": "v1.0.0",
    "name": "v1.0.0",
    "body": "Release notes...",
    "draft": false,
    "prerelease": false
  }'
```

#### 5.7 添加 README Badges

```markdown
[![npm version](https://img.shields.io/badge/npm-v0.1.0-blue)](https://www.npmjs.com/package/package-name)
[![CI](https://img.shields.io/github/actions/workflow/status/owner/repo/ci.yml?label=CI)](https://github.com/owner/repo/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)
[![Stars](https://img.shields.io/github/stars/owner/repo?style=social)](https://github.com/owner/repo/stargazers)
[![Good First Issues](https://img.shields.io/github/issues/owner/repo/good%20first%20issue)](https://github.com/owner/repo/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22)
```

#### 5.8 开启 GitHub Discussions

```bash
curl -sk --resolve api.github.com:443:140.82.121.6 -X PATCH \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO" \
  -d '{"has_discussions":true}'
```

#### 5.9 添加 GitHub Pages 官网

```bash
# 创建 docs/index.html
mkdir -p docs
cat > docs/index.html << 'HTMLEOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>项目名称 - 简短描述</title>
<style>
body{font-family:system-ui;background:#0a0a0a;color:#e0e0e0;text-align:center;padding:4rem}
h1{font-size:3rem;margin-bottom:1rem}
a{color:#4ecdc4;text-decoration:none}
</style>
</head>
<body>
<h1>☯️ 项目名称</h1>
<p>简短描述</p>
<a href="https://github.com/owner/repo">⭐ Star on GitHub</a>
</body>
</html>
HTMLEOF

# 启用 GitHub Pages
curl -sk --resolve api.github.com:443:140.82.121.6 -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "$GH_API/repos/$OWNER/$REPO/pages" \
  -d '{"build_type":"workflow"}'

# 提交并推送
git add docs/index.html
git commit -m "docs: 添加 GitHub Pages 官网"
git push origin main
```

#### 5.10 添加社交预览图片

```bash
# 使用 Python 生成 1280x640 图片
python3 << 'PYEOF'
from PIL import Image, ImageDraw, ImageFont
img = Image.new('RGB', (1280, 640), '#0a0a0a')
draw = ImageDraw.Draw(img)
# 绘制项目信息...
img.save('social-preview.png')
PYEOF

# 提交并推送
git add social-preview.png
git commit -m "docs: 添加社交预览图片"
git push origin main
```

#### 5.11 项目互链（生态建设）

在相关项目的 README 中添加生态系统板块，互相引流：

```markdown
## 🌐 生态系统

| 项目 | 说明 | 链接 |
|------|------|------|
| **ProjectA** | 核心引擎 | [链接](https://github.com/owner/projectA) |
| **ProjectB** | 扩展工具 | [链接](https://github.com/owner/projectB) |
| **ProjectC** | 文档站点 | [链接](https://github.com/owner/projectC) |
```

### 六、日常运维

#### 6.1 每日运维检查清单

```bash
# 对所有仓库执行以下检查：

# 1. CI 状态
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  "$GH_API/repos/$OWNER/$REPO/actions/runs?per_page=1"

# 2. 新 PR 状态
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  "$GH_API/repos/$OWNER/$REPO/pulls?state=open"

# 3. 新 Issue 状态
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  "$GH_API/repos/$OWNER/$REPO/issues?state=open"

# 4. 依赖更新（Dependabot）
curl -sk --resolve api.github.com:443:140.82.121.6 \
  -H "Authorization: token $GITHUB_TOKEN" \
  "$GH_API/repos/$OWNER/$REPO/dependabot/alerts"

# 5. 分支清理（过期的 feature 分支）
git branch -r | grep -v main | grep -v master
```

#### 6.2 运维日报格式

```markdown
# GitHub 项目运维日报（YYYY-MM-DD）

## 📊 项目概览
| 项目 | CI | PR | Issue | 健康度 |
|------|----|----|----|----|

## 📁 各项目详情

### 项目名称
- **CI**: ✅ 通过 / ❌ 失败（说明原因）
- **PR**: X 个待合并 / Y 个待修复
- **Issue**: X 个开放 / Y 个已关闭
- **最新提交**: commit message（时间）
- **今日进展**: xxx
- **待办事项**: xxx

## 🛠️ 今日运维操作
- 执行了哪些维护操作

## ⚠️ 风险提示
- 需要关注的事项
```

### 七、开发者引流

#### 7.1 曝光提升清单

| 类别 | 项目 | 优先级 | 效果 |
|------|------|--------|------|
| **SEO 优化** | Topics + 描述优化 | 🔴 高 | 搜索排名提升 |
| **页面优化** | README badges + 社交图片 | 🔴 高 | 专业感↑ |
| **社区推广** | V2EX + 掘金 + 知乎 | 🔴 高 | 流量最大 |
| **包分发** | npm 发布 | 🔴 高 | 开发者发现 |
| **社区互动** | awesome 列表提交 | 🟡 中 | 持续被动曝光 |
| **内容营销** | 技术博客系列 | 🔴 高 | 长期 SEO |

#### 7.2 V2EX 发帖模板

```
标题：我做了一个[技术亮点]的[项目类型]，开源了，招贡献者

内容结构：
1. 背景问题（痛点描述）
2. 技术方案（核心原理）
3. 实际效果（数据支撑）
4. 开源地址（GitHub 链接）
5. 招贡献者（Good First Issues 列表）
```

#### 7.3 掘金文章模板

```
标题：[技术深度] 我用[技术方法]解决了[行业痛点]（开源）

内容结构：
1. 引言：行业背景和痛点
2. 技术原理：为什么这个方案更好
3. 核心实现：关键代码和架构图
4. 实际测试：数据和对比表格
5. 快速上手：安装和使用示例
6. 开源信息：GitHub 链接和 Issues 列表
7. 未来规划：Roadmap
```

---

## 错误处理

### 常见问题及解决方案

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `404 Not Found` | 仓库不存在或无权限 | 检查仓库名和 Token 权限 |
| `403 Forbidden` | Token 权限不足 | 使用 `repo` 范围的 Token |
| `409 Conflict` | 分支冲突或 Head 过期 | `git pull` 后重试 |
| `422 Validation Failed` | 标签已存在 | 使用不同的标签名或 `--force` |
| Secret Scanning 拒绝 | 提交中包含 Token | 从历史中移除 Token 文件 |
| SSL 握手失败 | 云服务器 SSL 拦截 | 使用 `--resolve` 参数 |
| npm publish 失败 | 未登录 | `npm adduser` 后重试 |
| PR 合并按钮灰色 | 保护规则要求审核 | 手动点击网页合并按钮 |

### Token 安全规则

1. **绝对不要**在 commit 中包含 Token
2. **绝对不要**在日志中打印 Token
3. **只在** HTTP Authorization Header 中使用 Token
4. 克隆时使用 `https://owner:TOKEN@github.com/owner/repo.git`
5. 脚本中使用 `$GITHUB_TOKEN` 环境变量

---

## 实战经验

### 从 OpenTaiji 生态项目中总结的经验

1. **先审计，再操作** — 全面了解仓库状态后再制定计划
2. **sub-agent 会超时** — 复杂任务设置 5 分钟超时，超时的话手动兜底
3. **保护分支无法 API 合并** — main 有保护规则时需要网页手动合并
4. **SSL 问题** — 云服务器连接 GitHub API 需要 `--resolve` 绕过
5. **Secret Scanning** — 绝对不能提交包含 Token 的文件
6. **Good First Issues 是引流核心** — 9 个 Issues 比 9 篇博客更有效
7. **生态互链** — 项目之间互相引用，形成合力
8. **npm 发布是质变** — 发布后开发者可通过 `npm install` 发现

### 推荐操作顺序

```
1. 仓库审计 → 2. 描述优化 → 3. Topics → 4. Labels → 
5. Good First Issues → 6. CONTRIBUTING.md → 7. Release Tag → 
8. README Badges → 9. GitHub Pages → 10. 社交图片 → 
11. 生态互链 → 12. V2EX/掘金发帖 → 13. npm 发布
```

---

## 参考资源

- [GitHub REST API 文档](https://docs.github.com/en/rest)
- [GitHub Topics API](https://docs.github.com/en/rest/repos/repos#update-all-repository-topics)
- [GitHub Pages API](https://docs.github.com/en/rest/pages)
- [Conventional Commits](https://www.conventionalcommits.org/)
