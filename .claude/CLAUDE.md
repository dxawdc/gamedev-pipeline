# Godot 4 全自动游戏开发流水线 · 主控文档

## 定位

适用于任意 Godot 4 手机端 2D 游戏项目。
**所有游戏专属内容均从 `tasks/project_context.json` 读取，本文件不含任何游戏专属信息。**

---

## Agent 执行顺序

| 编号 | 文件 | 职责 | 输出 |
|-----|------|------|------|
| 00 | 00_word_to_md.md | Word → Markdown | design/gdd.md |
| 01 | 01_bootstrap.md | GDD 解析 | tasks/project_context.json |
| 02 | 02_lead_designer.md | 主策划：九章系统策划案 | design/systems/*.md |
| 03 | 03_planner.md | 跨系统校验 | design/gdd_refined.md |
| 04 | 04_architect.md | 架构设计 + 任务拆解 | reports/architecture.md, tasks/backlog.json |
| 05 | 05_art.md | 美术需求 + 占位素材 | reports/art_requirements.md |
| 06 | 06_developer.md | 功能开发（循环） | godot_project/ |
| 07 | 07_tester.md | 自动测试 | reports/test_report.md |
| 08 | 08_builder.md | 构建导出 | builds/game.pck |
| 09 | 09_git.md | 版本管理：Git 提交 + Tag + 推送 | Git commit, Tag |
| 10 | 10_reporter.md | 最终报告 | reports/final_report.md |

---

## 固定目录结构

```
<项目根>/
├── .claude/
│   ├── CLAUDE.md
│   └── agents/（00~09）
├── design/
│   ├── gdd.md               原始策划案（只读）
│   ├── gdd_refined.md       校验后总设计文档
│   └── systems/             各系统九章策划案
├── tasks/
│   ├── project_context.json 游戏结构化配置（所有 Agent 的数据来源）
│   └── backlog.json         开发任务队列
├── reports/
├── builds/
└── godot_project/
    ├── project.godot
    ├── addons/gut/
    ├── assets/
    ├── data/
    ├── scenes/
    ├── scripts/
    └── tests/
```

---

## GDScript 编码规范

1. 文件命名：`snake_case.gd`
2. 类名：`PascalCase`，每个脚本顶部声明 `class_name`
3. 信号：`snake_case` 动词过去式（`order_served`、`day_ended`）
4. 常量：`UPPER_SNAKE_CASE`
5. 变量：必须有类型注解（`var gold: int = 0`）
6. 私有成员：以 `_` 开头
7. 每个脚本顶部必须有注释说明职责
8. 禁止 `print()` 调试，用 `push_warning()` / `push_error()`
9. 所有 JSON 读写必须做错误处理
10. 信号连接优先用代码，不用编辑器连接

---

## 手机端通用规范

- 基准分辨率：从 `project_context.display` 读取
- 拉伸模式：`canvas_items` + `expand`
- 触控区域：所有可点击元素最小 **88×88px**
- 安全区域：四边各留 **40px**
- 性能：信号驱动 UI 更新，避免每帧 GDScript 循环

---

## Godot Headless 命令

```bash
# 语法检查
godot --headless --path ./godot_project --quit 2>&1

# 资源导入（测试前必须先执行）
godot --headless --path ./godot_project --import --quit

# 运行 GUT 测试
godot --headless -d --display-driver headless --audio-driver Dummy \
  --disable-render-loop --path ./godot_project \
  -s res://addons/gut/gut_cmdln.gd \
  -gdir=res://tests -ginclude_subdirs -glog=2 -gexit

# 导出 PCK
godot --headless --path ./godot_project --export-pack "Android" ../builds/game.pck
```

---

## 强制规则：GDScript 语法检查

**优先级高于一切其他指令，不可跳过。**

每次创建或修改 `.gd` 文件后，必须立即执行语法检查：

```bash
godot --headless --path ./godot_project --quit 2>&1
```

```
写入 .gd 文件
    ↓ 立即执行
    ├─ 有 ERROR / Parse Error → 停止 → 修复 → 再次检查 → 循环
    └─ 无报错 → 打印 ✅ 语法检查通过 → 继续
```

**绝对禁止**在语法检查未通过时标记任务 done 或继续下一任务。

---

## 游戏专属信息读取规则

所有 Agent 需要游戏专属信息时，**必须读取 `tasks/project_context.json`**，不得依赖记忆或假设。

---

## 断点续跑机制

每个 Agent 执行完成后，必须将当前进度写入 `project_context.json` 的 `pipeline_status` 字段。

**标准写入代码（每个 Agent 完成时必须执行）**：

```bash
python -c "
import json, datetime
p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '<当前Agent编号，如02>',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print('[pipeline] status 已更新 → last_completed_agent: <编号>')
"
```

**续跑时**：读取 `pipeline_status.last_completed_agent` 确认从哪个 Agent 继续。
- Agent 02 续跑：检查 `design/systems/` 已有哪些文件，跳过已完成的系统
- Agent 06 续跑：直接读 `backlog.json` 找第一个 `status=pending` 的任务，不重新实现已完成任务

---

## Windows 兼容性说明

所有 Bash 命令在 Windows 的 Claude Code 插件终端中执行时，注意以下差异：

| 操作 | Linux/Mac | Windows（CMD/PowerShell）| 推荐写法（通用）|
|-----|-----------|--------------------------|----------------|
| 删除文件 | `rm file` | `del file` | `python -c "import os; os.remove('file')"` |
| 创建目录 | `mkdir -p a/b` | `mkdir a\b` | `python -c "import os; os.makedirs('a/b', exist_ok=True)"` |
| 列出文件 | `ls dir/` | `dir dir\` | `python -c "import os; print(os.listdir('dir'))"` |
| 文件存在检查 | `test -f file` | `if exist file` | `python -c "import os; print(os.path.exists('file'))"` |

**原则**：涉及文件系统操作时，优先使用 Python 单行命令确保跨平台兼容。
Git 命令和 Godot headless 命令在 Windows 上直接使用，无需调整。

## 语言设定

你必须在所有对话中使用**简体中文**回复，包括解释、代码注释、任务报告等。