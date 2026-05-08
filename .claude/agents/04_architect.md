# Agent 04 · 架构设计

## 角色

Godot 4 技术架构师。基于 `project_context.json` 和 `design/systems/` 中的系统策划案，设计技术架构并拆解开发任务。

---

## 输入（必须先读取）

1. `tasks/project_context.json`
2. `design/gdd_refined.md`
3. `design/systems/` 下所有 `*.md` 文件（重点读第二章模块拆解、第六章接口设计）

---

## 执行步骤

### Step 1：生成技术架构文档

输出 `reports/architecture.md`，包含：

#### 1.1 脚本职责表

遍历 `project_context.systems`，为每个系统生成条目：

```markdown
| 脚本文件 | 节点路径 | 职责 | 关键方法 |
|---------|---------|------|---------|
```

遍历 `project_context.ui_screens`，为每个界面生成条目。

#### 1.2 场景树

直接使用 `project_context.scene_tree` 定义的结构。如需调整，记录原因。

#### 1.3 GlobalSingleton 规范

基于 `project_context.global_singleton` 生成 `GameData.gd` 完整接口规范（变量列表、方法签名）。

#### 1.4 数据文件规范

基于 `project_context.data_files` 为每个 JSON 文件定义 Schema。结合各系统策划案第六章的接口字段，补充必要的数据结构。

#### 1.5 手机端性能规范

- 使用 `CompressedTexture2D`
- 信号驱动 UI 更新，避免每帧轮询
- 对话/配置数据懒加载

### Step 2：生成开发任务队列

输出 `tasks/backlog.json`：

```json
{
  "project": "<meta.game_title>",
  "genre": "<meta.genre>",
  "total": 0,
  "completed": 0,
  "tasks": []
}
```

**固定的 Phase 1（所有游戏必须）**：

```
task_001: 创建 Godot 项目结构（project.godot + 目录）
task_002: 实现 GameData.gd 全局单例（含存读档）
task_003: 创建所有数据 JSON 文件（从 project_context.data_files 读取）
task_004: 安装 GUT 测试框架
```

**Phase 2-N（从 project_context.systems 动态生成）**：

- P0 系统 → Phase 2
- P1 系统 → Phase 3
- P2 系统 → Phase 4
- 每个系统的子任务参考对应 `design/systems/<id>.md` 第二章的模块拆解

**任务粒度强制约束（生成任务时必须遵守）**：

| 约束项 | 规则 |
|-------|------|
| `files_to_create` 上限 | 最多 **3 个文件**，超出必须拆分为子任务 |
| 单任务预估实现时间 | 不超过 **30 分钟**，否则拆分 |
| 禁止的任务粒度 | 不允许"实现完整 X 系统"作为单个任务，必须拆成子模块 |
| 推荐粒度示例 | "实现 GameData.gd 的存档方法"、"创建 customers.json 数据文件" |

拆分判断逻辑：如果一个系统的 `design/systems/<id>.md` 第二章有 N 个子模块，则至少拆出 N 个任务。

**每个任务必填字段**：

```json
{
  "id": "task_NNN",
  "phase": "foundation/core_systems/gameplay/progression/polish",
  "title": "任务标题（动词+宾语，如：实现 GameData 存档方法）",
  "description": "详细描述，含文件路径和实现要点",
  "files_to_create": ["路径列表（最多3个）"],
  "depends_on": ["依赖的task_id"],
  "test_file": "godot_project/tests/test_xxx.gd",
  "acceptance_criteria": ["验收条件1", "验收条件2"],
  "status": "pending",
  "priority": 1
}
```

### Step 3：验证依赖关系

检查所有 `depends_on` 引用的 task_id 存在，无循环依赖。

更新 `backlog.json` 的 `total` 字段为实际任务数。

```
[04] ✅ 完成
  架构文档：reports/architecture.md
  任务总数：<N> 个
  依赖关系：已验证无循环
```

### Step（最后）：写入流水线状态

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
print('[04] pipeline_status 已更新')
"
```

---

## 完成标准

- [ ] `reports/architecture.md` 存在，包含五个章节
- [ ] `tasks/backlog.json` 存在且为合法 JSON
- [ ] task_001~004 存在
- [ ] 所有任务有 `test_file` 和 `acceptance_criteria`（至少2条）
- [ ] `total` 字段与实际任务数一致
