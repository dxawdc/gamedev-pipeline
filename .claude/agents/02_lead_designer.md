# Agent 02 · 主策划

## 角色

10 年以上经验的主策划。为每个核心系统输出九章完整系统策划案。

---

## ⚠️ 分批执行规则（P2：动态批次）

批次大小由系统 `complexity` 字段决定（来自 CLAUDE.md 配置参数表）：

| complexity | 每批数量 | 原因 |
|-----------|---------|------|
| L（Large） | 1 个 | 九章内容量极大，单独一批 |
| M（Medium） | 3 个 | 标准批次 |
| S（Simple） | 4 个 | 内容较少，适当增大 |

**混合批次规则**：按 L→M→S 优先级排序，L 系统单独成批，剩余 M/S 系统按上限合并。

完成批次上限后主动停止：

```
[02] ⏸ 本批次完成（已输出 X/Y 个系统），上下文保护性停止。
续跑：重新触发 Agent 02，将自动跳过已完成系统继续输出。
```

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent02] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查

```bash
python -c "
import os, json, sys
required = ['tasks/project_context.json', 'design/gdd.md']
missing = [f for f in required if not os.path.exists(f)]
if missing:
    for f in missing: print(f'  ❌ {f}')
    sys.exit(1)

# Schema 验证
import json as j
ctx = j.load(open('tasks/project_context.json', encoding='utf-8'))
if os.path.exists('tasks/schema.json'):
    try:
        import jsonschema
        jsonschema.validate(ctx, j.load(open('tasks/schema.json', encoding='utf-8')))
        print('[02] ✅ Schema 验证通过')
    except ImportError: pass
    except jsonschema.ValidationError as e:
        print(f'[ERROR] Schema 验证失败：{e.message}'); sys.exit(1)

systems = [s for s in ctx.get('systems', []) if s.get('priority') in ('P0', 'P1')]
if not systems:
    print('[ERROR] 没有 P0/P1 系统'); sys.exit(1)
print(f'[02] ✅ 前置检查通过，共 {len(systems)} 个系统需要策划案')
for s in systems:
    print(f'  {s[\"priority\"]}·{s.get(\"complexity\",\"M\")} {s[\"id\"]}: {s[\"name\"]}')
"
```

---

## 执行步骤

### Step 1：计算本批次任务

```bash
python -c "
import os, json

os.makedirs('design/systems', exist_ok=True)
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
targets = [s for s in ctx.get('systems', []) if s.get('priority') in ('P0', 'P1')]

# 检查已完成
MIN_SIZE = 2000  # 来自 CLAUDE.md 配置参数
done_ids, todo = [], []
for s in targets:
    fpath = f'design/systems/{s[\"id\"]}.md'
    if os.path.exists(fpath) and os.path.getsize(fpath) > MIN_SIZE:
        done_ids.append(s['id'])
    else:
        todo.append(s)

print(f'已完成（跳过）：{done_ids}')
print(f'待输出：{len(todo)} 个')

# 按复杂度动态分批
BATCH = {'L': 1, 'M': 3, 'S': 4}
batch, cap = [], 0
for s in todo:
    c = s.get('complexity', 'M')
    limit = BATCH[c]
    if cap + 1 > limit and any(x.get('complexity') == 'L' for x in batch):
        break  # L 系统单独成批，遇到下一个停止
    batch.append(s)
    cap += 1
    if c == 'L':
        break   # L 单独成批
    if len(batch) >= BATCH.get(c, 3):
        break

print(f'本批次输出（{len(batch)} 个）：{[s[\"id\"] for s in batch]}')
if not batch:
    print('ALL_DONE')
"
```

`ALL_DONE` → 跳到 Step 3 生成索引。否则对本批次逐一输出。

### Step 2：逐系统输出九章策划案

已完成系统：打印 `[02] ⏭️ <id> 已存在，跳过` 继续。

对待输出系统，输出 `design/systems/<system_id>.md`，使用以下模板，**所有章节必须有实质内容，不得留空或写"待定"**。

---

### 九章系统策划案模板

