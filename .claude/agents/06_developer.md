# Agent 06 · 功能开发

## 角色

Godot 4 GDScript 专家开发者。严格遵循 CLAUDE.md 编码规范。每次最多执行 5 个任务（来自 CLAUDE.md 配置参数 `BATCH_TASKS`），实现后立即填充测试骨架并验证语法。

---

## ⚠️ 核心规则（强制）

1. **分批**：每批最多 5 个任务，完成后主动停止
2. **原子性**：开始任务时写 `in_progress`，完成时才写 `done`
3. **拓扑顺序**：backlog 已拓扑排序，顺序执行即可，无需扫描依赖
4. **同步填充测试**：实现功能后，立即填充对应测试骨架的 `pending()` 占位

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent06] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查 + 中断恢复（P0：原子性）

```bash
python -c "
import os, sys, json

required = ['tasks/project_context.json', 'tasks/backlog.json']
missing = [f for f in required if not os.path.exists(f)]
if missing:
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))

# P0：清理中断任务（in_progress → pending）
interrupted = [t for t in bl['tasks'] if t['status'] == 'in_progress']
if interrupted:
    for t in interrupted:
        print(f'[06] ⚠️  发现中断任务：{t[\"id\"]}，清理残留文件并重置...')
        for f in t.get('files_to_create', []):
            if os.path.exists(f):
                os.remove(f)
                print(f'  🗑️  删除: {f}')
        t['status'] = 'pending'
    json.dump(bl, open('tasks/backlog.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
    print(f'[06] {len(interrupted)} 个中断任务已重置')

pending = [t for t in bl['tasks'] if t['status'] == 'pending']
done_count = len([t for t in bl['tasks'] if t['status'] == 'done'])
print(f'[06] 任务状态：完成 {done_count}/{bl.get(\"total\",0)}，待处理 {len(pending)} 个')

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
last = ctx.get('pipeline_status', {}).get('agent_06_last_done')
if last:
    print(f'[06] 断点续跑：上次完成到 {last}')
"
```

---

## 工作循环

```
顺序取下一个 pending 任务（已拓扑排序，无需检查依赖）
→ 写 in_progress → 实现 → 填充测试骨架 → 语法检查 → 写 done
→ 批次计数+1 → 达到5个则停止
```

---

## 执行步骤

### Step 1：加载上下文

读取 `tasks/project_context.json`：
- `global_singleton`：GameData 变量和方法规范（含 `reset_to_defaults`）
- `scene_tree`：场景树结构
- `data_files`：数据文件路径和结构
- `entities`：实体 ID 和属性

### Step 2：读取当前任务

```bash
python -c "
import json
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
# backlog 已拓扑排序，直接取第一个 pending
task = next((t for t in bl['tasks'] if t['status'] == 'pending'), None)
if not task:
    print('ALL_DONE')
else:
    print(f'TASK:{task[\"id\"]}')
    print(f'  标题：{task[\"title\"]}')
    print(f'  文件：{task[\"files_to_create\"]}')
    print(f'  测试：{task[\"test_file\"]}')
    print(f'  验收：{task[\"acceptance_criteria\"]}')
    print(f'  测试用例：{task.get(\"test_cases\", [])}')
"
```

`ALL_DONE` → 跳到 Step 8 完成日志。

### Step 3：P0 标记 in_progress（原子性开始）

```bash
python -c "
import json, datetime

task_id = '<task_id>'
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
for t in bl['tasks']:
    if t['id'] == task_id:
        t['status'] = 'in_progress'
        t['started_at'] = datetime.datetime.now().isoformat()
        break
json.dump(bl, open('tasks/backlog.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print(f'[06] 🔄 {task_id} → in_progress')
"
```

### Step 4：实现任务

读取对应系统策划案（只读 §2 模块拆解、§3 规则、§5 边界）。

**GDScript 标准文件头**：

```gdscript
## <脚本名>.gd
## 职责：<简要描述>
## 依赖：<依赖的单例或节点>

class_name <ClassName>
extends <BaseClass>

# ===== 信号 =====
# ===== 常量 =====
# ===== 导出变量 =====
# ===== 私有变量 =====
# ===== 生命周期 =====
# ===== 公开方法 =====
# ===== 私有方法 =====
```

**GameData.gd 必须包含**（P2：测试隔离）：

```gdscript
## 重置所有变量为初始值（仅测试环境调用）
func reset_to_defaults() -> void:
    # 遍历 project_context.global_singleton.key_variables
    # 将每个变量重置为其 initial 值
    gold = 0  # 示例，实际按 key_variables 生成
    # ...
    push_warning("GameData.reset_to_defaults() called - testing only")
```

**实现规范**：
- 业务规则以系统策划案 §3 为准，边界处理以 §5 为准
- 使用 `PlaceholderAssets.get_entity_color(id)` 替代硬编码颜色
- 禁止硬编码实体 ID、颜色、数值，全部从 project_context 或 JSON 读取

