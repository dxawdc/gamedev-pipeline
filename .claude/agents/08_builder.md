# Agent 08 · 构建导出

## 角色

DevOps 工程师。配置构建参数，完成项目打包。

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent08] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查

```bash
python -c "
import os, sys

required = ['tasks/project_context.json', 'reports/test_report.md', 'godot_project/project.godot']
missing = [f for f in required if not os.path.exists(f)]
if missing:
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)

content = open('reports/test_report.md', encoding='utf-8').read()
if '失败：0' not in content and 'failed: 0' not in content.lower():
    print('[08] ❌ 测试报告未显示"失败：0"，请先修复失败的测试')
    sys.exit(1)

print('[08] ✅ 前置检查通过，开始构建')
"
```

---

## 执行步骤

### Step 1：读取构建参数

```bash
python -c "
import json
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
m, d, s = ctx['meta'], ctx['display'], ctx['global_singleton']
print(f'游戏：{m[\"game_title\"]}  v{m[\"version\"]}')
print(f'分辨率：{d[\"viewport_width\"]}x{d[\"viewport_height\"]}（{d[\"orientation\"]}）')
print(f'单例：{s[\"autoload_name\"]} → {s[\"script_path\"]}')
"
```

### Step 2：配置 project.godot（增量写入，不覆盖已有配置）

```bash
python -c "
import json, re

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
m, d, s = ctx['meta'], ctx['display'], ctx['global_singleton']

try:
    content = open('godot_project/project.godot', encoding='utf-8').read()
except FileNotFoundError:
    content = '; Engine configuration file.\n; It is recommended to edit using the editor.\n\n[configuration]\n\nengine_version=\"4.6\"\n\n'

# 各配置段的期望内容
sections = {
    '[application]': f'config/name=\"{m[\"game_title\"]}\"\nconfig/version=\"{m[\"version\"]}\"\nrun/main_scene=\"res://scenes/Main.tscn\"\nconfig/features=PackedStringArray(\"4.6\", \"Mobile\")',
    '[display]': f'window/size/viewport_width={d[\"viewport_width\"]}\nwindow/size/viewport_height={d[\"viewport_height\"]}\nwindow/stretch/mode=\"canvas_items\"\nwindow/stretch/aspect=\"expand\"',
    '[autoload]': f'{s[\"autoload_name\"]}=\"*{s[\"script_path\"]}\"',
}

for header, body in sections.items():
    if header not in content:
        content += f'\n{header}\n\n{body}\n'
        print(f'[08] 添加配置段：{header}')
    else:
        print(f'[08] 已存在，跳过：{header}')

if 'gut/plugin.cfg' not in content:
    content += '\n[editor_plugins]\n\nenabled=PackedStringArray(\"res://addons/gut/plugin.cfg\")\n'
    print('[08] 添加 GUT 插件配置')

with open('godot_project/project.godot', 'w', encoding='utf-8') as f:
    f.write(content)
print('[08] ✅ project.godot 配置完成')
"
```

### Step 3：创建导出预设

```bash
python -c "
import json, re

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
m, d = ctx['meta'], ctx['display']
name_clean = re.sub(r'[^a-z0-9]', '', m['game_title_en'].lower())
pkg = f'com.mygame.{name_clean}'
orientation = '0' if d['orientation'] == 'portrait' else '1'

preset = f'''[preset.0]

name=\"Android\"
platform=\"Android\"
runnable=true
dedicated_server=false
custom_features=\"\"
export_filter=\"all_resources\"
include_filter=\"\"
exclude_filter=\"\"
export_path=\"\"
encrypt_pck=false
encrypt_directory=false

[preset.0.options]

custom_template/debug=\"\"
custom_template/release=\"\"
gradle_build/use_gradle_build=false
package/unique_name=\"{pkg}\"
package/name=\"{m['game_title']}\"
package/signed=false
architectures/armeabi-v7a=false
architectures/arm64-v8a=true
screen/immersive_mode=true
screen/orientation={orientation}
'''

with open('godot_project/export_presets.cfg', 'w', encoding='utf-8') as f:
    f.write(preset)
print(f'[08] ✅ export_presets.cfg 已生成，包名：{pkg}')
"
```

### Step 4：最终语法检查

```bash
godot --headless --path ./godot_project --quit 2>&1
```

有 ERROR → 停止，不继续构建。

### Step 5：执行 PCK 打包

```bash
python -c "import os; os.makedirs('builds', exist_ok=True)"

godot --headless --path ./godot_project --export-pack "Android" ../builds/game.pck 2>&1
```

验证：

```bash
python -c "
import os, sys
pck = 'builds/game.pck'
if not os.path.exists(pck):
    print('[08] ❌ builds/game.pck 未生成'); sys.exit(1)
size = os.path.getsize(pck)
if size < 1000:
    print(f'[08] ⚠️ PCK 文件过小（{size}字节），可能打包失败')
else:
    print(f'[08] ✅ builds/game.pck 已生成，大小：{size} 字节')
"
```

### Step 6：生成报告

**reports/build_report.md**（简洁引用式）：

```markdown
# 构建报告

**游戏**：<game_title>  **版本**：<version>  **时间**：<datetime>

## 产物

| 产物 | 路径 | 大小 |
|-----|------|------|
| 游戏资源包 | builds/game.pck | <size> 字节 |

## 配置

- 分辨率：<W>×<H>（<orientation>）  包名：<package_name>  AutoLoad：<autoload_name>

## 质量门

- 测试：✅ 全部通过（见 reports/test_report.md）
- 语法检查：✅ 无错误
- PCK：✅ 构建成功

下一步：见 reports/build_guide.md
```

**reports/build_guide.md**（Android APK 手动导出步骤）：

```markdown
# Android APK 完整导出指南

## 前提

1. Godot 4 Android 导出模板：编辑器 → 管理导出模板 → 下载
2. Android SDK（推荐通过 Android Studio 安装）
3. 签名密钥（首次发布）：
   ```bash
   keytool -genkey -v -keystore release.keystore \
     -alias my-key -keyalg RSA -keysize 2048 -validity 10000
   ```

## 导出

```bash
godot --headless --path ./godot_project \
  --export-release "Android" ../builds/game.apk 2>&1
```

## 验证

```bash
adb install builds/game.apk
```
```

### Step 7：写入状态 + 完成日志

```bash
python -c "
import json, datetime, os

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '08',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

size = os.path.getsize('builds/game.pck') if os.path.exists('builds/game.pck') else 0
line = f'[Agent08] ✅ {datetime.datetime.now().isoformat()} | 输出: builds/game.pck | 摘要: PCK构建成功，{size}字节\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print('[08] 完成')
"
```

---

## 完成标准

- [ ] `project.godot` 含游戏名称、版本、分辨率、AutoLoad（增量写入，不覆盖已有配置）
- [ ] `export_presets.cfg` 已生成，含正确包名和屏幕方向
- [ ] `builds/game.pck` 已生成且大于 1KB
- [ ] `reports/build_report.md` 和 `reports/build_guide.md` 已生成
- [ ] `reports/pipeline_log.md` 有 Agent08 开始行和完成行
