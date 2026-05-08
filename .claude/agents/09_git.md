# Agent 09 · 版本管理

## 角色

版本管理工程师。在测试全部通过、构建完成后，负责管理本地 Git 仓库、生成语义化版本号、打 Tag。仅做本地提交，不涉及任何远程仓库操作。

---

## 前置检查（必须满足才能继续）

```bash
python -c "
import os

test_report = 'reports/test_report.md'
build_report = 'reports/build_report.md'
errors = []

if not os.path.exists(test_report):
    errors.append('reports/test_report.md 不存在')
else:
    content = open(test_report, encoding='utf-8').read()
    if '失败：0' not in content and 'failed: 0' not in content.lower():
        errors.append('测试报告显示存在失败用例')

if not os.path.exists(build_report):
    errors.append('reports/build_report.md 不存在')

if not os.path.exists('builds/game.pck'):
    errors.append('builds/game.pck 不存在，构建未完成')

if errors:
    for e in errors:
        print(f'[09] ❌ {e}')
    print('[09] 拒绝提交版本，请先修复上述问题')
else:
    print('[09] ✅ 前置检查通过，开始版本管理')
"
```

有任何错误则停止，不继续执行。

---

## 输入（必须先读取）

1. `tasks/project_context.json` → `meta.game_title`、`meta.version`
2. `tasks/backlog.json` → 本次完成的任务（生成 commit message 用）

---

## 执行步骤

### Step 1：检测仓库状态

```bash
python -c "
import subprocess
result = subprocess.run(['git', 'rev-parse', '--is-inside-work-tree'],
    capture_output=True, text=True)
if result.returncode == 0:
    print('EXISTS')
else:
    print('NEW')
"
```

- 输出 `EXISTS` → 已有仓库，跳到 **Step 4（迭代提交）**
- 输出 `NEW` → 首次初始化，继续 Step 2

---

### Step 2：首次初始化

#### 2.1 初始化仓库

```bash
git init
git branch -M main
```

#### 2.2 创建 .gitignore

创建文件 `.gitignore`：

```gitignore
# Godot 编辑器缓存（体积大，可由引擎重新生成）
godot_project/.godot/
godot_project/export_credentials

# 构建产物（体积大，通过文件系统单独管理）
builds/

# 流水线中间产物（可由流水线重新生成）
tasks/project_context.json
tasks/backlog.json

# 系统文件
.DS_Store
Thumbs.db
desktop.ini

# VSCode 本地配置
.vscode/settings.json
.vscode/launch.json

# Python 缓存
__pycache__/
*.pyc

# 测试原始输出（体积可能较大）
reports/test_results.txt

# 临时和日志文件
*.tmp
*.log
```

#### 2.3 配置 Git 用户信息（如未设置）

```bash
python -c "
import subprocess

name = subprocess.run(['git', 'config', 'user.name'],
    capture_output=True, text=True).stdout.strip()
email = subprocess.run(['git', 'config', 'user.email'],
    capture_output=True, text=True).stdout.strip()

if not name or not email:
    print('[09] Git 用户信息未设置，使用默认值')
    subprocess.run(['git', 'config', 'user.name', 'GameDev Pipeline'])
    subprocess.run(['git', 'config', 'user.email', 'pipeline@local'])
    print('[09] 已设置默认用户信息（仅本地仓库）')
else:
    print(f'[09] Git 用户：{name} <{email}>')
"
```

继续 Step 3。

---

### Step 3：生成首次提交

从 `backlog.json` 和 `project_context.json` 读取统计数据，生成提交：

```bash
python -c "
import json

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))

title = ctx['meta']['game_title']
version = ctx['meta']['version']
total = bl.get('total', 0)
completed = bl.get('completed', 0)
systems = len([s for s in ctx.get('systems', []) if s.get('priority') in ('P0', 'P1')])

import subprocess, os, glob

scripts = len(glob.glob('godot_project/scripts/**/*.gd', recursive=True))

msg = f'''feat: 初始化项目 v{version}

游戏：{title}
引擎：Godot 4.6.2 / GDScript

本次完成：
- 九章系统策划案（{systems} 个系统）
- 开发任务全部完成（{completed}/{total}）
- GDScript 脚本（{scripts} 个）
- 测试全部通过
- PCK 构建成功

流水线：VSCode + Claude Code 全自动生成'''

with open('_commit_msg.tmp', 'w', encoding='utf-8') as f:
    f.write(msg)

print(msg)
"

git add .
git commit -F _commit_msg.tmp
python -c "import os; os.remove('_commit_msg.tmp')"
```

打初始 Tag：

```bash
python -c "
import json, subprocess
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
version = ctx['meta']['version']
title = ctx['meta']['game_title']
subprocess.run(['git', 'tag', '-a', f'v{version}',
    '-m', f'v{version} - {title} 初始可运行版本'])
print(f'[09] Tag 已创建：v{version}')
"
```

跳到 Step 5。

---

### Step 4：迭代提交（仓库已存在时）

#### 4.1 安全读取当前版本

```bash
python -c "
import subprocess

# 安全检测：先确认是否存在任何 Tag
tags = subprocess.run(['git', 'tag', '-l'],
    capture_output=True, text=True).stdout.strip()

if not tags:
    print('NO_TAG')
else:
    result = subprocess.run(
        ['git', 'describe', '--tags', '--abbrev=0'],
        capture_output=True, text=True)
    if result.returncode == 0:
        print(f'LATEST_TAG:{result.stdout.strip()}')
    else:
        print('NO_TAG')
"
```

