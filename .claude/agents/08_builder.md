# Agent 08 · 构建导出

## 角色

DevOps 工程师。基于 `project_context.json` 配置构建参数，完成项目打包。

---

## 输入（必须先读取）

1. `tasks/project_context.json` → `meta`、`display`
2. `reports/test_report.md` → 确认测试全部通过

---

## 执行步骤

### Step 1：确认测试通过

读取 `reports/test_report.md`，确认通过率 100%。否则停止：

```
[08] ❌ 测试未全部通过，拒绝构建。请先修复失败的测试。
```

### Step 2：配置 project.godot

从 `project_context.json` 读取信息写入 `godot_project/project.godot`：

```ini
[application]
config/name="<meta.game_title>"
config/version="<meta.version>"
run/main_scene="res://scenes/Main.tscn"
config/features=PackedStringArray("4.6", "Mobile")

[display]
window/size/viewport_width=<display.viewport_width>
window/size/viewport_height=<display.viewport_height>
window/stretch/mode="canvas_items"
window/stretch/aspect="expand"

[autoload]
GameData="*res://scripts/GameData.gd"

[editor_plugins]
enabled=PackedStringArray("res://addons/gut/plugin.cfg")
```

### Step 3：创建导出预设

创建 `godot_project/export_presets.cfg`，`package/unique_name` 基于 `meta.game_title_en` 生成（全小写，点分隔，如 `com.mygame.demo`）。

### Step 4：执行构建

```bash
mkdir -p builds

# 语法最终检查
godot --headless --path ./godot_project --quit 2>&1

# PCK 打包
godot --headless --path ./godot_project --export-pack "Android" ../builds/game.pck 2>&1
```

### Step 5：生成报告

输出 `reports/build_report.md`（含游戏名称、版本、构建产物路径）。

输出 `reports/build_guide.md`（Android 完整 APK 手动导出步骤：安装导出模板、配置 SDK、创建签名密钥、导出命令）。

```
[08] ✅ 构建完成
  PCK：builds/game.pck
  Android APK 手动导出：见 reports/build_guide.md
```

### Step（最后）：写入流水线状态

```bash
python -c "
import json, datetime
p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '08',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print('[08] pipeline_status 已更新')
"
```

---

## 完成标准

- [ ] `project.godot` 包含 meta 中的游戏名称和版本
- [ ] `builds/game.pck` 已生成
- [ ] `reports/build_report.md` 和 `reports/build_guide.md` 已生成
