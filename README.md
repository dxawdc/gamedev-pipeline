# Godot 4 全自动游戏开发流水线

> VSCode + Claude Code 插件
> 放入 Word 策划文档，其余全自动。支持任意游戏类型。

---

## 两步启动

### 第一步：放入策划文档

把 `.docx` 文件复制到 `design/` 目录，文件名随意：

```
gamedev-pipeline/
└── design/
    └── 你的游戏策划案.docx
```

### 第二步：在 Claude Code 面板粘贴启动提示词

用 VSCode 打开 `gamedev-pipeline/` 文件夹，点击侧边栏 Claude 图标，复制以下内容发送：

```
请读取 .claude/CLAUDE.md，然后按顺序依次读取并执行以下 Agent：

Agent 00：读取并执行 .claude/agents/00_word_to_md.md
  → 将 design/ 中的 .docx 转换为 design/gdd.md

Agent 01：读取并执行 .claude/agents/01_bootstrap.md
  → 解析 design/gdd.md，生成 tasks/project_context.json

Agent 02：读取并执行 .claude/agents/02_lead_designer.md
  → 为每个核心系统输出九章完整系统策划案到 design/systems/

Agent 03：读取并执行 .claude/agents/03_planner.md
  → 跨系统一致性校验，输出 design/gdd_refined.md

Agent 04：读取并执行 .claude/agents/04_architect.md
  → 技术架构设计，输出 tasks/backlog.json

Agent 05：读取并执行 .claude/agents/05_art.md
  → 美术需求分析，输出 reports/art_requirements.md

Agent 06：读取并执行 .claude/agents/06_developer.md
  → 循环实现 backlog 中所有 pending 任务，每次写入 .gd 文件后立即执行：
    godot --headless --path ./godot_project --quit
    有 ERROR 则停止修复，通过后继续

Agent 07：读取并执行 .claude/agents/07_tester.md
  → 编写并运行 GUT 测试，修复失败项直到 100% 通过

Agent 08：读取并执行 .claude/agents/08_builder.md
  → 配置 project.godot，打包 builds/game.pck

Agent 09：读取并执行 .claude/agents/09_git.md
  → 检查测试和构建均通过后，初始化本地 Git 仓库（首次）或生成迭代提交
  → 打语义化版本 Tag，仅做本地管理，不推送远程

Agent 10：读取并执行 .claude/agents/10_reporter.md
  → 生成 reports/final_report.md

每个 Agent 完成后打印摘要，再继续下一个。
```

---

## 中途中断后续跑

| 情况 | 发送指令 |
|------|---------|
| Word 还没转换 | `读取并执行 .claude/agents/00_word_to_md.md` |
| 重新解析 GDD | `读取并执行 .claude/agents/01_bootstrap.md` |
| 重新生成系统策划案 | `读取并执行 .claude/agents/02_lead_designer.md` |
| 重新校验策划 | `读取并执行 .claude/agents/03_planner.md` |
| 重新生成架构和任务 | `读取并执行 .claude/agents/04_architect.md` |
| 开发中途中断 | `读取 tasks/project_context.json 和 .claude/agents/06_developer.md，继续实现所有 pending 任务（已完成的跳过）` |
| 只跑测试 | `读取并执行 .claude/agents/07_tester.md` |
| 重新构建 | `读取并执行 .claude/agents/08_builder.md` |
| 重新提交版本 | `读取并执行 .claude/agents/09_git.md` |
| 重新生成报告 | `读取并执行 .claude/agents/10_reporter.md` |

---

## Godot 命令行配置（只需做一次）

1. 将 `Godot_v4.6.2-stable_win64.exe` 重命名为 `godot.exe`
2. 放到 `C:\Godot\`
3. 将 `C:\Godot` 加入系统 PATH 环境变量
4. 重启 VSCode，在终端验证：`godot --version`

---

## 换新游戏

1. 将新的 `.docx` 放入 `design/`（删掉旧的）
2. 删除 `tasks/project_context.json`、`tasks/backlog.json`、`godot_project/`、`design/systems/`
3. 重新发送启动提示词

`.claude/agents/` 中的文件无需改动，对所有游戏类型通用。

---

## 项目结构

```
gamedev-pipeline/
├── .claude/
│   ├── CLAUDE.md                 全局规范
│   └── agents/
│       ├── 00_word_to_md.md      Word → Markdown
│       ├── 01_bootstrap.md       GDD 解析
│       ├── 02_lead_designer.md   主策划（九章系统策划案）
│       ├── 03_planner.md         策划校验
│       ├── 04_architect.md       架构设计
│       ├── 05_art.md             美术需求
│       ├── 06_developer.md       功能开发
│       ├── 07_tester.md          自动测试
│       ├── 08_builder.md         构建导出
│       ├── 09_git.md             版本管理
│       └── 10_reporter.md        最终报告
├── design/
│   └── 你的策划案.docx            ← 放这里
├── tasks/                        自动生成
├── reports/                      自动生成
├── builds/                       自动生成
└── godot_project/                自动生成
```

---

## 常见问题

**Q：GDD 格式有要求吗？**
A：无固定格式。Agent 01 自动识别游戏类型，Agent 02 会在 `notes.gdd_gaps` 中记录不足之处并主动补全。

**Q：Word 里有表格和图片怎么办？**
A：表格会保留为 Markdown 格式，图片会被忽略（不影响文字信息）。转换后可在 `design/gdd.md` 核查。

**Q：流水线跑到一半停了？**
A：参考「中途中断后续跑」表格，从停止的阶段发送对应指令续跑。已完成的任务（backlog 中 status=done）不会重复执行。

**Q：API Error: 400 Invalid user_id？**
A：在系统环境变量中新增 `ANTHROPIC_USER_ID=user001`，重启 VSCode 后重试。
