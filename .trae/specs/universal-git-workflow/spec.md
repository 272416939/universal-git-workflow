# 通用 Git 工作流技能 (universal-git-workflow) Spec

## Why
当前开发环境中已配置 GitHub (gh CLI) 和 Gitee (git credential) 双平台，但在日常开发中存在以下痛点：PowerShell 下 git 命令编码乱码、pull 操作因分支追踪/冲突频繁失败、提交信息不规范、版本号和更新日志手动维护繁琐、发布流程需多次手动操作。需要一个统一的 Skill 封装这些操作，提供一键式安全、规范的 Git 工作流。

## What Changes
- 新增 `.trae/skills/universal-git-workflow/SKILL.md` Skill 定义文件
- 该 Skill 封装以下能力：
  - 双平台 (GitHub + Gitee) remote 自动检测与管理
  - 交互式平台推送选择（每次操作可决定双端推送或指定单平台）
  - UTF-8 编码安全保障（解决 PowerShell 乱码）
  - 安全的 pull 流程（fetch → diff → merge/rebase，冲突检测）
  - 基于 Conventional Commits 的自动提交
  - 语义化版本号自动递增
  - 从提交记录自动生成更新日志 (CHANGELOG.md)
  - 通过 gh CLI 创建 GitHub Release，同步推送 Gitee
  - 双平台端到端验证测试（创建测试仓库 → 推送验证 → 清理删除）

## Impact
- Affected specs: 无（新建项目）
- Affected code: `.trae/skills/universal-git-workflow/SKILL.md`

## ADDED Requirements

### Requirement: 双平台 Remote 检测
系统 SHALL 自动检测当前仓库配置的 GitHub 和 Gitee remote，支持一个或两个平台同时存在。

#### Scenario: 检测到双平台 remote
- **WHEN** 仓库同时配置了 GitHub 和 Gitee 的 remote
- **THEN** 系统识别两个 remote 及其平台归属
- **AND** 系统进入交互式平台选择流程（见"平台推送目标选择交互"需求）

#### Scenario: 仅检测到单一平台 remote
- **WHEN** 仓库仅配置了 GitHub 或仅 Gitee 的 remote
- **THEN** 系统仅操作该单一平台，不报错，跳过平台选择交互

#### Scenario: 未检测到任何 remote
- **WHEN** 仓库未配置任何 remote
- **THEN** 系统提示用户先配置 remote，中止后续操作

### Requirement: 平台推送目标选择交互
当仓库同时配置了 GitHub 和 Gitee remote 时，系统 SHALL 在推送/发布操作前询问用户选择目标平台。

#### Scenario: 用户选择双端推送
- **WHEN** 系统检测到双平台 remote 并询问推送目标
- **AND** 用户选择双端推送 (both)
- **THEN** 系统依次向 GitHub 和 Gitee 推送，任一失败则立即报错并停止

#### Scenario: 用户选择仅 GitHub
- **WHEN** 系统检测到双平台 remote 并询问推送目标
- **AND** 用户选择仅推送到 GitHub
- **THEN** 系统仅向 GitHub 推送，跳过 Gitee

#### Scenario: 用户选择仅 Gitee
- **WHEN** 系统检测到双平台 remote 并询问推送目标
- **AND** 用户选择仅推送到 Gitee
- **THEN** 系统仅向 Gitee 推送，跳过 GitHub

#### Scenario: 用户为未来操作记忆偏好
- **WHEN** 用户指定 `--target github` / `--target gitee` / `--target both` 参数
- **THEN** 系统跳过交互询问，直接按指定目标执行

### Requirement: UTF-8 编码安全
系统 SHALL 确保所有 git 操作在 PowerShell 环境下使用 UTF-8 编码，避免中文乱码。

#### Scenario: PowerShell 环境下的 git 命令输出
- **WHEN** 在 PowerShell 中执行 git log / git status / git diff 等命令
- **THEN** 输出中的中文字符正确显示，无乱码
- **AND** 系统在执行 git 命令前设置 `chcp 65001`、`$OutputEncoding = [System.Text.UTF8Encoding]::new()` 等 UTF-8 环境变量

### Requirement: 安全 Pull 工作流
系统 SHALL 执行安全的分步 pull 操作，避免直接 `git pull` 导致的意外合并或冲突。

#### Scenario: 正常无冲突 pull
- **WHEN** 远程有新提交且本地修改已提交
- **THEN** 先 fetch 远程变更，检测无冲突后执行 merge/rebase

#### Scenario: 本地有未提交修改
- **WHEN** 工作区有未暂存的修改时触发 pull
- **THEN** 系统先提示用户处理本地修改（stash 或 commit），不强行 pull

#### Scenario: pull 发生冲突
- **WHEN** fetch 后发现合并会产生冲突
- **THEN** 系统明确提示冲突文件列表，给出解决建议，不静默失败

### Requirement: Conventional Commits 自动提交
系统 SHALL 基于 git-commit Skill 规范，分析 diff 内容自动生成 Conventional Commit 消息。

#### Scenario: 有暂存变更
- **WHEN** 用户已 `git add` 文件并请求提交
- **THEN** 系统分析 staged diff，推断 type 和 scope，生成 `<type>(<scope>): <description>` 格式的提交消息

#### Scenario: 无暂存变更
- **WHEN** 工作区有修改但未 staged
- **THEN** 系统询问是否自动 stage 所有变更，然后生成提交消息

### Requirement: 语义化版本号管理
系统 SHALL 支持语义化版本号 (MAJOR.MINOR.PATCH) 的自动递增与持久化。

