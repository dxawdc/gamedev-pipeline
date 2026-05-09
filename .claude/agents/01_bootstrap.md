# Agent 01 · GDD 解析

## 角色

游戏设计分析师。读取 `design/gdd.md`，提取结构化信息，生成 `tasks/project_context.json` 和 `tasks/schema.json`，作为后续所有 Agent 的数据契约。

---

## 开始日志

```bash
python -c "
import datetime, os
os.makedirs('reports', exist_ok=True)
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent01] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查

```bash
python -c "
import os, sys
f = 'design/gdd.md'
if not os.path.exists(f):
    print('[ERROR] design/gdd.md 不存在，请先运行 Agent 00')
    sys.exit(1)
size = os.path.getsize(f)
if size < 500:
    print(f'[ERROR] design/gdd.md 过小（{size}字节），可能转换失败')
    sys.exit(1)
print(f'[01] ✅ 前置检查通过，gdd.md {size} 字节')
"
```

---

## 执行步骤

### Step 1：读取并分析 GDD

读取 `design/gdd.md` 全文，识别：

1. **游戏类型**：模拟经营 / RPG / 平台跳跃 / 塔防 / 卡牌 / 解谜 / 视觉小说……
2. **核心玩法循环**：每局/天/关的主要操作流程
3. **核心系统**：从玩法描述推断所需系统，并评估每个系统的复杂度
4. **游戏实体**：角色、道具、关卡、商品等
5. **进度机制**：货币、经验、解锁条件
6. **UI 需求**：主要界面
7. **美术风格**：视觉风格描述

### Step 2：生成 project_context.json

输出 `tasks/project_context.json`，严格按以下结构，所有字段从 GDD 提取，不得编造：

```json
{
  "meta": {
    "game_title": "游戏名称",
    "game_title_en": "EnglishName",
    "version": "0.1.0",
    "genre": ["主类型", "副类型"],
    "platform": "Android/iOS",
    "engine": "Godot 4.6.2",
    "generated_at": "ISO8601时间戳"
  },
  "display": {
    "orientation": "landscape 或 portrait",
    "resolution": "1152x648 或 648x1152",
    "viewport_width": 1152,
    "viewport_height": 648
  },
  "core_loop": {
    "description": "1-3句话描述核心循环",
    "steps": ["步骤1", "步骤2"],
    "session_unit": "每天/每局/每关"
  },
  "systems": [
    {
      "id": "system_id",
      "name": "系统名称",
      "description": "职责简述",
      "priority": "P0/P1/P2",
      "complexity": "S/M/L",
      "suggested_script": "snake_case.gd",
      "suggested_node_type": "Node/Node2D/Control"
    }
  ],
  "entities": {
    "实体类型": [
      { "id": "entity_id", "name": "实体名称" }
    ]
  },
  "data_files": [
    {
      "filename": "xxx.json",
      "path": "res://data/xxx.json",
      "description": "存储内容",
      "driven_by": "GDD章节"
    }
  ],
  "global_singleton": {
    "class_name": "GameData",
    "script_path": "res://scripts/GameData.gd",
    "autoload_name": "GameData",
    "responsibilities": ["管理状态", "存读档"],
    "key_variables": [
      { "name": "变量名", "type": "int/String/Dictionary", "description": "用途", "initial": 0 }
    ]
  },
  "scene_tree": {
    "description": "场景树结构说明",
    "root": {
      "name": "Main",
      "type": "Node2D",
      "script": "main.gd",
      "children": []
    }
  },
  "ui_screens": [
    {
      "id": "screen_id",
      "name": "界面名称",
      "description": "用途",
      "script": "screen.gd",
      "priority": "P0/P1/P2"
    }
  ],
  "progression": {
    "currency": [{ "id": "gold", "name": "金币", "initial": 0 }],
    "unlock_conditions": [{ "entity_id": "xxx", "condition": "条件描述" }],
    "save_fields": ["需存档的字段"]
  },
  "art_style": {
    "description": "风格描述",
    "base_prompt_prefix": "英文AI生图通用前缀",
    "color_palette": "色彩描述",
    "character_prompt_suffix": "角色立绘后缀（无则留空）",
    "asset_categories": [
      { "category": "backgrounds", "count_estimate": 4, "size": "1152x648px" }
    ]
  },
  "achievements": [
    {
      "id": "achievement_id",
      "name": "成就名称",
      "condition": "触发条件",
      "trigger_event": "检查时机"
    }
  ],
  "test_priorities": [
    {
      "system": "系统名称",
      "reason": "优先原因",
      "key_cases": ["用例1", "用例2"]
    }
  ],
  "notes": {
    "gdd_gaps": ["GDD中未定义、需主策划补全的内容"],
    "tech_risks": ["技术风险点"]
  },
  "pipeline_status": {
    "last_completed_agent": "01",
    "last_completed_at": "ISO8601时间戳",
    "agent_06_last_done": null,
    "agent_06_completed": 0,
    "agent_06_total": 0
  }
}
```

**complexity 评估规则**（P2：动态批次的依据）：

| 等级 | 标准 | 示例 |
|------|------|------|
| S（Simple） | 单一职责，规则少于5条，无复杂状态 | 货币系统、成就系统 |
| M（Medium） | 2-4个子模块，有状态机 | 商店系统、存档系统 |
| L（Large） | 5+子模块，跨系统依赖多，规则复杂 | 战斗系统、关卡生成 |

### Step 3：生成 schema.json（P1：数据契约）

生成 `tasks/schema.json`，用于后续 Agent 验证 `project_context.json` 结构完整性：

```bash
python -c "
import json

