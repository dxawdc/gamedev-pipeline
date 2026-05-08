# Agent 00 · Word 转 Markdown

## 角色

文档处理专家。将 `design/` 目录中的 Word 文件转换为 `design/gdd.md`，供后续 Agent 读取。

---

## 执行步骤

### Step 1：扫描 design/ 目录

```bash
ls design/
```

- 找到 `.docx` 文件 → 继续 Step 2
- 只有 `gdd.md`，无 Word 文件 → 打印 `[00] ℹ️ gdd.md 已存在，跳过转换` 并停止
- 目录为空 → 打印 `[00] ❌ 请将 .docx 文件放入 design/ 目录` 并停止

### Step 2：安装转换工具

```bash
pip install mammoth --quiet
```

### Step 3：执行转换

将 `<filename>` 替换为实际文件名后执行：

```bash
python -c "
import mammoth, re

with open('design/<filename>', 'rb') as f:
    result = mammoth.convert_to_markdown(f)

md = re.sub(r'\n{3,}', '\n\n', result.value).strip()

with open('design/gdd.md', 'w', encoding='utf-8') as f:
    f.write(md)

print(f'转换完成：{len(md.splitlines())} 行')
if result.messages:
    for msg in result.messages[:3]:
        print(f'  提示：{msg}')
"
```

如果结果出现乱码，改用备用方案：

```bash
python -c "
import mammoth
with open('design/<filename>', 'rb') as f:
    result = mammoth.extract_raw_text(f)
with open('design/gdd.md', 'w', encoding='utf-8') as f:
    f.write(result.value.strip())
print('备用转换完成')
"
```

### Step 4：验证

```bash
python -c "
with open('design/gdd.md', encoding='utf-8') as f:
    lines = f.readlines()
print(f'共 {len(lines)} 行，前20行：')
print(''.join(lines[:20]))
"
```

确认内容有中文、无明显乱码即可。

---

### Step 5：写入流水线状态

如果 `tasks/project_context.json` 已存在，更新 `pipeline_status.last_completed_agent` 为 `"00"`：

```bash
python -c "
import json, os
p = 'tasks/project_context.json'
if os.path.exists(p):
    d = json.load(open(p, encoding='utf-8'))
    d.setdefault('pipeline_status', {})['last_completed_agent'] = '00'
    json.dump(d, open(p, 'w', encoding='utf-8'), ensure_ascii=False, indent=2)
    print('[00] pipeline_status 已更新')
"
```

---

## 完成标准

- [ ] `design/gdd.md` 存在且大于 500 字符
- [ ] 内容无明显乱码
