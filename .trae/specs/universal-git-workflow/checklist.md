# 通用 Git 工作流技能 检查清单

## 结构验证
- [x] SKILL.md 文件存在于 `.trae/skills/universal-git-workflow/SKILL.md`
- [x] SKILL.md frontmatter 包含有效的 `name` 和 `description` 字段
- [x] SKILL.md 正文内容完整、可读

## 功能覆盖
- [x] 双平台 Remote 检测：文档包含 GitHub + Gitee remote 检测与区分逻辑
- [x] 平台推送选择交互：文档包含双平台时询问用户选择推送目标的交互流程
- [x] `--target` 参数：文档包含通过参数跳过交互直接指定平台的说明
- [x] UTF-8 编码：文档包含 PowerShell 环境下 UTF-8 编码设置指令
- [x] 安全 Pull：文档包含分步 pull 流程（fetch → diff → merge/rebase）
- [x] 冲突处理：文档包含冲突检测与处理说明
- [x] Conventional Commit：文档包含基于 diff 分析的自动提交消息生成规则
- [x] 版本号管理：文档包含语义化版本号自动递增规则
- [x] CHANGELOG 生成：文档包含从提交记录生成更新日志的规则
- [x] Release 发布：文档包含 gh release create 及 Gitee 同步推送流程
- [x] 完整工作流：文档包含一键 `--full` 模式的使用说明
- [x] 双平台测试：文档包含 `--test` 模式创建临时仓库、推送验证、清理的完整流程
- [x] GitHub 测试仓库清理：文档包含 gh repo delete 自动清理说明
- [x] Gitee 测试仓库清理：文档包含 Gitee 测试仓库删除指引
- [x] 故障恢复：文档包含常见错误场景的处理建议
