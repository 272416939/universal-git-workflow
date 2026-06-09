---
name: "universal-git-workflow"
description: "通用 Git 工作流技能，封装 GitHub+Gitee 双平台安全 pull/push、Conventional Commits 自动提交、语义化版本管理、CHANGELOG 自动生成、Release 发布、双平台端到端测试等完整开发工作流。当用户需要提交代码、同步仓库、发布版本、管理更新日志或执行双平台 Git 操作时调用。"
---

# 通用 Git 工作流 Skill

本 Skill 为 AI Agent 提供完整的 Git 工作流操作指南，涵盖从代码提交到版本发布的全流程自动化支持。所有操作均以安全为第一原则，避免数据丢失和误操作。

---

## 一、双平台 Remote 检测与平台选择

### 1.1 检测逻辑

通过 `git remote -v` 解析 URL 自动判定平台归属：

- 包含 `github.com` → GitHub 平台
- 包含 `gitee.com` → Gitee 平台
- 如果同时存在两者 → 双平台

支持的 remote 名称模式：`origin`、`github`、`gitee` 或任意自定义名称，只要 URL 匹配即可判定。

在 PowerShell 中执行：

```powershell
$remotes = git remote -v 2>&1
$hasGithub = $remotes | Select-String "github.com"
$hasGitee = $remotes | Select-String "gitee.com"
```

应遍历所有 remote，记录每个 remote 名称与其 URL 的平台归属。不要假设 remote 名称就是平台名，必须通过 URL 判定。

### 1.2 各场景处理

| 场景 | 处理方式 |
|------|----------|
| **双平台存在** | 进入平台选择交互流程（见 1.3） |
| **单一平台** | 自动使用该平台，跳过交互 |
| **无 remote** | 提示用户先用 `git remote add` 配置远程仓库，中止后续操作 |

### 1.3 平台选择交互

当检测到双平台时，Agent 应使用 `AskUserQuestion` 工具询问用户选择推送目标：

- **选项1**："双端推送 (GitHub + Gitee)"（标签: `"双端推送"`）
- **选项2**："仅 GitHub"
- **选项3**："仅 Gitee"

询问时的标准话术示例：

> 检测到当前仓库同时配置了 GitHub 和 Gitee 远程仓库。请选择本次推送的目标平台：

### 1.4 --target 参数

用户可通过参数跳过交互：

- `--target both` / `-t both` → 双端推送
- `--target github` / `-t github` → 仅 GitHub
- `--target gitee` / `-t gitee` → 仅 Gitee

推送时根据选择执行：

- **选 GitHub**：`git push github <branch>` 或 `git push origin <branch>`（如 origin 指向 GitHub）
- **选 Gitee**：`git push gitee <branch>`
- **选双端**：先 `git push github <branch>`，再 `git push gitee <branch>`；任一步失败即停止报错

代码示例：

```powershell
if ($target -eq "both") {
    git push github $branch
    if ($LASTEXITCODE -ne 0) {
        Write-Error "GitHub 推送失败，中止双端推送"
        exit 1
    }
    git push gitee $branch
    if ($LASTEXITCODE -ne 0) {
        Write-Error "Gitee 推送失败"
        exit 1
    }
}
elseif ($target -eq "github") {
    git push github $branch
}
elseif ($target -eq "gitee") {
    git push gitee $branch
}
```

---

## 二、UTF-8 编码安全保障

### 2.1 PowerShell 环境设置

在执行任何 git 命令之前，务必先执行以下 UTF-8 初始化：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()
chcp 65001 > $null
$env:LC_ALL = 'C.UTF-8'
$env:LANG = 'C.UTF-8'
```

此初始化适用于所有场景，包括 `--full`、`--pull-only`、`--commit-only` 等子功能。

### 2.2 Git 编码配置

建议用户配置以下 Git 全局编码设置（仅首次建议，不强制修改）：

```bash
git config --global core.quotepath false
git config --global i18n.commitencoding utf-8
git config --global i18n.logoutputencoding utf-8
git config --global gui.encoding utf-8
```

### 2.3 乱码排查指南

| 症状 | 检查项 | 解决方案 |
|------|--------|----------|
| 提交消息乱码 | `git config i18n.commitencoding` | 设为 `utf-8` |
| log 输出乱码 | `git config i18n.logoutputencoding` | 设为 `utf-8`，必要时设置 `$env:LESSCHARSET=utf-8` |
| 文件内容乱码 | `file -bi <filename>`（Git Bash）或 `Get-Content` 查看编码 | 用 `iconv` 或 PowerShell `Set-Content -Encoding UTF8` 转换 |

---

## 三、安全 Pull 工作流

### 3.1 标准安全 Pull 流程

**禁止直接使用 `git pull`**，始终按以下分步执行：

```bash
# 步骤1: 检查工作区是否干净
$status = git status --porcelain
if ($status) {
    # 工作区不干净，走 3.2 脏工作区处理流程
}

