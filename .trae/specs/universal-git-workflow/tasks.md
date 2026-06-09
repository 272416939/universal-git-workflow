# 通用 Git 工作流技能 任务清单

- [x] Task 1: 创建 Skill 目录结构与元数据
  - [x] 1.1 创建 `.trae/skills/universal-git-workflow/` 目录
  - [x] 1.2 编写 SKILL.md frontmatter（name、description）

- [x] Task 2: 编写双平台 Remote 检测与平台选择交互指南
  - [x] 2.1 编写 GitHub + Gitee remote 自动检测逻辑说明（通过 `git remote -v` 解析 URL 判定平台）
  - [x] 2.2 编写交互式平台选择流程：同时有双平台时询问用户选 both / github / gitee
  - [x] 2.3 编写 `--target` 参数跳过交互直接指定平台的说明
  - [x] 2.4 覆盖单平台、双平台、无 remote 三种场景的完整流程

- [x] Task 3: 编写 UTF-8 编码安全保障指南
  - [x] 3.1 编写 PowerShell 环境下 UTF-8 编码设置指令（chcp 65001、$OutputEncoding、$PSDefaultParameterValues）
  - [x] 3.2 编写 git 命令级别的 UTF-8 配置建议（`git config core.quotepath false` 等）
  - [x] 3.3 编写常见乱码场景与排查步骤（commit 消息乱码、文件内容乱码、log 输出乱码）

- [x] Task 4: 编写安全 Pull 工作流指南
  - [x] 4.1 编写分步 pull 流程（git fetch → git diff HEAD..@{u} → git merge/rebase）
  - [x] 4.2 编写冲突检测与处理说明（列出冲突文件、给出 resolve 命令）
  - [x] 4.3 编写 dirty worktree 处理（stash vs commit 提示，禁止强拉）

- [x] Task 5: 编写 Conventional Commit 自动提交指南
  - [x] 5.1 集成 git-commit Skill 的 type/scope 分析规则（通过 git diff 推断变更类型）
  - [x] 5.2 编写 staged/unstaged 不同场景的处理流程
  - [x] 5.3 编写 breaking change（`!`）检测规则

- [x] Task 6: 编写语义化版本号管理指南
  - [x] 6.1 编写版本号自动递增规则（feat!→MAJOR, feat→MINOR, fix→PATCH）
  - [x] 6.2 编写版本号持久化方案（VERSION 文件，同时支持从 package.json / Cargo.toml 读取）
  - [x] 6.3 编写手动指定版本号（`--version`）与预发布后缀（`--prerelease alpha`）的支持说明

- [x] Task 7: 编写 CHANGELOG 自动生成指南
  - [x] 7.1 编写从 git log 提取 Conventional Commits 的规则（`git log --grep` 按类型筛选）
  - [x] 7.2 编写 CHANGELOG.md 格式规范（Keep a Changelog 风格，按 Added/Changed/Fixed 分组）
  - [x] 7.3 编写首次生成与增量追加两种场景的处理方式

- [x] Task 8: 编写 Release 发布指南（含平台选择）
  - [x] 8.1 编写 GitHub Release 创建流程（`gh release create --notes-file CHANGELOG.md`）
  - [x] 8.2 编写通过 `git push gitee <tag>` 同步 tag 到 Gitee 的流程
  - [x] 8.3 编写 pre-release / draft release 支持
  - [x] 8.4 编写根据用户平台选择决定是否跳过 Gitee/仅发布 GitHub

- [x] Task 9: 编写双平台端到端测试验证指南
  - [x] 9.1 编写 `--test` 模式完整流程说明：创建临时仓库 → 推送验证 → 清理
  - [x] 9.2 编写 GitHub 测试仓库创建与删除方法（`gh repo create` / `gh repo delete`）
  - [x] 9.3 编写 Gitee 测试仓库创建指引（通过 Web API 或手动创建说明）
  - [x] 9.4 编写测试失败时的清理流程（确保不留垃圾仓库）

- [x] Task 10: 编写完整工作流编排与故障恢复指南
  - [x] 10.1 编写 `--full` 一键流程串联说明（含平台选择交互插入点）
  - [x] 10.2 编写各步骤独立执行的使用说明（`--pull-only`、`--commit-only`、`--release-only` 等）
  - [x] 10.3 编写常见故障处理（push rejected、token expired、remote not found、merge conflict）

# Task Dependencies
- Task 3 ~ Task 9 无相互依赖，可并行编写
- Task 2 需优先完成（后续平台选择逻辑多处引用）
- Task 10 依赖 Task 2 ~ Task 9 完成后再编写（汇总编排）