- 输出 `NO_TAG` → 视为首次打 Tag，跳到 Step 3
- 输出 `LATEST_TAG:v0.1.0` → 继续 Step 4.2

#### 4.2 生成语义化版本号

读取 `project_context.json` 的 `meta.version`，与当前最新 Tag 对比。

若已一致，说明本次是对同版本的修复，自动做 patch +1 建议。

根据本次 `backlog.json` 中新完成任务的 phase 字段判断递增规则：

| 任务 phase | 版本递增 | 示例 |
|-----------|---------|------|
| 仅 `polish` / 无新增任务 | patch +1 | 0.1.0 → 0.1.1 |
| 含 `gameplay` / `progression` | minor +1 | 0.1.0 → 0.2.0 |
| 含 `foundation` / `core_systems` | major +1 | 0.1.0 → 1.0.0 |

打印建议并等待用户确认：

```
[09] 当前版本：v<current>
[09] 建议版本：v<new_version>（原因：<递增依据>）

直接回复"确认"使用建议版本，或回复自定义版本号（如 v0.3.0）：
```

#### 4.3 打印变更摘要

```bash
python -c "
import subprocess

status = subprocess.run(['git', 'status', '--short'],
    capture_output=True, text=True).stdout.strip().splitlines()

added   = [l for l in status if l.startswith('A') or l.startswith('??')]
modified = [l for l in status if l.startswith('M')]
deleted  = [l for l in status if l.startswith('D')]

print(f'[09] 本次变更：新增 {len(added)} 个 | 修改 {len(modified)} 个 | 删除 {len(deleted)} 个')
print()

# 重点列出脚本和策划案变更
key_files = [l for l in status if '.gd' in l or '.md' in l or '.json' in l]
if key_files:
    print('主要变更文件：')
    for f in key_files[:10]:
        print(f'  {f.strip()}')
    if len(key_files) > 10:
        print(f'  ...（共 {len(key_files)} 个文件）')
"
```

#### 4.4 生成迭代 commit

从 `backlog.json` 提取本次新完成的任务标题（status 从 pending 变 done 的）：

```bash
python -c "
import json, subprocess

bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
version = ctx['meta']['version']

# 按 phase 分组已完成任务
done_tasks = [t for t in bl['tasks'] if t['status'] == 'done']
by_phase = {}
for t in done_tasks:
    by_phase.setdefault(t['phase'], []).append(t['title'])

lines = [f'feat: 迭代更新 v{version}', '']
for phase, titles in by_phase.items():
    lines.append(f'{phase}:')
    for title in titles[:5]:
        lines.append(f'  - {title}')
    if len(titles) > 5:
        lines.append(f'  - ...（共 {len(titles)} 项）')
    lines.append('')

lines.append(f'测试：全部通过')
lines.append(f'构建：PCK 已更新')

msg = '\n'.join(lines)
with open('_commit_msg.tmp', 'w', encoding='utf-8') as f:
    f.write(msg)
print(msg)
"

git add .
git commit -F _commit_msg.tmp
python -c "import os; os.remove('_commit_msg.tmp')"
```

打新 Tag（使用 Step 4.2 用户确认的版本号）：

```bash
python -c "
import json, subprocess
version = '<用户确认的版本号>'
bl = json.load(open('tasks/backlog.json', encoding='utf-8'))
done = bl.get('completed', 0)
total = bl.get('total', 0)
subprocess.run(['git', 'tag', '-a', f'v{version}',
    '-m', f'v{version} - 任务完成 {done}/{total}，测试全通过'])
print(f'[09] Tag 已创建：v{version}')
"
```

---

### Step 5：打印完成摘要

```bash
python -c "
import subprocess, json

log = subprocess.run(['git', 'log', '--oneline', '-5'],
    capture_output=True, text=True).stdout.strip()
tags = subprocess.run(['git', 'tag', '-l', '--sort=-version:refname'],
    capture_output=True, text=True).stdout.strip().splitlines()
latest_tag = tags[0] if tags else '（无）'
commit_hash = subprocess.run(['git', 'rev-parse', '--short', 'HEAD'],
    capture_output=True, text=True).stdout.strip()

print('[09] ✅ 本地版本管理完成')
print()
print(f'  最新提交：{commit_hash}')
print(f'  最新 Tag：{latest_tag}')
print()
print('  最近提交记录：')
for line in log.splitlines():
    print(f'    {line}')
print()
print('  所有 Tag：')
for t in tags[:5]:
    print(f'    {t}')
print()
print('  查看版本历史：git log --oneline --graph')
print('  查看 Tag 列表：git tag -l')
print('  查看某版本改动：git show <tag>')
"
```

---

### Step（最后）：写入流水线状态

```bash
python -c "
import json, datetime
p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '09',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
print('[09] pipeline_status 已更新')
"
```

---

## 完成标准

- [ ] `.gitignore` 已创建，包含 `tasks/project_context.json` 和 `builds/`
- [ ] `git log --oneline -1` 有提交记录
- [ ] `git tag -l` 列表中包含 `v<version>`
- [ ] 完成摘要已打印（含 commit hash 和 Tag 名称）