# 步骤2: 获取远程变更
git fetch <remote>

# 步骤3: 查看本地与远程的差异
git diff HEAD..<remote>/<branch> --stat
# 查看提交差异
git log HEAD..<remote>/<branch> --oneline

# 步骤4: 如果无冲突，合并或变基
git merge <remote>/<branch>
# 或 git rebase <remote>/<branch>
```

详解：
- `git diff HEAD..<remote>/<branch> --stat` 展示文件级变更摘要，让用户了解远程有什么改动
- `git log HEAD..<remote>/<branch> --oneline` 展示远程新增的提交列表
- Agent 应将这些信息展示给用户，让用户知悉即将合并的内容后再执行 merge/rebase

### 3.2 工作区脏时处理

当 `git status --porcelain` 输出非空时：

```
当前工作区有未暂存的修改：

→ 建议1: git stash（暂存后 pull，然后 git stash pop）
→ 建议2: git add + git commit（先提交再 pull）
→ 严禁：跳过检查直接强制拉取
```

Agent 应使用 `AskUserQuestion` 让用户选择：

- **选项1**："git stash（暂存修改 → pull → 恢复修改）"
- **选项2**："先提交再 pull"
- **选项3**："放弃 pull 操作"

若用户选择 stash，流程如下：

```bash
git stash
# 执行 3.1 的 pull 流程
git stash pop
# 如果有冲突，提示用户解决
```

### 3.3 冲突处理

当 `git merge` 或 `git rebase` 产生冲突时：

1. **列出冲突文件**：

```bash
git diff --name-only --diff-filter=U
```

2. **明确告知用户**：每个冲突文件的完整路径，以及冲突的大致内容

3. **提示解决步骤**：

```
发现合并冲突，请按以下步骤解决：

1. 编辑冲突文件，找到并解决 <<<<<<< / ======= / >>>>>>> 标记
2. 将解决后的文件加入暂存：
   git add <已解决的文件路径>
3. 继续合并/变基：
   git merge --continue
   # 或
   git rebase --continue

若要放弃合并/变基：
   git merge --abort
   # 或
   git rebase --abort
```

---

## 四、Conventional Commit 自动提交

### 4.1 分析 Diff 确定提交类型

执行 `git diff --staged`（如无 staged 文件则用 `git diff`）分析变更内容，按以下规则推断 type：

| 变更特征 | 推断 type |
|---------|----------|
| 新文件/新函数/新功能/新模块 | `feat` |
| 修复 bug/错误处理/异常修复 | `fix` |
| 仅改变格式/缩进/空格/代码风格 | `style` |
| 代码重构无功能变化 | `refactor` |
| 性能优化/性能相关改进 | `perf` |
| 文档变更（.md 文件、注释修改） | `docs` |
| 测试文件/测试用例/测试配置 | `test` |
| 构建/依赖变更（package.json、Cargo.toml 等） | `build` |
| CI 配置变更（.github/workflows、Jenkinsfile 等） | `ci` |
| 其他杂项/无法归类 | `chore` |

当变更同时匹配多个特征时，优先级：`feat` > `fix` > `perf` > `refactor` > `style` > `docs` > `test` > `build` > `ci` > `chore`。

### 4.2 推断 Scope

从变更文件的路径推断 scope，取第一级目录或模块名：

- `src/auth/login.ts` → scope: `auth`
- `docs/api.md` → scope: `docs`
- `package.json` → scope: 空（不写 scope）
- 多个目录变更时 → 取出现最频繁的目录，或留空

Scope 应为小写英文字母、数字和短横线，长度不超过 20 字符。

### 4.3 Breaking Change 检测

以下情况标记为 Breaking Change：

- diff 中包含 `BREAKING CHANGE:` 字样
- 函数签名变化、API 接口参数变化
- 配置文件格式发生不兼容变更
- 删除公开 API 或导出的模块

标记方式：在 type 后添加 `!`，如 `feat(api)!: 重构用户认证接口`。

提交体（body）中应附加：

```
BREAKING CHANGE: <具体说明破坏性变更的内容和迁移方法>
```

### 4.4 提交执行

标准提交流程：

```bash
# 1. 检查是否有 staged 文件
$staged = git diff --staged --name-only
if (-not $staged) {
    # 提示用户：没有暂存的文件，是否暂存所有变更？
    # 使用 AskUserQuestion 让用户选择
}

