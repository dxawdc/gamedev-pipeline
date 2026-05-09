# Agent 09 · 版本管理

## 角色

版本管理工程师。管理本地 Git 仓库、生成语义化版本号、打 Tag，并检测大文件决定是否启用 Git LFS（P2）。仅做本地操作，不涉及远程仓库。

---

## 开始日志

```bash
python -c "
import datetime
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(f'[Agent09] 🚀 {datetime.datetime.now().isoformat()} | 开始执行\n')
"
```

## 前置检查

```bash
python -c "
import os, sys

errors = []
for f, desc in [('reports/test_report.md', '测试报告'),
                ('reports/build_report.md', '构建报告'),
                ('builds/game.pck', 'PCK 文件')]:
    if not os.path.exists(f):
        errors.append(f'{desc}（{f}）不存在')

if os.path.exists('reports/test_report.md'):
    c = open('reports/test_report.md', encoding='utf-8').read()
    if '失败：0' not in c and 'failed: 0' not in c.lower():
        errors.append('测试报告显示存在失败用例')

if errors:
    for e in errors: print(f'[09] ❌ {e}')
    sys.exit(1)
print('[09] ✅ 前置检查通过')
"
```

---

## 执行步骤

### Step 1：P2 Git LFS 大文件检测

```bash
python -c "
import os, glob

# 检查资源目录下是否有大文件（>1MB）
asset_files = glob.glob('godot_project/assets/**/*', recursive=True)
large_files = [(f, os.path.getsize(f)) for f in asset_files
               if os.path.isfile(f) and os.path.getsize(f) > 1024*1024]

if large_files:
    print(f'[09] ⚠️  发现 {len(large_files)} 个大文件（>1MB），建议启用 Git LFS：')
    for f, size in large_files[:5]:
        print(f'  {f}（{size//1024} KB）')
    print('LFS_NEEDED')
else:
    print('[09] ✅ 无大文件，无需 Git LFS')
    print('LFS_SKIP')
"
```

若输出 `LFS_NEEDED`，初始化 Git LFS：

```bash
python -c "
import subprocess, os

# 检查 git lfs 是否可用
result = subprocess.run(['git', 'lfs', 'version'], capture_output=True, text=True)
if result.returncode != 0:
    print('[09] ⚠️  git lfs 未安装，请安装后重新运行')
    print('  安装：https://git-lfs.com/')
else:
    subprocess.run(['git', 'lfs', 'install'])
    for pattern in ['*.png', '*.jpg', '*.jpeg', '*.ogg', '*.wav', '*.mp3']:
        subprocess.run(['git', 'lfs', 'track', pattern])
    print('[09] ✅ Git LFS 已启用，追踪：*.png *.jpg *.ogg *.wav *.mp3')
    print('  .gitattributes 已更新')
"
```

并在 `reports/build_guide.md` 末尾追加 LFS 说明：

```bash
python -c "
with open('reports/build_guide.md', 'a', encoding='utf-8') as f:
    f.write('\n## Git LFS\n\n本项目使用 Git LFS 管理大型资源文件。\n克隆时需先安装 git lfs，然后执行 git lfs pull 拉取资源。\n')
print('[09] build_guide.md 已追加 LFS 说明')
"
```

### Step 2：检测仓库状态

```bash
python -c "
import subprocess
r = subprocess.run(['git', 'rev-parse', '--is-inside-work-tree'], capture_output=True, text=True)
print('EXISTS' if r.returncode == 0 else 'NEW')
"
```

`EXISTS` → 跳到 Step 5（迭代提交）；`NEW` → 继续 Step 3。

---

### Step 3：首次初始化

```bash
git init && git branch -M main
```

创建 `.gitignore`：