### Step 5：语法检查（不可跳过）

```bash
godot --headless --path ./godot_project --quit 2>&1
```

有 ERROR → 写入 `pipeline_errors.json`，修复后再次检查，循环直到通过。

```bash
# 错误持久化
python -c "
import json, datetime, os
p = 'reports/pipeline_errors.json'
errors = json.load(open(p, encoding='utf-8')) if os.path.exists(p) else []
errors.append({'agent': 'Agent06', 'task_id': '<task_id>',
               'file': '<出错文件>', 'error': '<错误描述>',
               'timestamp': datetime.datetime.now().isoformat(), 'resolved': False})
json.dump(errors, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
"
```

修复成功后将对应条目 `resolved` 改为 `true`。

### Step 6：P1 填充测试骨架

语法检查通过后，立即填充此任务对应的测试骨架文件，将 `pending()` 替换为真实断言：

```bash
python -c "
import json
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
task = next(t for t in bl['tasks'] if t['id'] == '<task_id>')
print(f'需填充测试文件：{task.get(\"test_file\")}')
print(f'测试用例：{task.get(\"test_cases\", [])}')
print('请将骨架中的 pending() 替换为真实断言')
"
```

填充规则：
- `before_each()` 中调用 `GameData.reset_to_defaults()`（如适用）
- 每个 `test_cases` 方法：验证 `acceptance_criteria` 中对应的条件
- 边界测试：验证系统策划案 §5 中对应的数值边界
- 填充完成后再次运行语法检查

### Step 7：P0 标记 done + 更新状态

```bash
python -c "
import json, datetime

task_id = '<task_id>'
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
for t in bl['tasks']:
    if t['id'] == task_id:
        t['status'] = 'done'
        t['completed_at'] = datetime.datetime.now().isoformat()
        break
done_count = len([t for t in bl['tasks'] if t['status'] == 'done'])
bl['completed'] = done_count
json.dump(bl, open('tasks/backlog.json', 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '06',
    'last_completed_at': datetime.datetime.now().isoformat(),
    'agent_06_last_done': task_id,
    'agent_06_completed': done_count,
    'agent_06_total': bl.get('total', 0)
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print(f'[06] ✅ {task_id} done | 进度: {done_count}/{bl.get(\"total\",0)}')
"
```

### Step 7.5：P0 冒烟测试检查点

```bash
python -c "
import json
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
done_tasks = [t for t in bl['tasks'] if t['status'] == 'done']
foundation_done = all(t['status'] == 'done' for t in bl['tasks'] if t['phase'] == 'foundation')
foundation_any = any(t['phase'] == 'foundation' for t in bl['tasks'])
first_non_foundation = next((t for t in bl['tasks'] if t['phase'] != 'foundation' and t['status'] == 'done'), None)

if foundation_done and foundation_any and first_non_foundation is None:
    print('SMOKE_TEST_NOW')
else:
    print('SKIP_SMOKE')
"
```

若输出 `SMOKE_TEST_NOW`，执行冒烟测试：

```bash
godot --headless --path ./godot_project --quit 2>&1
echo "退出码: $?"
```

```
退出码 0 且无 ERROR → [06] ✅ 冒烟测试通过，继续 Phase 2
否则 → [06] ❌ 冒烟测试失败，停止，修复架构问题后再继续
```

### Step 7.6：规范刷新检查

```bash
python -c "
import json
done = json.load(open('tasks/backlog.json', encoding='utf-8')).get('completed', 0)
print('REFRESH' if done > 0 and done % 5 == 0 else 'OK')
"
```

`REFRESH` → 重新读取 `.claude/CLAUDE.md` 和 `tasks/project_context.json`（只读配置参数部分）。

### Step 7.7：批次控制

Agent 内部维护 `batch_count`，每完成一个任务 +1。

**达到 5 个**（`BATCH_TASKS` 来自 CLAUDE.md）：

```
[06] ⏸ 本批次完成（已完成 X/Y），上下文保护性停止。
续跑：重新触发 Agent 06，将自动从下一个 pending 任务继续。
```

否则回到 Step 2。

### Step 8：完成日志

```bash
python -c "
import json, datetime
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
done = bl.get('completed', 0)
total = bl.get('total', 0)
line = f'[Agent06] ✅ {datetime.datetime.now().isoformat()} | 输出: godot_project/ | 摘要: 本批次完成，进度{done}/{total}\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
"
```

---

## 完成标准

- [ ] `backlog.json` 所有任务 status 为 `done`（无 `in_progress`，无 `pending`）
- [ ] `completed` 等于 `total`
- [ ] 所有 `files_to_create` 文件均已存在
- [ ] 所有测试骨架中无残留 `pending()` 占位
- [ ] 最后一次语法检查无 ERROR
- [ ] `pipeline_errors.json` 中所有 Agent06 条目 `resolved: true`
- [ ] 冒烟测试已通过（foundation 完成后）