# 2. 暂存所有变更
git add -A

# 3. 生成 Conventional Commit 消息
$type = "feat"       # 根据 4.1 推断
$scope = "auth"      # 根据 4.2 推断
$desc = "添加用户登录功能"  # 根据 diff 内容生成

$commitMsg = "$type($scope): $desc"
# 或带 ! 的 breaking change: "$type($scope)!: $desc"

# 4. 提交
git commit -m $commitMsg
```

### 4.5 提交消息格式规范

- **description** 使用现在时、祈使语气，中文用动词开头
- **首行不超过 72 字符**（中文字符计为 2 字符宽度，约 36 个中文字）
- **type 和 scope** 始终使用英文小写
- scope 用括号包裹，紧跟 type 后、冒号前

标准格式：`<type>(<scope>): <description>`

示例：

- `feat(user): 添加用户登录功能`
- `fix(api): 修复空指针异常`
- `refactor(database): 重构连接池管理`
- `docs(readme): 更新安装说明`
- `perf(query)!: 优化搜索算法，变更查询接口签名`

---

## 五、语义化版本号管理

### 5.1 版本号格式

遵循 [SemVer 2.0.0](https://semver.org/lang/zh-CN/) 规范：`MAJOR.MINOR.PATCH`

- **MAJOR**：破坏性 API 变更（不向后兼容）
- **MINOR**：向后兼容的新功能
- **PATCH**：向后兼容的 bug 修复

### 5.2 自动递增规则

基于最近一次提交的 type 和是否包含 Breaking Change 决定：

| 触发条件 | 版本变化 | 示例 |
|----------|----------|------|
| `feat` + 含 `!` 或 `BREAKING CHANGE` | MAJOR+1, MINOR=0, PATCH=0 | 1.2.3 → 2.0.0 |
| `feat`（非破坏性） | MINOR+1, PATCH=0 | 1.2.3 → 1.3.0 |
| `fix` / `perf` / `refactor` / `style` / `docs` / `chore` / `test` / `build` / `ci` | PATCH+1 | 1.2.3 → 1.2.4 |

### 5.3 版本号持久化

版本号读取优先级（按顺序查找）：

1. `VERSION` 文件（纯文本，内容如 `1.2.3`）
2. `package.json` 的 `version` 字段
3. `Cargo.toml` 的 `[package] version` 字段
4. 都不存在则创建 `VERSION` 文件，初始值 `0.1.0`

读取当前版本后，根据 5.2 规则递增，并写回同一位置。

更新版本号的 PowerShell 命令：

```powershell
# 写入 VERSION 文件
$newVersion = "1.3.0"
Set-Content -Path "VERSION" -Value $newVersion -Encoding UTF8

# 更新 package.json（如适用）
$package = Get-Content "package.json" -Raw -Encoding UTF8 | ConvertFrom-Json
$package.version = $newVersion
$package | ConvertTo-Json -Depth 10 | Set-Content "package.json" -Encoding UTF8

# 更新 Cargo.toml（如适用）
(Get-Content "Cargo.toml" -Encoding UTF8) -replace '^version = ".*"', "version = `"$newVersion`"" | Set-Content "Cargo.toml" -Encoding UTF8
```

### 5.4 手动指定与预发布

命令行参数：

