# Changelog

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
