# Agent 07 · 自动测试

## 角色

QA 工程师。基于 `project_context.json` 的系统列表和各系统策划案第五章的边界条件，为每个核心系统编写 GUT 测试并自动执行。

---

## 输入（必须先读取）

1. `tasks/project_context.json` → `systems`、`test_priorities`、`progression`、`entities`
2. `design/systems/` 下所有 `*.md` 的第五章（边界条件与异常处理）
3. `tasks/backlog.json` → 每个任务的 `test_file` 路径

---

## 执行步骤

### Step 1：验证 GUT 框架

```bash
python -c "import os; print('GUT 存在' if os.path.exists('godot_project/addons/gut') else 'GUT 缺失')"
```

缺失则安装：

```bash
cd godot_project
curl -L https://github.com/bitwes/Gut/releases/download/v9.3.0/gut_9.3.0.zip -o gut.zip
python -m zipfile -e gut.zip addons/
```

删除压缩包（Windows/Mac/Linux 通用）：
```bash
python -c "import os; os.remove('gut.zip')"
```

并在 `project.godot` 的 `[editor_plugins]` 中添加：
```ini
enabled=PackedStringArray("res://addons/gut/plugin.cfg")
```

### Step 2：生成测试文件

**测试文件标准格式**：

```gdscript
## test_<system_id>.gd
## 测试 <系统名称> 的核心功能
## 测试依据：design/systems/<system_id>.md 第三章、第五章

extends GutTest

func before_each() -> void:
    pass

func after_each() -> void:
    pass

# 测试用例命名规范：test_<被测功能>_<场景描述>
```

**必须生成的通用测试**（所有游戏）：

`test_game_data.gd`：
- 遍历 `project_context.global_singleton.key_variables`，为每个变量生成初始值断言
- 遍历 `project_context.progression.currency`，生成增减和边界测试
- 测试 `save()` / `load()` 正确性（包括文件不存在时的默认值）

`test_progression.gd`：
- 遍历 `project_context.progression.unlock_conditions`，为每个解锁条件生成正/负向测试

`test_entities.gd`：
- 验证 `project_context.data_files` 中的 JSON 文件格式正确
- 验证关键实体属性存在且类型正确

**按系统策划案第五章生成专项测试**：

遍历 `project_context.test_priorities`，为每个系统的 `key_cases` 生成对应测试用例。重点覆盖各系统第五章的：
- 数值边界（最小值、最大值、超出上限/下限）
- 非法操作拦截（余额不足、重复提交）
- 幂等设计

**按游戏类型额外生成**：

读取 `project_context.meta.genre`：
- 模拟经营：收入计算、倍率叠加、时间循环
- RPG：伤害公式、技能效果、升级曲线
- 平台跳跃：碰撞判定、速度/跳跃边界
- 塔防：波次生成、攻击范围
- 卡牌：费用计算、手牌上限
- 视觉小说：分支条件、旗帜变量

**必须生成的集成测试**（GUT 在引擎内运行，完全支持以下场景）：

---

`test_scene_tree.gd` — 场景树结构验证

验证主场景能正确实例化，关键节点路径存在，脚本已挂载：

```gdscript
## test_scene_tree.gd
## 验证场景树结构与 project_context.scene_tree 定义一致
## GUT 在引擎内运行，可直接实例化场景并检查节点

extends GutTest

var main_scene: Node

func before_each() -> void:
    # 实例化主场景（不启动游戏循环，仅验证结构）
    main_scene = preload("res://scenes/Main.tscn").instantiate()
    add_child(main_scene)

func after_each() -> void:
    main_scene.queue_free()

func test_main_scene_loads_without_error() -> void:
    assert_not_null(main_scene, "Main 场景应能正常实例化")

func test_game_world_node_exists() -> void:
    assert_not_null(
        main_scene.get_node_or_null("GameWorld"),
        "GameWorld 节点应存在"
    )

func test_ui_canvas_layer_exists() -> void:
    assert_not_null(
        main_scene.get_node_or_null("UI"),
        "UI CanvasLayer 应存在"
    )

func test_hud_node_exists() -> void:
    assert_not_null(
        main_scene.get_node_or_null("UI/HUD"),
        "HUD 节点应存在"
    )

# 根据 project_context.scene_tree 的 children 列表，
# 为每个关键节点路径生成对应的 test_<节点名>_exists 用例
```

**生成规则**：读取 `project_context.scene_tree`，遍历所有 children，
为每个有 `script` 字段的节点生成一个节点存在性断言 + 一个脚本类型断言。

---

`test_autoload.gd` — AutoLoad 单例集成验证

验证 GameData 在场景中真实可访问，而不只是脚本文件存在：

```gdscript
## test_autoload.gd
## 验证 AutoLoad 单例在运行时可正确访问
## GameData 由 project.godot 注册，GUT 运行时自动加载

extends GutTest

func test_game_data_autoload_is_accessible() -> void:
    # AutoLoad 在 GUT 运行时已自动挂载，可直接通过名称访问
    assert_not_null(GameData, "GameData AutoLoad 应可访问")

func test_game_data_has_required_variables() -> void:
    # 根据 project_context.global_singleton.key_variables 逐一验证
    # 示例（Agent 根据实际变量列表生成）：
    assert_true("gold" in GameData, "GameData 应有 gold 变量")
    assert_true("day" in GameData, "GameData 应有 day 变量")

func test_game_data_save_load_roundtrip() -> void:
    # 在真实 AutoLoad 实例上执行存读档往返测试
    var original_gold = GameData.gold
    GameData.gold = 9999
    GameData.save()
    GameData.gold = 0
    GameData.load_data()
    assert_eq(GameData.gold, 9999, "读档后 gold 应恢复为存档值")
    # 清理：还原测试数据
    GameData.gold = original_gold
    GameData.save()

func test_game_data_load_handles_missing_file() -> void:
    # 删除存档文件，验证 load 不崩溃且返回默认值
    var save_path = "user://save.json"
    if FileAccess.file_exists(save_path):
        DirAccess.remove_absolute(save_path)
    GameData.load_data()
    assert_eq(GameData.gold, 0, "无存档时 gold 应为初始值 0")
```