- `--version 2.0.0`：跳过自动递增，直接使用指定版本号
- `--prerelease alpha`：版本号加 `-alpha.N` 后缀（N 为当前时间戳 `Get-Date -Format "yyyyMMddHHmm"`）
- `--prerelease beta`：版本号加 `-beta.N` 后缀
- `--prerelease rc`：版本号加 `-rc.N` 后缀

示例：
- `--version 2.0.0` → 版本号设为 `2.0.0`
- `--prerelease alpha` → `1.3.0-alpha.202406091930`
- `--version 2.0.0 --prerelease rc` → `2.0.0-rc.202406091930`

---

## 六、CHANGELOG 自动生成

### 6.1 从 Git Log 提取提交

获取提交范围：

```bash
# 从上一个 tag 到 HEAD（增量）
$lastTag = git describe --tags --abbrev=0 2>$null
if ($lastTag) {
    git log --oneline --no-merges $lastTag..HEAD
} else {
    # 首次生成：获取全部提交
    git log --oneline --no-merges --format="%s"
}
```

需要获取每个提交的完整 Conventional Commit 消息用于分类。

### 6.2 CHANGELOG.md 格式规范

采用 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 风格：

- 使用 `# Changelog` 作为一级标题
- 每个版本使用 `## [版本号] - YYYY-MM-DD` 作为二级标题
- 分组标题使用 `### Added`、`### Changed`、`### Fixed`、`### Removed` 等

**Conventional Commit type → CHANGELOG 分组映射**：

| type | CHANGELOG 分组 |
|------|---------------|
| `feat` | **Added** |
| `fix` | **Fixed** |
| `refactor` / `perf` / `style` | **Changed** |
| `docs` | **Documentation** |
| `build` / `ci` / `chore` / `test` | **Changed**（或视情况省略） |
| 删除文件的提交 | **Removed** |

模板：

```markdown
# Changelog

## [1.2.0] - 2024-01-15

### Added
- feat(auth): 添加 OAuth2 登录支持
- feat(dashboard): 新增数据看板页面

### Changed
- refactor(api): 重构用户接口返回格式

### Fixed
- fix(login): 修复登录超时问题
- fix(cache): 修复 Redis 连接泄漏

### Removed
- 移除废弃的旧版 API v1
```

### 6.3 首次生成

当项目不存在 CHANGELOG.md 时：

1. 获取所有历史提交（`git log --format="%s" --no-merges`）
2. 按 Conventional Commit type 分类到对应分组
3. 生成首个版本条目（版本号为当前版本号，日期为当前日期）
4. 写入 CHANGELOG.md（UTF-8 编码）

### 6.4 增量追加

当项目已有 CHANGELOG.md 时：

1. 读取现有 CHANGELOG.md 全部内容
2. 在 `# Changelog` 标题后的第一行（即第一个 `##` 版本条目之前）插入新版本条目
3. 新版本条目仅包含自上次 tag/release 以来新增的提交
4. 写回文件（UTF-8 编码）

```powershell
$existing = Get-Content "CHANGELOG.md" -Raw -Encoding UTF8
$newEntry = @"
## [$newVersion] - $todayDate

### $group
- $item1
- $item2

"@
# 在 # Changelog 标题后插入
$updated = $existing -replace "(# Changelog\r?\n\r?\n)", "`$1$newEntry"
Set-Content -Path "CHANGELOG.md" -Value $updated -Encoding UTF8
```

---

## 七、Release 发布

### 7.1 GitHub Release 创建

完整的 Release 创建流程：

```powershell
# 1. 创建 git tag
$version = "1.2.0"
git tag -a "v$version" -m "Release v$version"

# 2. 提取 CHANGELOG 中对应版本的发布说明
if (Test-Path "CHANGELOG.md") {
    $changelog = Get-Content "CHANGELOG.md" -Raw -Encoding UTF8
    # 提取 ## [$version] 到下一个 ## 之间的内容作为 release notes
    $pattern = "## \[$version\].*?(?=## \[|\Z)"
    $match = [regex]::Match($changelog, $pattern, [System.Text.RegularExpressions.RegexOptions]::Singleline)
    if ($match.Success) {
        $releaseNotes = $match.Value.Trim()
    } else {
        $releaseNotes = "Release v$version"
    }
} else {
    $releaseNotes = "Release v$version"
}

# 3. 推送 tag 到 GitHub
git push github "v$version"

