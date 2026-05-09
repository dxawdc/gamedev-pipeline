# Agent 04 · 架构设计

## 角色

Godot 4 技术架构师。设计技术架构，用 Kahn 算法生成拓扑排序的任务队列，并为每个任务预生成测试文件骨架（P1：TDD 前置）。

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent04] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查 + Schema 验证

```bash
python -c "
import os, sys, json, glob

required = ['tasks/project_context.json', 'design/gdd_refined.md']
missing = [f for f in required if not os.path.exists(f)]
if missing:
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)

specs = [f for f in glob.glob('design/systems/*.md') if 'README' not in f]
if not specs:
    print('[ERROR] design/systems/ 下没有系统策划案'); sys.exit(1)

# Schema 验证
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
if os.path.exists('tasks/schema.json'):
    try:
        import jsonschema
        jsonschema.validate(ctx, json.load(open('tasks/schema.json', encoding='utf-8')))
        print('[04] ✅ Schema 验证通过')
    except ImportError: pass
    except jsonschema.ValidationError as e:
        print(f'[ERROR] {e.message}'); sys.exit(1)

print(f'[04] ✅ 前置检查通过，{len(specs)} 个系统策划案就绪')
"
```

---

## 执行步骤

### Step 1：生成技术架构文档

```bash
python -c "import os; os.makedirs('reports', exist_ok=True)"
```

输出 `reports/architecture.md`，包含：

**§1.1 脚本职责表**：遍历 `project_context.systems` 和 `ui_screens` 生成表格，含脚本路径、节点路径、职责、关键方法签名。

**§1.2 场景树**：使用 `project_context.scene_tree` 定义的结构，用文本树形图表示。

**§1.3 GlobalSingleton 规范**：基于 `project_context.global_singleton` 生成完整接口规范，包含所有 `key_variables` 类型注解、方法签名，以及 `reset_to_defaults()` 的接口定义（P2：测试隔离用）。

**§1.4 数据文件 Schema**：基于 `project_context.data_files` 为每个 JSON 文件定义结构。

**§1.5 手机端性能规范**：CompressedTexture2D、信号驱动、懒加载策略。

### Step 2：生成开发任务队列（含拓扑排序）

输出 `tasks/backlog.json`。

**固定 Phase 1**：

```json
[
  {
    "id": "task_001", "phase": "foundation",
    "title": "创建 Godot 项目结构",
    "description": "创建 project.godot、标准目录（scripts/scenes/data/assets/tests）",
    "files_to_create": ["godot_project/project.godot"],
    "depends_on": [],
    "test_file": "godot_project/tests/test_project_structure.gd",
    "acceptance_criteria": ["project.godot 存在且可被 godot --headless 解析", "所有目录已创建"],
    "status": "pending", "priority": 1
  },
  {
    "id": "task_002", "phase": "foundation",
    "title": "实现 GameData.gd 全局单例",
    "description": "按 project_context.global_singleton 规范实现，含 reset_to_defaults() 方法",
    "files_to_create": ["godot_project/scripts/GameData.gd"],
    "depends_on": ["task_001"],
    "test_file": "godot_project/tests/test_game_data.gd",
    "acceptance_criteria": ["GameData 可作为 AutoLoad 加载", "save/load_data/reset_to_defaults 方法存在"],
    "status": "pending", "priority": 1
  },
  {
    "id": "task_003", "phase": "foundation",
    "title": "创建所有数据 JSON 文件",
    "description": "按 project_context.data_files，每个文件含初始数据结构",
    "files_to_create": [],
    "depends_on": ["task_001"],
    "test_file": "godot_project/tests/test_entities.gd",
    "acceptance_criteria": ["所有 data_files 中的文件存在", "每个文件是合法 JSON"],
    "status": "pending", "priority": 1
  },
  {
    "id": "task_004", "phase": "foundation",
    "title": "安装并验证 GUT 测试框架",
    "description": "安装 GUT，在 project.godot 启用插件，运行空测试确认框架可用",
    "files_to_create": [],
    "depends_on": ["task_001"],
    "test_file": "godot_project/tests/test_gut_smoke.gd",
    "acceptance_criteria": ["addons/gut/plugin.cfg 存在", "GUT 命令行可正常退出"],
    "status": "pending", "priority": 1
  }
]
```

**Phase 2-N（从 systems 动态生成）**：
- P0 → phase `core_systems`，P1 → `gameplay`，P2 → `progression`
- 每个系统参考对应 `design/systems/<id>.md` **第二章模块拆解**：有 N 个模块 → 至少 N 个任务
- 每个任务 `files_to_create` ≤ 3 个，单任务粒度约 30 分钟

**P1：测试骨架字段**：每个任务除了 `test_file` 路径外，新增 `test_cases` 字段：

```json
{
  "test_file": "godot_project/tests/test_xxx.gd",
  "test_cases": [
    "test_<功能>_正常流程",
    "test_<功能>_边界值",
    "test_<功能>_非法输入"
  ]
}
```

`test_cases` 来源：对应系统策划案第五章的边界条件列表，每个条目转换为一个测试方法名。

