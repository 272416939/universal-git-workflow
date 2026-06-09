# universal-git-workflow

通用 Git 工作流 Skill，为 AI Agent 提供 GitHub + Gitee 双平台完整开发工作流自动化支持。

[![GitHub Release](https://img.shields.io/badge/GitHub-v0.1.1-blue)](https://github.com/272416939/universal-git-workflow/releases)
[![Gitee](https://img.shields.io/badge/Gitee-v0.1.1-red)](https://gitee.com/Allen0528/universal-git-workflow)

## 功能

| 功能 | 说明 |
|------|------|
| **双平台 Remote 检测** | 自动识别 GitHub + Gitee remote，支持交互式平台选择 |
| **UTF-8 编码安全** | PowerShell 环境下自动配置 UTF-8，避免中文乱码 |
| **安全 Pull** | 分步 fetch → diff → merge/rebase，冲突检测与处理 |
| **Conventional Commits** | 基于 diff 自动推断 type/scope，生成规范提交消息 |
| **语义化版本号** | MAJOR.MINOR.PATCH 自动递增，支持手动指定和预发布 |
| **CHANGELOG 生成** | 从提交历史自动生成 Keep a Changelog 风格更新日志 |
| **Release 发布** | GitHub Release 自动创建，Gitee tag 同步推送 |
| **完整工作流** | `--full` 一键从 pull 到发布全流程 |
| **端到端测试** | `--test` 双平台临时仓库创建、推送验证、自动清理 |
| **Gitee API** | 自动创建仓库和 Release，无需手动操作 |
| **隐私过滤** | 敏感文件/Token/密码自动检测脱敏，防泄露 |

## 使用

在 TRAE 中通过 Skill 调用：

```
Use Skill: universal-git-workflow
```

### 常用模式

```
# 一键完整流程（pull → commit → version → changelog → push → release）
--full

# 仅安全拉取
--pull-only

# 仅提交变更
--commit-only

# 仅更新版本号
--version-only

# 仅生成更新日志
--changelog-only

# 仅推送代码（含平台选择）
--push-only

# 仅创建 Release
--release-only

# 指定版本号
--version 1.2.0

# 预发布版本
--prerelease alpha

# 指定推送目标
--target both
--target github
--target gitee

# 端到端测试
--test
```

## 环境要求

- Git
- GitHub CLI (`gh`) 已登录
- Gitee PAT（Personal Access Token，用于 API 操作）
- PowerShell (Windows) 或 pwsh

## 安装

将本仓库根目录的 `SKILL.md` 复制到你项目 `.trae/skills/universal-git-workflow/` 目录下，或直接通过 TRAE Skill 市场安装。

## License

MIT