# 4. 创建 GitHub Release
if ($isDraft) {
    gh release create "v$version" --title "v$version" --notes $releaseNotes --draft
} elseif ($isPrerelease) {
    gh release create "v$version" --title "v$version" --notes $releaseNotes --prerelease
} else {
    gh release create "v$version" --title "v$version" --notes $releaseNotes
}
```

### 7.2 同步 Tag 到 Gitee

根据平台选择决定是否执行：

```powershell
if ($target -eq "both" -or $target -eq "gitee") {
    git push gitee "v$version"
}
```

### 7.3 平台选择与 Release 行为

| 用户选择 | Release 行为 |
|----------|-------------|
| **双端推送** | 创建 GitHub Release + push tag 到 Gitee |
| **仅 GitHub** | 仅创建 GitHub Release，不推送到 Gitee |
| **仅 Gitee** | 尝试创建 GitHub Release（如 gh 已登录）+ push tag 到 Gitee；若 gh 不可用则仅 push tag 到 Gitee |

**重要提示**：gh CLI 只能创建 GitHub Release。Gitee 没有官方 CLI，Release 信息只能通过 Git tag 传递。Agent 应在 Release 完成后告知用户：

> Gitee Release 需要手动在网页端创建。Tag `v1.2.0` 已推送到 Gitee，您可以在 Gitee 仓库页面基于此 Tag 创建 Release。

### 7.4 Pre-release 和 Draft

- `--prerelease`：创建 GitHub Pre-release（标记为非稳定版本）
- `--draft`：创建 GitHub Draft Release（不公开发布，需手动发布）
- 可同时使用：`--prerelease --draft`

---

## 八、双平台端到端测试验证

### 8.1 --test 模式概述

`--test` 模式用于验证整个工作流在双平台环境下的可用性。它会创建临时测试仓库，执行完整的推送和 Release 流程，验证成功后自动清理。

### 8.2 测试流程步骤

**步骤1：创建 GitHub 测试仓库**

```powershell
$timestamp = Get-Date -Format "yyyyMMddHHmmss"
$testRepoName = "test-universal-git-workflow-" + $timestamp
gh repo create $testRepoName --private --clone
Set-Location $testRepoName
```

如果 `gh repo create` 失败，检查 gh auth 状态并报告错误。

**步骤2：初始化测试项目**

```powershell
git init

# 创建 README.md
@"
# Test Project

universal-git-workflow 端到端测试项目
"@ | Set-Content -Path "README.md" -Encoding UTF8

# 创建 VERSION 文件
Set-Content -Path "VERSION" -Value "0.1.0" -Encoding UTF8

# 创建 .gitignore
@"
node_modules/
.env
*.log
"@ | Set-Content -Path ".gitignore" -Encoding UTF8

git add -A
git commit -m "chore: initial commit for test"
```

**步骤3：在 Gitee 创建测试仓库**

Gitee 没有 CLI 工具。Agent 应输出以下提示信息，让用户手动处理：

```
========================================
请在 Gitee 上创建测试仓库：

仓库名称：test-universal-git-workflow-<timestamp>
访问 https://gitee.com/<你的Gitee用户名>/<仓库名>
或者通过 API 创建：

curl -X POST "https://gitee.com/api/v5/user/repos" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-universal-git-workflow-<timestamp>","private":"true"}'

创建完成后，请确认以便继续测试。
========================================
```

如果用户确认已创建，添加 remote：

```powershell
# 需要用户提供 Gitee 用户名
$giteeUsername = "<用户输入的 Gitee 用户名>"
git remote add gitee "https://gitee.com/$giteeUsername/$testRepoName.git"
```

**步骤4：测试双平台推送**

```powershell
Write-Host "--- 测试推送到 GitHub ---"
git push -u github main
if ($LASTEXITCODE -ne 0) {
    Write-Error "GitHub 推送失败"
    exit 1
}
Write-Host "GitHub 推送成功" -ForegroundColor Green

Write-Host "--- 测试推送到 Gitee ---"
git push -u gitee main
if ($LASTEXITCODE -ne 0) {
    Write-Error "Gitee 推送失败"
    exit 1
}
Write-Host "Gitee 推送成功" -ForegroundColor Green
```

**步骤5：验证 Release 功能**

```powershell
Write-Host "--- 测试 Release 功能 ---"

