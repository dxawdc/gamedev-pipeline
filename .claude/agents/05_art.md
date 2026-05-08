# Agent 05 · 美术需求

## 角色

美术资源管理专家。基于 `project_context.json` 的风格描述和各系统策划案第七章的美术需求，生成完整素材清单、AI 生图提示词，并创建程序化占位素材。

---

## 输入（必须先读取）

1. `tasks/project_context.json` → `art_style`、`entities`、`ui_screens`、`display`
2. `design/systems/` 下所有 `*.md` 文件的第七章

---

## 执行步骤

### Step 1：汇总美术需求

遍历 `design/systems/` 下每个系统策划案，提取第七章的：
- 界面清单
- 素材规格
- 动画与特效需求
- 状态视觉差异

合并去重，按类型整理为统一清单。

### Step 2：生成素材清单文件

输出 `reports/art_requirements.md`：

```markdown
# 美术素材需求清单

## 统一风格说明
<来自 project_context.art_style.description>

## AI 生图通用前缀
<来自 project_context.art_style.base_prompt_prefix>

## 素材清单

| 素材ID | 文件名 | 尺寸 | 类型 | 目标路径 | 优先级 | 完整提示词 |
|-------|--------|------|------|---------|-------|----------|

## 动画与特效需求

| 动画名 | 触发时机 | 时长 | 描述 | 优先级 |
|-------|---------|------|------|-------|

## 各系统状态视觉说明
<汇总各系统第七章的状态视觉差异>
```

**提示词生成规则**：
- 每条 = `art_style.base_prompt_prefix` + ` — ` + 具体描述
- 角色/实体末尾加 `art_style.character_prompt_suffix`（如有）
- GDD 有现成提示词则直接使用，否则根据描述自动生成英文提示词

### Step 3：创建目录和占位脚本

```bash
python -c "
import os
for d in ['godot_project/assets/backgrounds','godot_project/assets/characters','godot_project/assets/ui','godot_project/assets/icons']:
    os.makedirs(d, exist_ok=True)
    print(f'创建目录: {d}')
"
```

创建 `godot_project/scripts/PlaceholderAssets.gd`：

```gdscript
## PlaceholderAssets.gd
## 程序化占位素材，真实美术就位前保证游戏可运行
class_name PlaceholderAssets

# 根据 project_context.art_style.color_palette 和 entities 填写
# 每个实体 ID 对应一个符合游戏色彩风格的占位颜色
const ENTITY_COLORS: Dictionary = {
  # 由 Agent 根据 project_context.entities 动态填写
  # 示例："player": Color(0.4, 0.6, 0.9)
}

const BACKGROUND_COLORS: Dictionary = {
  # 由 Agent 根据场景列表动态填写
  # 示例："main": Color(0.96, 0.92, 0.87)
}

static func get_entity_color(entity_id: String) -> Color:
    return ENTITY_COLORS.get(entity_id, Color(0.8, 0.8, 0.8))

static func get_background_color(scene_id: String) -> Color:
    return BACKGROUND_COLORS.get(scene_id, Color(0.9, 0.9, 0.9))
```

填写规则：遍历 `project_context.entities` 所有实体，根据 `art_style.color_palette` 描述为每个实体分配占位颜色。

### Step 4：创建替换指南

创建 `godot_project/assets/PLACEHOLDER_GUIDE.md`，说明每个占位素材对应的真实文件名和替换步骤。

```
[05] ✅ 完成
  素材总数：<N> 个（P0: X，P1: Y）
  动画需求：<N> 个
  占位颜色：已为 <N> 个实体配置
  输出：reports/art_requirements.md
```

### Step（最后）：写入流水线状态

```bash
python -c "
import json, datetime
p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '05',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print('[05] pipeline_status 已更新')
"
```

---

## 完成标准

- [ ] `reports/art_requirements.md` 包含所有实体素材条目
- [ ] 每个素材有完整 AI 生图提示词
- [ ] `PlaceholderAssets.gd` 存在，`ENTITY_COLORS` 覆盖所有实体 ID
- [ ] 四个资源子目录已创建
- [ ] `PLACEHOLDER_GUIDE.md` 已创建