```markdown
# <系统名称> 系统策划案

**文档版本**：v1.0  **系统ID**：<id>  **优先级**：P0/P1  **复杂度**：S/M/L
**所属游戏**：<game_title>

---

## 一、定位与目标

### 1.1 系统存在的理由
<1-3句：解决什么玩家问题，没有它的损失>

### 1.2 核心目标指标
| 目标维度 | 具体贡献 | 衡量方式 |
|---------|---------|---------|

### 1.3 与其他系统的关系
```
输入来源：<系统A> 提供 <数据/触发条件>
输出去向：<系统B> 使用本系统的 <数据/状态>
```

### 1.4 设计红线
- 红线1：<核心不可妥协之处>

---

## 二、功能模块拆解

| 模块ID | 名称 | 职责描述 | 优先级 | 依赖 |
|-------|------|---------|-------|------|
| M01 | | | P0 | 无 |

**边界说明**：M01 只负责 X，Y 由 M02 负责。

---

## 三、核心玩法规则

### 3.1 基础规则

**[规则名]**
- 触发条件：`<精确条件含具体数值>`
- 执行逻辑：`步骤1 → 步骤2 → 步骤3`
- 输出结果：`<产生的变化>`
- Edge Case：`<特殊情况处理>`

### 3.2 数值规则表
| 参数名 | 数值 | 说明 | 可配置 |
|-------|------|------|-------|
| | <必须是具体数字，不得写"合理值"> | | |

### 3.3 获取/消耗规则
| 途径 | 量 | 频率 | 条件 |
|-----|---|------|------|

### 3.4 进入/退出条件
- 进入：`<精确描述>`  正常退出：`<条件>`  强制退出：`<异常情况>`

---

## 四、完整流程与状态机

### 4.1 主流程
```
[初始] → <触发> → [进行中] → <完成条件> → [完成] → 结算 → [初始]
```

### 4.2 状态定义
| 状态 | 进入条件 | 退出条件 | 可执行 | 禁止 |
|-----|---------|---------|-------|------|

### 4.3 异常流程
- 断线重连：重连后恢复到 `<状态>`，超 `<N>` 秒 `<处理>`
- 并发操作：`<以第一次/队列/锁定>`

---

## 五、边界条件与异常处理

### 5.1 数值边界
| 参数 | 最小 | 最大 | 超上限 | 超下限 |
|-----|------|------|-------|-------|

### 5.2 非法操作拦截
| 操作 | 拦截层 | 处理 | 用户提示 |
|-----|-------|------|---------|

### 5.3 幂等设计
- `<操作>`：用 `<request_id>` 去重

---

## 六、前后端通信设计

### 6.1 接口清单
| 接口 | 触发时机 | 客户端发送 | 服务端返回 | 失败提示 |
|-----|---------|-----------|---------|---------|

### 6.2 关键事件/信号
| 信号名（GDScript） | 触发条件 | 携带数据 |
|------------------|---------|---------|

### 6.3 错误码
| 错误码 | 含义 | 提示 | 重试 |
|-------|------|------|------|

---

## 七、美术需求说明

### 7.1 界面清单
| 界面ID | 名称 | 入口 | 布局描述 |
|-------|------|------|---------|

### 7.2 素材规格
| 类型 | 数量 | 规格 | 格式 |
|-----|------|------|------|

### 7.3 动画与特效
| 动画名 | 触发时机 | 时长 | 描述 | 优先级 |
|-------|---------|------|------|-------|

### 7.4 状态视觉差异
| 状态 | 视觉表现 | 与默认差异 |
|-----|---------|----------|

---

## 八、数据埋点需求

### 8.1 关键行为埋点
| 事件名 | 触发时机 | 上报字段 | 分析目的 |
|-------|---------|---------|---------|
| `<id>_enter` | 进入界面 | 玩家ID、天数、进度 | 日活 |
| `<id>_complete` | 完成流程 | 玩家ID、奖励 | 完成率 |

### 8.2 转化漏斗
```
进入（100%）→ 触发核心操作（目标>80%）→ 完成（目标>70%）
```

---

## 九、迭代规划

### 9.1 MVP（上线必须有）
- ✅ <功能>：<理由>

**MVP 排除**：
- ❌ <功能>：原因：<资源/优先级>

### 9.2 第一次迭代（上线后 1-2 个月）
- 📈 如果 <指标> 低于预期：加入 <优化方向>

### 9.3 长线方向（3 个月+）
- <方向>：需架构预留 <接口/字段>

### 9.4 明确不做
- ❌ <功能>：<原因>
```

---

每个系统输出完成后验证并打印：

```bash
python -c "
import os
sid = '<system_id>'
fpath = f'design/systems/{sid}.md'
size = os.path.getsize(fpath) if os.path.exists(fpath) else 0
status = '✅' if size >= 2000 else '❌ 过小，内容可能不完整'
print(f'[02] {status} {sid} → {fpath}（{size} 字节）')
"
```

### Step 3：生成索引文件

```bash
python -c "
import json, glob, os, datetime

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
title = ctx['meta']['game_title']
all_systems = ctx.get('systems', [])
done_files = {os.path.splitext(os.path.basename(f))[0]
              for f in glob.glob('design/systems/*.md') if 'README' not in f}

lines = [f'# 系统策划案索引\n\n**游戏**：{title}  **生成时间**：{datetime.date.today()}\n']
for pri in ('P0', 'P1', 'P2'):
    group = [s for s in all_systems if s.get('priority') == pri]
    if not group: continue
    lines.append(f'## {pri} 系统\n\n| 系统ID | 名称 | 复杂度 | 状态 | 文档 |\n|-------|------|-------|------|------|\n')
    for s in group:
        status = '✅' if s['id'] in done_files else '⏳'
        lines.append(f'| {s[\"id\"]} | {s[\"name\"]} | {s.get(\"complexity\",\"M\")} | {status} | [{s[\"id\"]}.md](./{s[\"id\"]}.md) |\n')
    lines.append('')

with open('design/systems/README.md', 'w', encoding='utf-8') as f:
    f.writelines(lines)
print('[02] ✅ 索引已生成')
"
```

### Step 4：更新状态 + 完成日志

```bash
python -c "
import json, datetime, os, glob

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
specs = [{'system_id': os.path.splitext(os.path.basename(f))[0],
          'spec_path': f'design/systems/{os.path.splitext(os.path.basename(f))[0]}.md',
          'status': 'draft'}
         for f in sorted(glob.glob('design/systems/*.md')) if 'README' not in f]
d['system_specs'] = specs
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '02',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

total = len([s for s in d.get('systems', []) if s.get('priority') in ('P0','P1')])
done = len(specs)
line = f'[Agent02] ✅ {datetime.datetime.now().isoformat()} | 输出: design/systems/（{done}个文件）| 摘要: {done}/{total}个系统策划案完成\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print(f'[02] 完成 {done}/{total}')
"
```

---

## 完成标准

- [ ] 每个 P0/P1 系统都有对应文件（> 2000 字节）
- [ ] 每个文件包含完整九章，§6.2 有信号名列表
- [ ] 所有数值参数有具体数字
- [ ] `design/systems/README.md` 含 complexity 列
- [ ] `reports/pipeline_log.md` 有 Agent02 开始行和完成行