# 创建 tag
git tag -a "v0.1.0" -m "Test release v0.1.0"

# 推送 tag 到两个平台
git push github v0.1.0
git push gitee v0.1.0

# 创建 GitHub Release
gh release create "v0.1.0" --title "v0.1.0" --notes "Test release from universal-git-workflow test"

Write-Host "Release 测试完成" -ForegroundColor Green
```

**步骤6：清理测试环境**

```powershell
Write-Host "--- 清理测试环境 ---"

# 退出测试仓库目录
Set-Location ..

# 删除 GitHub 测试仓库
gh repo delete $testRepoName --yes
Write-Host "已删除 GitHub 测试仓库: $testRepoName"

# 删除本地测试目录
Remove-Item -Recurse -Force $testRepoName
Write-Host "已删除本地测试目录: $testRepoName"

# 提示手动操作
Write-Host @"

========================================
请手动完成以下清理操作：

1. 登录 Gitee 网页端
2. 删除测试仓库: $testRepoName
   URL: https://gitee.com/<用户名>/$testRepoName/settings
========================================

"@ -ForegroundColor Yellow
```

由于 Gitee 无 CLI，Agent 必须输出明确的提示信息，告知用户去 Gitee 网页端删除测试仓库。

### 8.3 测试失败处理

测试过程中任何步骤失败时：

1. 报告具体失败的步骤和错误信息
2. **继续执行清理操作**（删除 GitHub 仓库和本地文件）
3. 不因测试失败留下垃圾仓库

```powershell
try {
    # 测试流程...
} catch {
    Write-Error "测试失败: $_"
} finally {
    # 始终执行清理
    Set-Location $originalDir
    if (Test-Path $testRepoName) {
        gh repo delete $testRepoName --yes 2>$null
        Remove-Item -Recurse -Force $testRepoName -ErrorAction SilentlyContinue
    }
}
```

---

## 九、完整工作流编排

### 9.1 --full 一键模式

执行顺序（每步失败即停止并报告错误）：

```
1. UTF-8 初始化（始终执行）
   ↓
2. 检测 remote + 平台选择（双平台则交互询问）
   ↓
3. 安全 Pull（fetch remote 变更，检查工作区状态）
   ↓
4. 分析变更（git status + git diff）
   ↓
5. 生成 Conventional Commit（推断 type/scope，生成消息）
   ↓
6. 提交（git add + git commit）
   ↓
7. 更新版本号（根据提交类型自动递增或手动指定）
   ↓
8. 生成/更新 CHANGELOG
   ↓
9. 推送代码（按平台选择推送到对应 remote）
   ↓
