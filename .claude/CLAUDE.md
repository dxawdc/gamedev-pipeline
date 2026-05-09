# Godot 4 全自动游戏开发流水线 · 主控文档

## 定位

适用于任意 Godot 4 手机端 2D 游戏项目。
**所有游戏专属内容均从 `tasks/project_context.json` 读取，本文件不含任何游戏专属信息。**

---

## 流水线配置参数（P2：集中配置，所有 Agent 读此表）

| 参数 | 值 | 说明 |
|-----|---|------|
| `BATCH_SYSTEMS` | 动态（S:4 / M:3 / L:1） | Agent 02 每批系统数，按 complexity 字段决定 |
| `BATCH_TASKS` | 5 | Agent 06 每批任务数上限 |
| `REFRESH_EVERY` | 5 | Agent 06 每完成 N 个任务触发规范刷新 |
| `MIN_SPEC_SIZE` | 2000 | 系统策划案最小字节数，低于此值视为不完整 |
| `GUT_VERSION` | 9.3.0 | GUT 测试框架版本 |
| `GUT_DOWNLOAD_URL` | `https://github.com/bitwes/Gut/releases/download/v9.3.0/gut_9.3.0.zip` | GUT 下载地址 |
| `SCHEMA_PATH` | `tasks/schema.json` | project_context.json 的 JSON Schema 路径 |
| `SMOKE_TEST_PHASE` | foundation | 完成此 phase 后执行冒烟测试 |

> **修改参数只需改此表，不需要逐个修改 Agent 文件。**

---

## Agent 执行顺序