#### Scenario: 根据提交类型自动递增版本号
- **WHEN** 提交类型为 `feat` 且有 `!` 标记（破坏性变更）
- **THEN** MAJOR 版本号 +1，MINOR 和 PATCH 归零
- **WHEN** 提交类型为 `feat`（非破坏性）
- **THEN** MINOR 版本号 +1，PATCH 归零
- **WHEN** 提交类型为 `fix` / `perf` / `refactor` 等
- **THEN** PATCH 版本号 +1

#### Scenario: 手动指定版本号
- **WHEN** 用户显式指定版本号（如 `--version 2.0.0`）
- **THEN** 使用用户指定的版本号，不做自动递增

### Requirement: 更新日志自动生成
系统 SHALL 基于 git 提交历史自动生成和维护 CHANGELOG.md 文件。

#### Scenario: 首次生成 CHANGELOG
- **WHEN** 项目尚不存在 CHANGELOG.md
- **THEN** 系统创建 CHANGELOG.md，包含当前版本的所有提交记录，按 Conventional Commit 类型分类展示

#### Scenario: 追加更新日志
- **WHEN** 项目已有 CHANGELOG.md
- **THEN** 系统在文件顶部插入新版本条目，按类型分组列出新增提交

### Requirement: 发行版发布
系统 SHALL 通过 gh CLI 创建 GitHub Release，并根据用户选择同步推送 tag 至 Gitee。

#### Scenario: 创建正式 Release 并双端发布
- **WHEN** 用户请求发布新版本且选择双端推送
- **THEN** 系统创建 git tag，通过 `gh release create` 发布 GitHub Release（含 CHANGELOG 内容），并将 tag push 到 Gitee

#### Scenario: 创建预发布版本
- **WHEN** 用户指定 `--prerelease` 选项
- **THEN** 系统创建 GitHub Pre-release，版本号带 `-alpha` / `-beta` / `-rc` 后缀

#### Scenario: 仅 GitHub 发布
- **WHEN** 用户选择仅发布到 GitHub
- **THEN** 系统仅创建 GitHub Release，跳过 Gitee 推送

### Requirement: 一键完整工作流
系统 SHALL 支持从提交到发布的完整一键工作流。

#### Scenario: 执行完整工作流
- **WHEN** 用户触发完整工作流（`--full` 模式）
- **THEN** 系统依次执行：安全 pull → 分析变更 → 平台选择交互 → 生成 commit → 更新版本号 → 生成/更新 CHANGELOG → 按选择推送代码 → 按选择创建 Release

#### Scenario: 单步执行
- **WHEN** 用户仅触发子功能（如只提交、只发布）
- **THEN** 系统仅执行该子功能，不执行其他步骤

### Requirement: 双平台端到端验证测试
系统 SHALL 提供自动化测试能力，在 GitHub 和 Gitee 创建临时测试仓库、推送验证，完成后自动清理。

#### Scenario: 执行双平台验证测试
- **WHEN** 用户触发测试模式（`--test`）
- **THEN** 系统依次执行：
  1. 在 GitHub 创建临时私有仓库（通过 `gh repo create`）
  2. 在 Gitee 创建临时私有仓库
  3. 初始化本地测试项目，添加双 remote
  4. 提交测试内容并推送至两个平台
  5. 验证双端推送是否成功
  6. 通过 `gh repo delete` 删除 GitHub 测试仓库
  7. 提示用户手动删除 Gitee 测试仓库（或提供删除指引）
  8. 清理本地临时文件

#### Scenario: 测试推送失败
- **WHEN** 任一平台推送失败
- **THEN** 系统报告具体失败平台和原因，继续进行清理操作（删除已创建的测试仓库）

### Requirement: 隐私过滤与敏感信息保护
系统 SHALL 在提交、推送、生成文档等环节自动检测和过滤敏感信息，防止凭据、Token、私人数据泄露到公开仓库。

#### Scenario: 提交前敏感文件扫描
- **WHEN** Agent 执行 `git add` 准备提交
- **THEN** 系统检测 staged files 中是否包含敏感文件（`.env`、`credentials.*`、`*.token`、`*.pem`、`*.db`、`bak/` 等）
- **AND** 发现敏感文件时阻止提交并明确提示文件路径

#### Scenario: 提交消息脱敏检查
- **WHEN** Agent 生成 Conventional Commit 消息
- **THEN** 系统检查消息中是否包含 Token 模式（`ghp_***`、`gho_***` 等）、密码、手机号等
- **AND** 发现敏感内容时阻止提交并要求修改消息

#### Scenario: CHANGELOG/Release Notes 脱敏
- **WHEN** 生成 CHANGELOG.md 或 Release Notes
- **THEN** 系统扫描内容中的敏感模式并自动替换为脱敏占位符（如 `<TOKEN_MASKED>`）

#### Scenario: .gitignore 自动管理
- **WHEN** 项目不存在 `.gitignore` 或缺少必要的敏感条目（如 `.trae/`、`*.bak`、`.env` 等）
- **THEN** 系统主动建议创建或补充 `.gitignore`

#### Scenario: 敏感信息泄露补救
- **WHEN** 发现敏感信息已被提交或推送
- **THEN** 系统告知用户具体泄露的 commit 和内容类型
- **AND** 提供 `git filter-branch` 或 BFG 清理方案
- **AND** 提醒用户在平台上撤销并重新生成泄露的 Token/密码

## MODIFIED Requirements
无（新建项目）

## REMOVED Requirements
无（新建项目）