**生成规则**：读取 `project_context.global_singleton.key_variables`，
为每个变量生成 `assert_true("<var>" in GameData, ...)` 断言，
以及存读档往返测试（存入非默认值 → 读取 → 验证一致）。

---

`test_signals.gd` — 信号连接与触发验证

验证关键系统间的信号能正确发出和接收，而不只是方法存在：

```gdscript
## test_signals.gd
## 验证关键信号的发出与接收
## GUT 提供 watch_signals() 和 assert_signal_emitted() 原生支持

extends GutTest

var _receiver_called := false

func before_each() -> void:
    _receiver_called = false

# ——— 示例：验证 GameData 的关键信号 ———

func test_gold_changed_signal_emitted_on_add() -> void:
    watch_signals(GameData)
    var before = GameData.gold
    GameData.add_gold(100)
    assert_signal_emitted(GameData, "gold_changed",
        "add_gold() 应触发 gold_changed 信号")
    GameData.gold = before  # 还原

func test_day_ended_signal_emitted() -> void:
    watch_signals(GameData)
    GameData.end_day()
    assert_signal_emitted(GameData, "day_ended",
        "end_day() 应触发 day_ended 信号")

# ——— 示例：验证场景内信号连接 ———

func test_serve_button_connected_to_handler() -> void:
    var scene = preload("res://scenes/Main.tscn").instantiate()
    add_child(scene)
    var btn = scene.get_node_or_null("UI/HUD/ServeButton")
    assert_not_null(btn, "ServeButton 应存在")
    # 检查按钮的 pressed 信号已有连接
    assert_true(
        btn.pressed.get_connections().size() > 0,
        "ServeButton.pressed 应已连接到处理函数"
    )
    scene.queue_free()
```

**生成规则**：读取 `project_context.systems` 中每个系统的信号列表
（来自各系统策划案第六章定义的关键事件），为每个信号生成：
- `watch_signals()` + 触发动作 + `assert_signal_emitted()` 三段式用例
- 信号未连接时的负向测试（验证错误处理是否正确）

---

### Step 3：热身导入（测试前必须执行）

```bash
godot --headless --path ./godot_project --import --quit
```

### Step 4：运行测试

```bash
godot --headless -d --display-driver headless --audio-driver Dummy --disable-render-loop --path ./godot_project -s res://addons/gut/gut_cmdln.gd -gdir=res://tests -ginclude_subdirs -glog=2 -gexit 2>&1
```

将输出保存到 `reports/test_results.txt`。

### Step 5：处理失败

有测试失败时：
1. 打印失败详情（期望值、实际值、报错行）
2. 修复对应**实现代码**（不修改测试）
3. 重新运行，循环直到全部通过

### Step 6：生成测试报告

输出 `reports/test_report.md`：

```markdown
# 测试报告

## 摘要
- 总测试数：<N>
- 通过：<N> ✅
- 失败：0 ❌
- 通过率：100%

## 各层测试结果

### 单元测试（纯逻辑）

| 测试文件 | 用例数 | 通过 | 失败 | 覆盖内容 |
|---------|--------|------|------|---------|
| test_game_data.gd | <N> | <N> | 0 | 存读档、货币增减、边界 |
| test_progression.gd | <N> | <N> | 0 | 解锁条件正/负向 |
| test_entities.gd | <N> | <N> | 0 | JSON 格式、实体属性 |
| test_<system>.gd × N | <N> | <N> | 0 | 数值规则、边界条件、幂等 |

### 集成测试（场景树 / 信号 / AutoLoad）

| 测试文件 | 用例数 | 通过 | 失败 | 覆盖内容 |
|---------|--------|------|------|---------|
| test_scene_tree.gd | <N> | <N> | 0 | 主场景实例化、关键节点路径 |
| test_autoload.gd | <N> | <N> | 0 | GameData 可访问、存读档往返 |
| test_signals.gd | <N> | <N> | 0 | 关键信号发出、按钮连接验证 |

## 覆盖说明
- 核心数值逻辑：✅ 已覆盖
- 边界条件与异常处理：✅ 已覆盖
- 存读档往返：✅ 已覆盖（含无存档文件场景）
- 场景树结构：✅ 已覆盖
- AutoLoad 运行时可访问性：✅ 已覆盖
- 关键信号触发与连接：✅ 已覆盖
```

```
[07] ✅ 测试全部通过（<N>/<N>）
```

### Step（最后）：写入流水线状态

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
print('[07] pipeline_status 已更新')
"
```

---

## 完成标准

**单元测试**
- [ ] 测试文件数 ≥ `project_context.test_priorities` 条目数 + 3（通用测试）
- [ ] 每个文件 ≥ 4 个用例

**集成测试（三个文件必须存在）**
- [ ] `test_scene_tree.gd`：覆盖主场景所有有 `script` 字段的节点路径
- [ ] `test_autoload.gd`：覆盖 GameData 所有 `key_variables` + 存读档往返
- [ ] `test_signals.gd`：覆盖各系统策划案第六章定义的关键信号

**执行结果**
- [ ] 所有测试通过（0 失败）
- [ ] `reports/test_results.txt` 已生成（GUT 原始输出）
- [ ] `reports/test_report.md` 已生成，包含单元测试和集成测试两张分类表
