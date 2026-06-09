# Changelog

## [0.1.1] - 2026-06-09

### Added
- feat(gitee): 新增 Gitee API 自动创建仓库和 Release 实战指南
  - Gitee API 端点速查表（创建/更新/删除仓库和 Release）
  - 通过 `curl.exe` + UTF-8 JSON 文件绕过 PowerShell 编码陷阱
  - Git credential PAT 持久化方法

### Fixed
- fix(encoding): 新增 PowerShell 编码陷阱章节（Invoke-RestMethod 乱码、here-string 语法、chcp 管道输出）
- fix(test): 测试流程改为 API 自动创建/删除 Gitee 仓库，不再依赖手动操作
- fix(faults): 补充 5 条常见故障（Gitee API 401、空仓库公开限制、重复 Release、Safe-RM 保护、tag 冲突）

### Changed
- docs(env): 新增附录 C：TRAE 环境限制（受保护目录、命令长度、curl 别名冲突）

## [0.1.0] - 2026-06-09

### Added
- feat(skill): 初始化 universal-git-workflow 通用 Git 工作流技能
  - 双平台 (GitHub + Gitee) remote 自动检测与平台选择交互
  - PowerShell 环境下 UTF-8 编码安全保障
  - 安全 Pull 工作流（分步 fetch → diff → merge/rebase）
  - 基于 Conventional Commits 的自动提交
  - 语义化版本号自动递增管理
  - 从提交记录自动生成更新日志 (CHANGELOG.md)
  - 通过 gh CLI 创建 GitHub Release，同步推送 Gitee
  - 完整工作流编排、子功能独立执行、常见故障恢复
  - 双平台端到端验证测试（`--test` 模式）
- feat(security): 新增全面隐私过滤和敏感信息保护规则
  - 敏感文件清单：`.trae/`、`bak/`、数据库文件、凭据文件等禁止提交
  - 敏感内容检测：Token/密码/手机号等提交消息脱敏检查
  - `.gitignore` 自动管理：检测缺失时自动建议创建
  - 泄露补救流程：已提交/已推送的敏感信息修复方案
  - CHANGELOG/Release Notes 脱敏检查
  - SKILL.md 移除硬编码凭据引用