```bash
python -c "
content = '''# Godot 编辑器缓存
godot_project/.godot/
godot_project/export_credentials

# 构建产物
builds/

# 流水线中间产物
tasks/project_context.json
tasks/backlog.json

# 系统文件
.DS_Store
Thumbs.db
desktop.ini

# VSCode
.vscode/settings.json
.vscode/launch.json

# Python
__pycache__/
*.pyc

# 测试原始输出
reports/test_results.txt

# 临时文件
*.tmp
*.log
_commit_msg.tmp
'''
with open('.gitignore', 'w', encoding='utf-8') as f:
    f.write(content)
print('[09] ✅ .gitignore 已创建')
"
```

配置 Git 用户：

```bash
python -c "
import subprocess
name  = subprocess.run(['git','config','user.name'],  capture_output=True, text=True).stdout.strip()
email = subprocess.run(['git','config','user.email'], capture_output=True, text=True).stdout.strip()
if not name:  subprocess.run(['git','config','user.name',  'GameDev Pipeline'])
if not email: subprocess.run(['git','config','user.email', 'pipeline@local'])
print(f'[09] Git 用户：{name or \"GameDev Pipeline\"} <{email or \"pipeline@local\"}>')
"
```

### Step 4：首次提交 + Tag

```bash
python -c "
import json, glob, datetime

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
bl  = json.load(open('tasks/backlog.json',         encoding='utf-8'))
m   = ctx['meta']
scripts = len(glob.glob('godot_project/scripts/**/*.gd', recursive=True))
tests   = len(glob.glob('godot_project/tests/**/test_*.gd', recursive=True))
systems = len([s for s in ctx.get('systems',[]) if s.get('priority') in ('P0','P1')])

msg = f'''feat: 初始化项目 v{m['version']}

游戏：{m['game_title']}
引擎：Godot 4.6.2 / GDScript

完成内容：
- 九章系统策划案（{systems} 个系统）
- 开发任务全部完成（{bl.get('completed',0)}/{bl.get('total',0)}）
- GDScript 脚本（{scripts} 个）
- 测试文件（{tests} 个），全部通过
- PCK 构建成功

时间：{datetime.datetime.now().isoformat()}
流水线：VSCode + Claude Code 全自动生成'''

with open('_commit_msg.tmp', 'w', encoding='utf-8') as f:
    f.write(msg)
print(msg)
"
```

```bash
git add . && git commit -F _commit_msg.tmp
```

**无论 commit 成功与否，立即清理临时文件**：

```bash
python -c "import os; os.path.exists('_commit_msg.tmp') and os.remove('_commit_msg.tmp')"
```

打 Tag：

```bash
python -c "
import json, subprocess
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
v, t = ctx['meta']['version'], ctx['meta']['game_title']
r = subprocess.run(['git','tag','-a',f'v{v}','-m',f'v{v} - {t} 初始可运行版本'],
                   capture_output=True, text=True)
print(f'[09] ✅ Tag v{v}' if r.returncode == 0 else f'[09] ⚠️  Tag 失败：{r.stderr}')
"
```

跳到 Step 6。

---

### Step 5：迭代提交

#### 5.1 自动计算版本号（全自动，不等待人工确认）

```bash
python -c "
import json, subprocess

ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
bl  = json.load(open('tasks/backlog.json', encoding='utf-8'))
current = ctx['meta']['version']

phases = {t['phase'] for t in bl['tasks'] if t['status'] == 'done'}
major, minor, patch = (int(x) for x in current.split('.'))

if 'foundation' in phases or 'core_systems' in phases:
    reason = 'foundation/core_systems 任务'
    if minor == 0 and patch == 0: major += 1; minor = patch = 0
    else: minor += 1; patch = 0
elif 'gameplay' in phases or 'progression' in phases:
    reason = 'gameplay/progression 任务'; minor += 1; patch = 0
else:
    reason = 'polish 任务'; patch += 1

new_version = f'{major}.{minor}.{patch}'
print(f'[09] 版本：{current} → {new_version}（{reason}）')

# 同步更新 meta.version
ctx['meta']['version'] = new_version
json.dump(ctx, open('tasks/project_context.json','w', encoding='utf-8'), ensure_ascii=False, indent=2)
print(f'NEW_VERSION:{new_version}')
"
```

