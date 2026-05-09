# Agent 00 · Word 转 Markdown

## 角色

文档处理专家。将 `design/` 目录中的 Word 文件转换为 `design/gdd.md`，供后续 Agent 读取。

---

## 前置检查

```bash
python -c "
import os, sys
os.makedirs('design', exist_ok=True)
os.makedirs('tasks', exist_ok=True)
os.makedirs('reports', exist_ok=True)
print('[00] 目录结构已就绪')
"
```

---

## 执行步骤

### Step 1：扫描 design/ 目录

```bash
python -c "
import os, glob
files = glob.glob('design/*.docx') + glob.glob('design/*.doc')
md_exists = os.path.exists('design/gdd.md')
if files:
    print('DOCX:' + files[0])
elif md_exists:
    size = os.path.getsize('design/gdd.md')
    print(f'MD_EXISTS:{size}')
else:
    print('EMPTY')
"
```

- 输出 `DOCX:<路径>` → 继续 Step 2
- 输出 `MD_EXISTS:<size>` 且 size > 500 → 打印 `[00] ℹ️ gdd.md 已存在且有效，跳过转换` 并跳到 Step 5
- 输出 `EMPTY` → 打印 `[00] ❌ 请将 .docx 文件放入 design/ 目录` 并停止

### Step 2：安装转换工具

```bash
pip install mammoth --quiet --break-system-packages 2>/dev/null || pip install mammoth --quiet
```

### Step 3：执行转换

将 `<filename>` 替换为实际文件名后执行：

```bash
python -c "
import mammoth, re, sys

filepath = 'design/<filename>'
try:
    with open(filepath, 'rb') as f:
        result = mammoth.convert_to_markdown(f)
    md = re.sub(r'\n{3,}', '\n\n', result.value).strip()
    if len(md) < 100:
        raise ValueError('转换结果过短，可能乱码')
    with open('design/gdd.md', 'w', encoding='utf-8') as f:
        f.write(md)
    print(f'[00] ✅ 主方案转换成功：{len(md.splitlines())} 行，{len(md)} 字符')
    if result.messages:
        for msg in result.messages[:3]:
            print(f'  提示：{msg}')
except Exception as e:
    print(f'[00] ⚠️ 主方案失败（{e}），切换备用方案...')
    sys.exit(1)
"
```

如果以上命令退出码为 1，立即执行备用方案：

```bash
python -c "
import mammoth
with open('design/<filename>', 'rb') as f:
    result = mammoth.extract_raw_text(f)
text = result.value.strip()
with open('design/gdd.md', 'w', encoding='utf-8') as f:
    f.write(text)
print(f'[00] ✅ 备用方案转换成功：{len(text.splitlines())} 行')
"
```

### Step 4：验证输出

```bash
python -c "
import sys
with open('design/gdd.md', encoding='utf-8') as f:
    content = f.read()
    lines = content.splitlines()

# 检查基本质量
has_chinese = any('\u4e00' <= c <= '\u9fff' for c in content)
is_long_enough = len(content) > 500
has_content = len(lines) > 10

print(f'[00] 验证结果：{len(lines)} 行，{len(content)} 字符')
print(f'  包含中文：{has_chinese}')
print(f'  长度充足：{is_long_enough}')
print(f'  前5行预览：')
for line in lines[:5]:
    print(f'  {line[:80]}')

if not (has_chinese and is_long_enough):
    print('[00] ❌ 内容质量不合格，请检查源文件')
    sys.exit(1)
print('[00] ✅ 验证通过')
"
```

### Step 5：写入流水线状态 + 摘要日志

```bash
python -c "
import json, datetime, os

# 更新 pipeline_status
p = 'tasks/project_context.json'
if os.path.exists(p):
    d = json.load(open(p, encoding='utf-8'))
    d.setdefault('pipeline_status', {}).update({
        'last_completed_agent': '00',
        'last_completed_at': datetime.datetime.now().isoformat()
    })
    json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)

# 追加摘要日志
import glob
lines_count = len(open('design/gdd.md', encoding='utf-8').readlines())
log_line = f'[Agent00] ✅ {datetime.datetime.now().isoformat()} | 输出: design/gdd.md | 摘要: Word转换完成，{lines_count}行\n'
os.makedirs('reports', exist_ok=True)
with open('reports/pipeline_log.md', 'a', encoding='utf-8') as f:
    f.write(log_line)
print('[00] pipeline_status 和摘要日志已更新')
"
```

---

## 完成标准

- [ ] `design/gdd.md` 存在且大于 500 字符
- [ ] 内容包含中文，无明显乱码
- [ ] `reports/pipeline_log.md` 已追加 Agent00 摘要行
