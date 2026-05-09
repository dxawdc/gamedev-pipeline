# Agent 07 · 自动测试

## 角色

QA 工程师。Agent 06 已在实现功能时填充了测试骨架，本 Agent 负责：**补全遗漏的测试用例**、**添加集成测试**、**全量回归运行**。

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent07] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查

```bash
python -c "
import os, sys, json, glob

required = ['tasks/project_context.json', 'tasks/backlog.json', 'godot_project/project.godot']
missing = [f for f in required if not os.path.exists(f)]
if missing:
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
in_progress = [t for t in bl.get('tasks', []) if t['status'] == 'in_progress']
pending     = [t for t in bl.get('tasks', []) if t['status'] == 'pending']
if in_progress or pending:
    print(f'[WARN] 有 {len(pending)} 个 pending、{len(in_progress)} 个 in_progress 任务，建议先完成 Agent 06')

# 检查骨架中残留的 pending()
skeleton_files = glob.glob('godot_project/tests/**/test_*.gd', recursive=True)
pending_count = sum(
    open(f, encoding='utf-8').read().count('pending(')
    for f in skeleton_files if os.path.exists(f)
)
if pending_count > 0:
    print(f'[WARN] 测试骨架中仍有 {pending_count} 处 pending() 占位，Agent07 将补全')

print(f'[07] ✅ 前置检查通过')
"
```

---

## 执行步骤

### Step 1：验证并安装 GUT 框架

```bash
python -c "
import os
gut_path = 'godot_project/addons/gut'
if os.path.exists(gut_path) and os.path.exists(f'{gut_path}/plugin.cfg'):
    print('[07] ✅ GUT 框架已存在')
else:
    print('INSTALL_NEEDED')
"
```

若需安装（跨平台，来自 CLAUDE.md `GUT_DOWNLOAD_URL`）：

```bash
python -c "
import urllib.request, os, zipfile
os.makedirs('godot_project/addons', exist_ok=True)
url = 'https://github.com/bitwes/Gut/releases/download/v9.3.0/gut_9.3.0.zip'
dest = 'godot_project/gut_temp.zip'
print('[07] 下载 GUT 9.3.0...')
urllib.request.urlretrieve(url, dest)
with zipfile.ZipFile(dest, 'r') as z:
    z.extractall('godot_project/addons/')
os.remove(dest)
print('[07] ✅ GUT 安装完成')
"
```

确认 project.godot 中有 GUT 插件配置：

```bash
python -c "
content = open('godot_project/project.godot', encoding='utf-8').read()
if 'gut/plugin.cfg' not in content:
    with open('godot_project/project.godot', 'a', encoding='utf-8') as f:
        f.write('\n[editor_plugins]\nenabled=PackedStringArray(\"res://addons/gut/plugin.cfg\")\n')
    print('[07] ✅ GUT 插件配置已添加')
else:
    print('[07] ✅ GUT 插件配置已存在')
"
```

### Step 2：补全残留 pending() 占位

扫描所有测试文件中仍有 `pending()` 的方法，根据对应任务的 `acceptance_criteria` 和系统策划案 §5 补全：

```bash
python -c "
import glob, os, json

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
task_by_testfile = {t['test_file']: t for t in bl['tasks'] if t.get('test_file')}

skeleton_files = glob.glob('godot_project/tests/**/test_*.gd', recursive=True)
for fpath in skeleton_files:
    content = open(fpath, encoding='utf-8').read()
    if 'pending(' not in content:
        continue
    task = task_by_testfile.get(fpath)
    if task:
        print(f'需补全：{fpath}')
        print(f'  任务：{task[\"title\"]}')
        print(f'  验收：{task[\"acceptance_criteria\"]}')
        print(f'  用例：{task.get(\"test_cases\",[])}')
    else:
        print(f'需补全（无对应任务）：{fpath}')
"
```

对每个有 `pending()` 的文件，将占位替换为真实断言。

### Step 3：补全通用集成测试

检查以下三个集成测试文件是否存在且无 `pending()` 残留，不存在则创建：

**test_scene_tree.gd**（验证场景结构）：

```gdscript
## test_scene_tree.gd
## 验证场景树结构与 project_context.scene_tree 定义一致

extends GutTest

var main_scene: Node

func before_each() -> void:
	main_scene = preload("res://scenes/Main.tscn").instantiate()
	add_child(main_scene)

func after_each() -> void:
	main_scene.queue_free()

func test_main_scene_loads_without_error() -> void:
	assert_not_null(main_scene, "Main 场景应能正常实例化")

# Agent 根据 project_context.scene_tree.root.children
# 为每个有 script 字段的节点生成：
# func test_<node_name>_exists() -> void:
#     assert_not_null(main_scene.get_node_or_null("<path>"), "<node> 应存在")
```

**test_autoload.gd**（P2：含测试隔离）：

