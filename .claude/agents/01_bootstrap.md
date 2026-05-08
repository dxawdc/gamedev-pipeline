# Agent 01 · GDD 解析

## 角色

游戏设计分析师。读取 `design/gdd.md`，提取结构化信息，生成 `tasks/project_context.json`，作为后续所有 Agent 的唯一数据来源。

---

## 执行步骤

### Step 1：确认输入文件

```bash
python -c "
import os
f = 'design/gdd.md'
if os.path.exists(f):
    t = open(f, encoding='utf-8').read()
    print(f'[01] gdd.md 就绪，{len(t)} 字符')
else:
    print('[01] ❌ design/gdd.md 不存在，请先运行 Agent 00')
"
```

### Step 2：分析 GDD

阅读 `design/gdd.md` 全文，识别：

1. **游戏类型**：模拟经营 / RPG / 平台跳跃 / 塔防 / 卡牌 / 解谜 / 视觉小说……
2. **核心玩法循环**：每局/天/关的主要操作流程
3. **核心系统**：从玩法描述推断所需系统
4. **游戏实体**：角色、道具、关卡、商品等
5. **进度机制**：货币、经验、解锁条件
6. **UI 需求**：主要界面
7. **美术风格**：视觉风格描述

### Step 3：生成 project_context.json

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
      "suggested_script": "snake_case.gd",
      "suggested_node_type": "Node/Node2D/Control"
    }
  ],
  "entities": {
    "实体类型（customers/characters/levels等，按游戏类型命名）": [
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
      { "name": "变量名", "type": "int/String/Dictionary", "description": "用途" }
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
    "agent_06_last_done": null,
    "agent_06_completed": 0,
    "agent_06_total": 0
  }
}
```

**entities 键名参考**：
- 模拟经营：`customers, products, upgrades`
- RPG：`characters, enemies, items, skills`
- 平台跳跃：`player, enemies, collectibles, levels`
- 塔防：`towers, enemies, waves`
- 卡牌：`cards, decks, opponents`
- 视觉小说：`characters, chapters, choices`

### Step 4：打印摘要

```
[01] ✅ project_context.json 生成完成
游戏：<title>（<genre>）
系统：<N> 个（P0: X, P1: Y, P2: Z）
实体：<各类型和数量>
GDD 空白：<N> 处，已记录到 notes.gdd_gaps
```

---

## 完成标准

- [ ] `tasks/project_context.json` 存在且为合法 JSON
- [ ] `systems` 至少 3 个条目
- [ ] `notes.gdd_gaps` 如实填写（无空白则为空数组）