#### 5.2 生成迭代 commit

```bash
python -c "
import json, datetime

bl  = json.load(open('tasks/backlog.json', encoding='utf-8'))
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
new_v = ctx['meta']['version']

by_phase = {}
for t in bl['tasks']:
    if t['status'] == 'done':
        by_phase.setdefault(t['phase'], []).append(t['title'])

lines = [f'feat: 迭代更新 v{new_v}', '']
for phase, titles in sorted(by_phase.items()):
    lines.append(f'{phase}:')
    for title in titles[:5]: lines.append(f'  - {title}')
    if len(titles) > 5: lines.append(f'  - ...（共 {len(titles)} 项）')
    lines.append('')
lines += [f'测试：全部通过', f'构建：PCK 已更新', f'时间：{datetime.datetime.now().isoformat()}']

with open('_commit_msg.tmp', 'w', encoding='utf-8') as f:
    f.write('\n'.join(lines))
print('\n'.join(lines))
"
```

```bash
git add . && git commit -F _commit_msg.tmp
```

**立即清理**：

```bash
python -c "import os; os.path.exists('_commit_msg.tmp') and os.remove('_commit_msg.tmp')"
```

打 Tag：

```bash
python -c "
import json, subprocess
ctx = json.load(open('tasks/project_context.json', encoding='utf-8'))
bl  = json.load(open('tasks/backlog.json', encoding='utf-8'))
v = ctx['meta']['version']
done, total = bl.get('completed',0), bl.get('total',0)
r = subprocess.run(['git','tag','-a',f'v{v}','-m',f'v{v} - 任务{done}/{total}，测试全通过'],
                   capture_output=True, text=True)
print(f'[09] ✅ Tag v{v}' if r.returncode == 0 else f'[09] ⚠️  {r.stderr}')
"
```

---

### Step 6：打印完成摘要

```bash
python -c "
import subprocess
log  = subprocess.run(['git','log','--oneline','-5'], capture_output=True, text=True).stdout.strip()
tags = subprocess.run(['git','tag','-l','--sort=-version:refname'], capture_output=True, text=True).stdout.strip().splitlines()
h    = subprocess.run(['git','rev-parse','--short','HEAD'], capture_output=True, text=True).stdout.strip()
print(f'[09] ✅ 版本管理完成')
print(f'  提交：{h}  最新Tag：{tags[0] if tags else \"无\"}')
print('  最近提交：')
for l in log.splitlines(): print(f'    {l}')
print('  常用：git log --oneline --graph | git tag -l | git show <tag>')
"
```

### Step 7：写入状态 + 完成日志

```bash
python -c "
import json, datetime, subprocess

p = 'tasks/project_context.json'
d = json.load(open(p, encoding='utf-8'))
d.setdefault('pipeline_status', {}).update({
    'last_completed_agent': '09',
    'last_completed_at': datetime.datetime.now().isoformat()
})
json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

tag = subprocess.run(['git','describe','--tags','--abbrev=0'], capture_output=True, text=True).stdout.strip()
line = f'[Agent09] ✅ {datetime.datetime.now().isoformat()} | 输出: Git commit + Tag {tag} | 摘要: 本地版本管理完成\n'
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(line)
print('[09] 完成')
"
```

---

## 完成标准

- [ ] `.gitignore` 含 `_commit_msg.tmp`、`builds/`
- [ ] `_commit_msg.tmp` 不存在（已清理，即使 commit 失败也清理）
- [ ] `git log --oneline -1` 有提交记录
- [ ] `git tag -l` 包含 `v<version>`
- [ ] 若有大文件：`.gitattributes` 已创建，`build_guide.md` 已追加 LFS 说明
- [ ] `reports/pipeline_log.md` 有 Agent09 开始行和完成行