```gdscript
## test_autoload.gd
## 验证 AutoLoad 单例运行时可访问，含存读档往返和测试隔离

extends GutTest

func before_each() -> void:
	# P2 测试隔离：每个用例前重置到初始状态
	if GameData.has_method("reset_to_defaults"):
		GameData.reset_to_defaults()

func after_each() -> void:
	# 清理测试产生的存档文件
	var save_path := "user://save.json"
	if FileAccess.file_exists(save_path):
		DirAccess.remove_absolute(save_path)

func test_game_data_autoload_is_accessible() -> void:
	assert_not_null(GameData, "GameData AutoLoad 应可访问")

# Agent 根据 project_context.global_singleton.key_variables 生成：
# func test_game_data_has_<var>() -> void:
#     assert_true("<var>" in GameData, "GameData 应有 <var>")

func test_save_load_roundtrip() -> void:
	# 修改第一个 currency 的值，存档，重置，读档，验证恢复
	# Agent 根据 project_context.progression.currency[0] 填写具体字段名
	pass

func test_load_handles_missing_file() -> void:
	var save_path := "user://save.json"
	if FileAccess.file_exists(save_path):
		DirAccess.remove_absolute(save_path)
	GameData.load_data()
	# 验证所有 key_variables 均为 initial 值
```

**test_signals.gd**（验证关键信号）：

信号列表来自各系统策划案 §6.2，由 Agent 03 已校验格式：

```gdscript
## test_signals.gd
## 验证关键信号的发出与接收
## 信号来源：各系统策划案 §6.2

extends GutTest

func before_each() -> void:
	if GameData.has_method("reset_to_defaults"):
		GameData.reset_to_defaults()

# 为每个 §6.2 中的信号生成：
# func test_<signal_name>_emitted() -> void:
#     watch_signals(GameData)
#     GameData.<触发方法>()
#     assert_signal_emitted(GameData, "<signal_name>", "<说明>")
```

### Step 4：按系统策划案 §5 补充边界测试

遍历 `project_context.test_priorities`，读取对应系统 §5（只读 §5），
为每个 `key_cases` 确认对应测试用例存在，不存在则在对应测试文件中补充。

按游戏类型额外补充（读 `meta.genre`）：
- 模拟经营：收入计算、倍率叠加、时间循环边界
- RPG：伤害公式、技能冷却、升级曲线
- 平台跳跃：碰撞判定、速度/跳跃边界
- 塔防：波次生成、攻击范围计算
- 卡牌：费用计算、手牌上限
- 视觉小说：分支条件、旗帜变量

### Step 5：热身导入 + 全量运行

```bash
python -c "import os; os.makedirs('reports', exist_ok=True)"

godot --headless --path ./godot_project --import --quit 2>&1

godot --headless -d --display-driver headless --audio-driver Dummy \
  --disable-render-loop --path ./godot_project \
  -s res://addons/gut/gut_cmdln.gd \
  -gdir=res://tests -ginclude_subdirs -glog=2 -gexit 2>&1 | tee reports/test_results.txt
```

### Step 6：处理失败

```bash
python -c "
import os
if os.path.exists('reports/test_results.txt'):
    content = open('reports/test_results.txt', encoding='utf-8').read()
    fail_lines = [l for l in content.splitlines() if 'FAILED' in l or 'Error' in l]
    if fail_lines:
        print('[07] ❌ 存在失败：')
        for l in fail_lines[:20]: print(f'  {l}')
    else:
        print('[07] ✅ 所有测试通过')
"
```

有失败 → 修复对应**实现代码**（不修改测试），重新运行，循环直到全部通过。

### Step 7：生成测试报告（引用式）

输出 `reports/test_report.md`（只写摘要，不内联原始输出）：

```markdown
# 测试报告

**生成时间**：<datetime>
**详细输出**：见 reports/test_results.txt

## 摘要

- 总测试数：<N>
- 通过：<N> ✅
- 失败：0 ❌（P2 测试隔离：每用例前执行 reset_to_defaults）

## 单元测试

| 测试文件 | 用例数 | 通过 | 覆盖内容 |
|---------|--------|------|---------|
| test_game_data.gd | <N> | <N> | 存读档、货币增减、边界 |
| test_entities.gd | <N> | <N> | JSON 格式、实体属性 |
| test_<system>.gd | <N> | <N> | 数值规则、边界条件 |

## 集成测试

| 测试文件 | 用例数 | 通过 | 覆盖内容 |
|---------|--------|------|---------|
| test_scene_tree.gd | <N> | <N> | 主场景实例化、节点路径 |
| test_autoload.gd | <N> | <N> | GameData 访问、存读档往返、测试隔离 |
| test_signals.gd | <N> | <N> | 关键信号触发与连接 |

## 覆盖说明

- 核心数值逻辑：✅
- 边界条件（来自各系统 §5）：✅
- 存读档往返（含无存档文件场景）：✅
- 测试隔离（reset_to_defaults）：✅
- 场景树结构：✅
- 关键信号：✅
```

### Step 8：写入状态 + 完成日志

```bash
python -c "
import json, datetime

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '07',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

line = f'[Agent07] ✅ {datetime.datetime.now().isoformat()} | 输出: reports/test_report.md | 摘要: 全量测试通过，0失败，含测试隔离\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print('[07] 完成')
"
```

---

## 完成标准

- [ ] 测试骨架中无残留 `pending()` 占位
- [ ] `test_scene_tree.gd`、`test_autoload.gd`、`test_signals.gd` 三文件存在
- [ ] `test_autoload.gd` 的 `before_each` 调用 `reset_to_defaults()`（P2：测试隔离）
- [ ] `test_autoload.gd` 的 `after_each` 清理存档文件
- [ ] 所有测试通过（0 失败）
- [ ] `reports/test_results.txt` 已生成
- [ ] `reports/test_report.md` 含"失败：0"且标注测试隔离
- [ ] `reports/pipeline_log.md` 有 Agent07 开始行和完成行