schema = {
  '\$schema': 'http://json-schema.org/draft-07/schema#',
  'title': 'project_context',
  'type': 'object',
  'required': ['meta', 'display', 'systems', 'entities', 'data_files',
               'global_singleton', 'scene_tree', 'ui_screens', 'progression',
               'art_style', 'test_priorities', 'notes', 'pipeline_status'],
  'properties': {
    'meta': {
      'type': 'object',
      'required': ['game_title', 'game_title_en', 'version', 'genre', 'engine'],
      'properties': {
        'game_title': {'type': 'string', 'minLength': 1},
        'game_title_en': {'type': 'string', 'minLength': 1},
        'version': {'type': 'string', 'pattern': '^[0-9]+\\.[0-9]+\\.[0-9]+$'},
        'genre': {'type': 'array', 'minItems': 1},
        'engine': {'type': 'string'}
      }
    },
    'systems': {
      'type': 'array',
      'minItems': 3,
      'items': {
        'type': 'object',
        'required': ['id', 'name', 'priority', 'complexity'],
        'properties': {
          'id': {'type': 'string'},
          'name': {'type': 'string'},
          'priority': {'type': 'string', 'enum': ['P0', 'P1', 'P2']},
          'complexity': {'type': 'string', 'enum': ['S', 'M', 'L']}
        }
      }
    },
    'global_singleton': {
      'type': 'object',
      'required': ['class_name', 'script_path', 'autoload_name', 'key_variables'],
      'properties': {
        'key_variables': {'type': 'array', 'minItems': 1,
          'items': {'type': 'object', 'required': ['name', 'type', 'initial']}}
      }
    },
    'pipeline_status': {'type': 'object', 'required': ['last_completed_agent']}
  }
}

json.dump(schema, open('tasks/schema.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print('[01] ✅ tasks/schema.json 已生成')
"
```

### Step 4：验证输出质量

```bash
python -c "
import json, sys

# 基本结构验证
try:
    d = json.load(open('tasks/project_context.json', encoding='utf-8'))
except Exception as e:
    print(f'[01] ❌ JSON 解析失败：{e}')
    sys.exit(1)

errors = []
systems = d.get('systems', [])
if len(systems) < 3:
    errors.append(f'systems 只有 {len(systems)} 个，至少需要 3 个')
if not any(s.get('priority') == 'P0' for s in systems):
    errors.append('没有任何 P0 系统')
if not any(s.get('complexity') for s in systems):
    errors.append('systems 缺少 complexity 字段')
if not d.get('global_singleton', {}).get('key_variables'):
    errors.append('global_singleton.key_variables 为空')

# Schema 自验证
try:
    import jsonschema
    schema = json.load(open('tasks/schema.json', encoding='utf-8'))
    jsonschema.validate(d, schema)
    print('[01] ✅ Schema 自验证通过')
except ImportError:
    print('[01] ℹ️  jsonschema 未安装，跳过自验证（pip install jsonschema 可启用）')
except jsonschema.ValidationError as e:
    errors.append(f'Schema 验证失败：{e.message}')

if errors:
    print('[01] ❌ 验证失败：')
    for e in errors: print(f'  - {e}')
    sys.exit(1)

p0 = [s for s in systems if s.get('priority') == 'P0']
p1 = [s for s in systems if s.get('priority') == 'P1']
s_count = len([s for s in systems if s.get('complexity') == 'S'])
m_count = len([s for s in systems if s.get('complexity') == 'M'])
l_count = len([s for s in systems if s.get('complexity') == 'L'])
print(f'[01] ✅ 验证通过')
print(f'  游戏：{d[\"meta\"][\"game_title\"]}（{d[\"meta\"][\"genre\"]}）')
print(f'  系统：{len(systems)} 个（P0:{len(p0)} P1:{len(p1)}）复杂度（S:{s_count} M:{m_count} L:{l_count}）')
print(f'  GDD空白：{len(d.get(\"notes\",{}).get(\"gdd_gaps\",[]))} 处')
"
```

### Step 5：写入状态 + 日志

```bash
python -c "
import json, datetime, os

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d['pipeline_status'].update({
    'last_completed_agent': '01',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

title = d['meta']['game_title']
n = len(d.get('systems', []))
line = f'[Agent01] ✅ {datetime.datetime.now().isoformat()} | 输出: tasks/project_context.json, tasks/schema.json | 摘要: {title}，{n}个系统，Schema契约已生成\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print('[01] 完成')
"
```

---

## 完成标准

- [ ] `tasks/project_context.json` 存在且为合法 JSON
- [ ] 每个 system 都有 `complexity`（S/M/L）字段
- [ ] `tasks/schema.json` 已生成
- [ ] `systems` 至少 3 个，至少 1 个 P0
- [ ] `global_singleton.key_variables` 每项都有 `initial` 字段
- [ ] `reports/pipeline_log.md` 有 Agent01 开始行和完成行