10. 创建 Release（按平台选择创建 + 推送 tag）
```

每一步执行后检查 `$LASTEXITCODE`，失败则输出错误信息并中止流程。

### 9.2 子功能独立执行

| 参数 | 功能 | 说明 |
|------|------|------|
| `--full` | 一键全流程 | 执行全部 10 步 |
| `--pull-only` | 仅安全 Pull | 执行步骤 1-3 |
| `--commit-only` | 仅分析并提交 | 执行步骤 4-6，不推送不发布 |
| `--version-only` | 仅更新版本号 | 执行步骤 7 |
| `--changelog-only` | 仅生成 CHANGELOG | 执行步骤 8 |
| `--release-only` | 仅创建 Release | 执行步骤 10（前提是已有 tag） |
| `--push-only` | 仅推送代码 | 执行步骤 9（含平台选择） |
| `--test` | 端到端测试 | 执行完整的双平台测试流程（见第八节） |

### 9.3 常见故障处理

| 故障现象 | 可能原因 | 处理方案 |
|----------|----------|----------|
| `push rejected` | 远程有新提交 | 先 `git fetch` → `git rebase` 或 `git merge` → 重新 `git push` |
| `gh auth token expired` | GitHub Token 过期 | 运行 `gh auth refresh` 刷新或 `gh auth login` 重新登录 |
| `remote not found` | remote 地址错误或不存在 | 运行 `git remote -v` 检查，用 `git remote set-url <name> <correct-url>` 修正 |
| `merge conflict` | 本地与远程代码冲突 | 列出冲突文件（`git diff --name-only --diff-filter=U`），手动解决后 `git merge --continue` |
| `gh: command not found` | 未安装 GitHub CLI | 提示安装：`winget install GitHub.cli` 或访问 https://cli.github.com/ |
| `permission denied (Gitee)` | Gitee 凭据过期或错误 | 重新输入 Gitee 密码或更新 Windows Credential Manager 中的凭据 |
| `fatal: The current branch has no upstream branch` | 新分支未设置跟踪关系 | `git push -u <remote> <branch>` 设置上游跟踪 |
| `fatal: not a git repository` | 不在 Git 仓库中 | 检查当前目录，提示用户进入正确的仓库目录或 `git init` |

对于每个故障，Agent 应：
1. 清晰描述故障现象
2. 给出具体可执行的修复命令
3. 修复后自动重试失败的操作

---

## 十、安全守则

以下是 Agent 在执行任何 Git 操作时必须遵循的强制规则：

### 绝对禁止的操作

1. **绝不** 对 `main` / `master` 分支执行 `git push --force`
2. **绝不** 在未经用户明确确认的情况下执行 `git push --force`（即使是非保护分支）
3. **绝不** 修改用户现有的 git config（`user.name`、`user.email`、`remote` URL 等）
4. **绝不** 跳过 git hooks（`--no-verify` / `--no-gpg-sign`），除非用户明确要求
5. **绝不** 在未经用户确认的情况下删除仓库（包括测试中创建的仓库，需先提示用户）
6. **绝不** 提交敏感文件（`.env`、`credentials.json`、`*.pem`、`*_rsa`、私钥、API 密钥等）

### 必须遵守的操作

7. **提交前** 必须检查 staged files，排除敏感文件：

```powershell
# 检查是否有敏感文件
$staged = git diff --staged --name-only
$sensitivePatterns = @("\.env$", "credentials", "\.pem$", "privatekey", "secret")
foreach ($pattern in $sensitivePatterns) {
    $matches = $staged | Select-String $pattern
    if ($matches) {
        Write-Error "发现疑似敏感文件: $matches"
        Write-Error "请确认这些文件是否应该提交"
        exit 1
    }
}
```

8. **推送前** 始终先 fetch 确认远程状态
9. **双端推送时** 若第一端失败，立即停止，不继续操作第二端
10. **Pull 前** 始终检查工作区状态，不干净时提醒用户
11. **版本号变更** 遵循 SemVer 规范，不跳跃版本号
12. **CHANGELOG** 使用 UTF-8 编码写入，确保中文字符正确
13. **Tag 操作** 使用带注释的 tag（`git tag -a`），不创建轻量 tag
14. **错误处理** 每一步操作后检查 `$LASTEXITCODE`，非零时报告具体错误

---

## 附录 A：环境信息参考

本 Skill 设计基于以下环境假设，Agent 应了解这些信息以正确执行操作：

- **当前用户**：Allen528
- **Git 邮箱**：272416939@qq.com
- **GitHub 用户**：272416939
- **GitHub Token 权限**：gist、read:org、repo、workflow
- **操作系统**：Windows (PowerShell)
- **Shell**：PowerShell 5.1+ 或 PowerShell Core 7+

## 附录 B：关键命令速查

| 场景 | 命令 |
|------|------|
| 检测平台 | `git remote -v` |
| 检查工作区 | `git status --porcelain` |
| 获取远程变更 | `git fetch <remote>` |
| 查看差异 | `git diff HEAD..<remote>/<branch> --stat` |
| 查看远程提交 | `git log HEAD..<remote>/<branch> --oneline` |
| 分析变更 | `git diff --staged` 或 `git diff` |
| 查看提交历史 | `git log --oneline --no-merges` |
| 上次 tag | `git describe --tags --abbrev=0 2>$null` |
| 创建 tag | `git tag -a "v1.0.0" -m "Release v1.0.0"` |
| 推送 tag | `git push <remote> v1.0.0` |
| 创建 Release | `gh release create "v1.0.0" --title "v1.0.0" --notes "..."` |
| 刷新 gh 登录 | `gh auth refresh` 或 `gh auth login` |
| UTF-8 初始化 | `chcp 65001; $OutputEncoding = [System.Text.UTF8Encoding]::new()` |