### Step 3：P0 拓扑排序（Kahn 算法）

生成任务列表后，必须用拓扑排序重新排列，保证 Agent 06 顺序执行无需扫描依赖：

```bash
python -c "
import json, sys
from collections import deque

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
tasks = bl['tasks']
task_map = {t['id']: t for t in tasks}
task_ids = list(task_map.keys())

# 构建入度图
in_degree = {tid: 0 for tid in task_ids}
adj = {tid: [] for tid in task_ids}
errors = []

for t in tasks:
    for dep in t.get('depends_on', []):
        if dep not in task_map:
            errors.append(f'{t[\"id\"]} 依赖 {dep}，但该任务不存在')
        else:
            adj[dep].append(t['id'])
            in_degree[t['id']] += 1

if errors:
    for e in errors: print(f'[ERROR] {e}')
    sys.exit(1)

# Kahn 算法
queue = deque([tid for tid in task_ids if in_degree[tid] == 0])
sorted_ids = []
while queue:
    tid = queue.popleft()
    sorted_ids.append(tid)
    for neighbor in adj[tid]:
        in_degree[neighbor] -= 1
        if in_degree[neighbor] == 0:
            queue.append(neighbor)

if len(sorted_ids) != len(task_ids):
    print('[ERROR] 检测到循环依赖，拓扑排序失败')
    remaining = set(task_ids) - set(sorted_ids)
    print(f'  涉及任务：{remaining}')
    sys.exit(1)

# 按拓扑顺序重排 tasks 数组
sorted_tasks = [task_map[tid] for tid in sorted_ids]
bl['tasks'] = sorted_tasks
bl['total'] = len(sorted_tasks)
bl['completed'] = 0

# 字段完整性检查
field_errors = []
for t in sorted_tasks:
    if not t.get('acceptance_criteria') or len(t['acceptance_criteria']) < 2:
        field_errors.append(f'{t[\"id\"]} acceptance_criteria 少于2条')
    if not t.get('test_file'):
        field_errors.append(f'{t[\"id\"]} 缺少 test_file')
    if len(t.get('files_to_create', [])) > 3:
        field_errors.append(f'{t[\"id\"]} files_to_create 超过3个，请拆分')
    if not t.get('test_cases'):
        field_errors.append(f'{t[\"id\"]} 缺少 test_cases（测试骨架）')

if field_errors:
    print('[04] ❌ 字段验证失败：')
    for e in field_errors: print(f'  - {e}')
    sys.exit(1)

json.dump(bl, open('tasks/backlog.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print(f'[04] ✅ 拓扑排序完成，{len(sorted_tasks)} 个任务，无循环依赖')
print(f'  执行顺序前5：{sorted_ids[:5]}')
"
```

### Step 4：P1 生成测试骨架文件

根据 backlog 中每个任务的 `test_file` 和 `test_cases`，预生成骨架文件：

```bash
python -c "
import json, os

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
os.makedirs('godot_project/tests', exist_ok=True)
created = 0

for t in bl['tasks']:
    test_path = t.get('test_file', '')
    test_cases = t.get('test_cases', [])
    if not test_path or not test_cases:
        continue
    if os.path.exists(test_path):
        continue  # 不覆盖已有测试

    # 生成骨架
    lines = [
        f'## {os.path.basename(test_path)}',
        f'## 测试任务：{t[\"title\"]}',
        f'## 验收条件：{t[\"acceptance_criteria\"]}',
        '',
        'extends GutTest',
        '',
        'func before_each() -> void:',
        '\tif GameData.has_method(\"reset_to_defaults\"):',
        '\t\tGameData.reset_to_defaults()  # P2: 测试隔离',
        '',
        'func after_each() -> void:',
        '\tpass',
        '',
    ]
    for case in test_cases:
        lines += [
            f'func {case}() -> void:',
            '\tpending(\"待 Agent 06 实现后填充\")',
            '',
        ]

    os.makedirs(os.path.dirname(test_path), exist_ok=True)
    with open(test_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(lines))
    created += 1

print(f'[04] ✅ 测试骨架已生成：{created} 个文件')
"
```

### Step 5：更新状态 + 完成日志

```bash
python -c "
import json, datetime

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '04',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
total = bl.get('total', 0)
line = f'[Agent04] ✅ {datetime.datetime.now().isoformat()} | 输出: reports/architecture.md, tasks/backlog.json | 摘要: {total}个任务已拓扑排序，测试骨架已生成\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print(f'[04] 完成，任务总数：{total}')
"
```

---

## 完成标准

- [ ] `reports/architecture.md` 含五章，§1.3 包含 `reset_to_defaults()` 接口
- [ ] `tasks/backlog.json` 中 `tasks` 数组已拓扑排序（可直接顺序执行）
- [ ] 每个任务有 `test_file`、`acceptance_criteria`（≥2条）、`test_cases`（≥2条）
- [ ] 每个任务 `files_to_create` ≤ 3 个
- [ ] 测试骨架文件已生成（`pending()` 占位）
- [ ] `reports/pipeline_log.md` 有 Agent04 开始行和完成行