| 编号 | 文件 | 职责 | 输出 |
|-----|------|------|------|
| 00 | 00_word_to_md.md | Word → Markdown | design/gdd.md |
| 01 | 01_bootstrap.md | GDD 解析 + Schema 生成 | tasks/project_context.json, tasks/schema.json |
| 02 | 02_lead_designer.md | 主策划：九章系统策划案（按复杂度分批） | design/systems/*.md |
| 03 | 03_planner.md | 跨系统校验 | design/gdd_refined.md |
| 04 | 04_architect.md | 架构设计 + 拓扑排序任务拆解 + 测试骨架 | reports/architecture.md, tasks/backlog.json |
| 05 | 05_art.md | 美术需求 + 占位素材 | reports/art_requirements.md |
| 06 | 06_developer.md | 功能开发（分批、原子任务、冒烟检查点） | godot_project/ |
| 07 | 07_tester.md | 测试补全 + 全量回归（含测试隔离） | reports/test_report.md |
| 08 | 08_builder.md | 构建导出 | builds/game.pck |
| 09 | 09_git.md | 版本管理（含 Git LFS 检测） | Git commit, Tag |
| 10 | 10_reporter.md | 最终报告 | reports/final_report.md |

---

## 固定目录结构

```
<项目根>/
├── .claude/
│   ├── CLAUDE.md
│   └── agents/（00~10）
├── design/
│   ├── gdd.md
│   ├── gdd_refined.md
│   └── systems/
├── tasks/
│   ├── project_context.json   所有 Agent 的数据来源
│   ├── schema.json            ← P1新增：JSON Schema 契约
│   └── backlog.json
├── reports/
│   ├── pipeline_log.md        ← 开始/完成摘要日志
│   ├── pipeline_errors.json   ← 错误持久化
│   └── ...
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
11. **`GameData.gd` 必须包含 `reset_to_defaults()` 方法**（P2：测试隔离，仅测试环境调用）

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
# 语法检查 / 冒烟测试（退出码 0 且无 ERROR = 通过）
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

```
写入 .gd 文件
    ↓ 立即执行 godot --headless --path ./godot_project --quit 2>&1
    ├─ 有 ERROR / Parse Error → 写入 pipeline_errors.json → 修复 → 再次检查 → 循环
    └─ 无报错 → 打印 ✅ 语法检查通过 → 继续
```

**绝对禁止**在语法检查未通过时标记任务 done 或继续下一任务。

---

## P0：任务原子性（三态状态机）

backlog 任务状态：`pending` → `in_progress` → `done`

- Agent 06 **开始**一个任务时，立即将其状态写为 `in_progress`
- 续跑时发现 `in_progress` = 上次中断，必须先删除其 `files_to_create` 再重新实现
- 只有语法检查通过后才能写为 `done`

**中断恢复代码**（Agent 06 前置检查中执行）：

```bash
python -c "
import json, os

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
interrupted = [t for t in bl['tasks'] if t['status'] == 'in_progress']
for t in interrupted:
    print(f'[06] ⚠️  发现中断任务：{t[\"id\"]}，清理残留文件并重置...')
    for f in t.get('files_to_create', []):
        if os.path.exists(f):
            os.remove(f)
            print(f'  🗑️  删除: {f}')
    t['status'] = 'pending'
if interrupted:
    json.dump(bl, open('tasks/backlog.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
    print(f'[06] {len(interrupted)} 个中断任务已重置，将重新实现')
else:
    print('[06] 无中断任务')
"
```

---

## P0：冒烟测试检查点

Agent 06 完成 `foundation` phase 所有任务后执行：

```bash
godot --headless --path ./godot_project --quit 2>&1
```

```
退出码 0 且无 ERROR → ✅ 冒烟通过，继续 Phase 2
退出码非 0 或含 ERROR → ❌ 停止，修复架构问题后再继续
```

这是流水线最关键的质量门，架构问题在这里最早暴露。

---

## P0：backlog 拓扑排序规则

Agent 04 生成 backlog 时，必须用 Kahn 算法对 `tasks` 数组做拓扑排序。
排序后 Agent 06 顺序执行即可，无需每次扫描依赖状态。

---

## P1：Agent 日志规范（pipeline_log.md）

每个 Agent **开始时**和**完成时**各追加一行，格式固定：

```
[AgentXX] 🚀 <ISO时间戳> | 开始执行
[AgentXX] ✅ <ISO时间戳> | 输出: <关键文件> | 摘要: <一句话>
```

开始日志让"未启动"和"启动后崩溃"可区分；耗时 = 完成时间 - 开始时间。

---

## P1：全局错误日志（pipeline_errors.json）

语法检查失败、测试失败时写入，修复后标记 `resolved: true`。

---

## P1：Schema 契约验证

`tasks/schema.json` 由 Agent 01 生成，定义 `project_context.json` 的结构契约。
每个需要读取 `project_context.json` 的 Agent，读取前执行验证：

```bash
python -c "
import json, sys, os
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
schema_path = 'tasks/schema.json'
if os.path.exists(schema_path):
    try:
        import jsonschema
        jsonschema.validate(ctx, json.load(open(schema_path, encoding='utf-8')))
        print('[OK] Schema 验证通过')
    except ImportError:
        print('[WARN] jsonschema 未安装（pip install jsonschema），跳过验证')
    except jsonschema.ValidationError as e:
        print(f'[ERROR] Schema 验证失败：{e.message}（路径：{list(e.absolute_path)}）')
        sys.exit(1)
"
```

---

## P2：前置文件检查标准模板

```bash
python -c "
import os, sys
required = ['tasks/project_context.json']  # 各 Agent 按需调整
missing = [f for f in required if not os.path.exists(f)]
if missing:
    print('[ERROR] 缺少前置文件，请先运行上一个 Agent：')
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)
print('[OK] 前置文件检查通过')
"
```

---

## Windows 兼容性说明

| 操作 | 推荐写法（通用）|
|-----|----------------|
| 删除文件 | `python -c "import os; os.remove('file')"` |
| 创建目录 | `python -c "import os; os.makedirs('a/b', exist_ok=True)"` |
| 列出文件 | `python -c "import os; print(os.listdir('dir'))"` |
| 文件存在检查 | `python -c "import os; print(os.path.exists('file'))"` |
| 下载文件 | `python -c "import urllib.request; urllib.request.urlretrieve('url', 'dest')"` |

---

## 语言设定

你必须在所有对话中使用**简体中文**回复，包括解释、代码注释、任务报告等。
