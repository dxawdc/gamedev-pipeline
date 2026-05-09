# Godot 4 全自动游戏开发流水线

> VSCode + Claude Code 插件
> 放入 Word 策划文档，其余全自动。支持任意游戏类型。
> 
![架构图](img src="https://github.com/dxawdc/gamedev-pipeline/blob/main/godot_pipeline_architecture.svg",width="50%")
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
  → 解析 GDD，生成 tasks/project_context.json 和 tasks/schema.json（数据契约）

Agent 02：读取并执行 .claude/agents/02_lead_designer.md
  → 按系统复杂度分批，输出九章系统策划案到 design/systems/
  → 注意：本 Agent 每批最多处理 3 个系统，可能需要多次触发直到全部完成

Agent 03：读取并执行 .claude/agents/03_planner.md
  → 跨系统一致性校验，矛盾直接回写到 systems/*.md，输出 design/gdd_refined.md

Agent 04：读取并执行 .claude/agents/04_architect.md
  → 技术架构设计，生成拓扑排序后的 tasks/backlog.json 和测试骨架文件

Agent 05：读取并执行 .claude/agents/05_art.md
  → 美术需求分析，生成 reports/art_requirements.md 和 PlaceholderAssets.gd

Agent 06：读取并执行 .claude/agents/06_developer.md
  → 循环实现 backlog 中所有 pending 任务，每次写入 .gd 文件后立即语法检查
  → 注意：本 Agent 每批最多执行 5 个任务，可能需要多次触发直到全部完成
  → foundation 阶段完成后自动触发冒烟测试，通过后继续 Phase 2

Agent 07：读取并执行 .claude/agents/07_tester.md
  → 补全测试骨架中的 pending() 占位，添加集成测试，GUT 全量回归直到 0 失败

Agent 08：读取并执行 .claude/agents/08_builder.md
  → 配置 project.godot（增量写入），打包 builds/game.pck

Agent 09：读取并执行 .claude/agents/09_git.md
  → 检测大文件（>1MB 自动启用 Git LFS），初始化本地仓库或生成迭代提交，打语义化 Tag

Agent 10：读取并执行 .claude/agents/10_reporter.md
  → 生成 reports/final_report.md，含各阶段耗时分析

每个 Agent 开始和完成时均会写入 reports/pipeline_log.md，可随时查看进度。
```

---

## 关于分批执行

Agent 02（策划案）和 Agent 06（开发）内置了**上下文保护机制**，单次对话有执行上限：

| Agent | 每批上限 | 原因 |
|-------|---------|------|
| Agent 02 | L 系统 1 个 / M 系统 3 个 / S 系统 4 个 | 九章策划案内容量大 |
| Agent 06 | 5 个任务 | 防止上下文溢出导致质量下降 |

执行完一批后 Claude 会主动停止并提示续跑方式。**直接重新发送对应 Agent 的指令即可**，已完成的内容会被自动跳过。

---

## 中途中断后续跑

| 情况 | 发送指令 |
|------|---------|
| Word 还没转换 | `读取并执行 .claude/agents/00_word_to_md.md` |
| 重新解析 GDD | `读取并执行 .claude/agents/01_bootstrap.md` |
| 策划案未全部完成 | `读取并执行 .claude/agents/02_lead_designer.md`（自动跳过已完成系统） |
| 重新校验策划 | `读取并执行 .claude/agents/03_planner.md` |
| 重新生成架构和任务 | `读取并执行 .claude/agents/04_architect.md` |
| 开发中途中断 | `读取并执行 .claude/agents/06_developer.md`（自动检测并清理中断任务，从下一个 pending 继续） |
| 只跑测试 | `读取并执行 .claude/agents/07_tester.md` |
| 重新构建 | `读取并执行 .claude/agents/08_builder.md` |
| 重新提交版本 | `读取并执行 .claude/agents/09_git.md` |
| 重新生成报告 | `读取并执行 .claude/agents/10_reporter.md` |

> **开发中断说明**：Agent 06 采用三态状态机（`pending → in_progress → done`）。如果上次执行中途崩溃，下次启动时会自动识别 `in_progress` 状态的任务，删除其残留文件后重新实现，无需手动清理。

---

## 质量保障机制

本流水线内置三道质量门，无需手动干预：

| 质量门 | 触发时机 | 通过条件 |
|-------|---------|---------|
| **语法检查** | 每个 `.gd` 文件写入后 | `godot --headless --quit` 无 ERROR |
| **冒烟测试** | Agent 06 完成 foundation 阶段后 | 项目可正常启动，退出码为 0 |
| **GUT 全量回归** | Agent 07 结束前 | 0 个测试失败 |

任意质量门未通过，流水线自动停止并修复，不会继续推进。

---

## 查看执行进度

流水线运行期间可随时查看 `reports/pipeline_log.md`，记录格式如下：

```
[Agent01] 🚀 2024-01-15T14:10:05 | 开始执行
[Agent01] ✅ 2024-01-15T14:11:42 | 输出: project_context.json, schema.json | 摘要: 咖啡馆游戏，8个系统
[Agent02] 🚀 2024-01-15T14:11:43 | 开始执行
[Agent02] ✅ 2024-01-15T14:18:20 | 输出: design/systems/（3个文件）| 摘要: 第1批完成，还有5个系统待输出
```

通过开始/完成时间戳可计算每个阶段的实际耗时，也可判断某个 Agent 是否崩溃（有开始无完成）。

---

## Godot 命令行配置（只需做一次）

1. 将 `Godot_v4.6.2-stable_win64.exe` 重命名为 `godot.exe`
2. 放到 `C:\Godot\`
3. 将 `C:\Godot` 加入系统 PATH 环境变量
4. 重启 VSCode，在终端验证：`godot --version`

---

## 换新游戏

1. 将新的 `.docx` 放入 `design/`（删掉旧的）
2. 删除以下自动生成的内容：

```bash
# 用 Python 删除，跨平台兼容
python -c "
import shutil, os
for p in ['tasks/project_context.json', 'tasks/backlog.json', 'tasks/schema.json',
          'godot_project', 'design/systems', 'reports']:
    if os.path.exists(p):
        shutil.rmtree(p) if os.path.isdir(p) else os.remove(p)
        print(f'已删除: {p}')
"
```

3. 重新发送启动提示词

`.claude/agents/` 中的文件无需改动，对所有游戏类型通用。

---

## 项目结构

```
gamedev-pipeline/
├── .claude/
│   ├── CLAUDE.md                 全局规范 + 流水线配置参数
│   └── agents/
│       ├── 00_word_to_md.md      Word → Markdown
│       ├── 01_bootstrap.md       GDD 解析 + Schema 生成
│       ├── 02_lead_designer.md   主策划（九章系统策划案，分批）
│       ├── 03_planner.md         策划校验（矛盾回写到源文件）
│       ├── 04_architect.md       架构设计（拓扑排序 + 测试骨架）
│       ├── 05_art.md             美术需求 + PlaceholderAssets
│       ├── 06_developer.md       功能开发（分批 + 三态原子任务）
│       ├── 07_tester.md          测试补全 + GUT 全量回归
│       ├── 08_builder.md         构建导出
│       ├── 09_git.md             版本管理（含 Git LFS 检测）
│       └── 10_reporter.md        最终报告（含耗时分析）
├── design/
│   └── 你的策划案.docx            ← 放这里
├── tasks/                        自动生成
│   ├── project_context.json      游戏结构化配置（数据总线）
│   ├── schema.json               JSON Schema 数据契约
│   └── backlog.json              开发任务队列（拓扑排序）
├── reports/                      自动生成
│   ├── pipeline_log.md           执行进度 + 耗时日志
│   ├── pipeline_errors.json      错误持久化记录
│   ├── architecture.md           技术架构文档
│   ├── art_requirements.md       素材清单 + AI 生图提示词
│   ├── test_report.md            测试报告
│   ├── build_report.md           构建报告
│   ├── build_guide.md            Android APK 导出指南
│   └── final_report.md           最终交付报告
├── builds/                       自动生成
│   └── game.pck
└── godot_project/                自动生成
    ├── scripts/
    ├── scenes/
    ├── tests/                    含测试骨架和集成测试
    ├── data/
    └── assets/
```

---

## 常见问题

**Q：GDD 格式有要求吗？**
A：无固定格式。Agent 01 自动识别游戏类型和系统复杂度，Agent 02 会在 `notes.gdd_gaps` 中记录不足之处并主动补全。

**Q：Word 里有表格和图片怎么办？**
A：表格会保留为 Markdown 格式，图片会被忽略。转换后可在 `design/gdd.md` 核查，必要时手动补充图片描述。

**Q：Agent 02 / 06 跑到一半停了，怎么续跑？**
A：这是正常的分批保护机制，不是报错。重新发送对应 Agent 的指令即可，已完成的内容自动跳过。Agent 06 还会自动处理上次中断的任务（清理残留文件后重新实现）。

**Q：流水线某个阶段崩溃了，怎么判断是哪一步？**
A：查看 `reports/pipeline_log.md`，找到有 🚀（开始）但没有 ✅（完成）的 Agent，就是崩溃点。

**Q：测试一直无法 100% 通过怎么办？**
A：Agent 07 会循环修复直到全部通过。如果反复卡在同一个失败，可以单独发送：`读取并执行 .claude/agents/07_tester.md`，并告知具体错误信息让 Claude 重点处理。

**Q：项目有大量美术资源（图片、音频），会影响 Git 吗？**
A：Agent 09 会自动检测 `assets/` 目录下大于 1MB 的文件，如果存在则自动初始化 Git LFS 并追踪 `*.png *.jpg *.ogg *.wav` 等格式。详见 `reports/build_guide.md` 中的 LFS 说明。

**Q：API Error: 400 Invalid user_id？**
A：在系统环境变量中新增 `ANTHROPIC_USER_ID=user001`，重启 VSCode 后重试。

**Q：想调整分批大小或 GUT 版本怎么办？**
A：修改 `.claude/CLAUDE.md` 顶部的"流水线配置参数"表，所有 Agent 均从此表读取，无需逐个修改。
