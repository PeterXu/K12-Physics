# Dataset Rules - Physics

本文档记录了物理数据集的处理规则和脚本。

## 1. 数据提取

从 `China-K12-STEM-10K-CoT-Reasoning` 中提取物理题目并转换为 Markdown 格式。

### 数据源

- `train.jsonl` - 训练集
- `test.jsonl` - 测试集
- `validation.jsonl` - 验证集

### 输出格式

每个题目一个 Markdown 文件，命名格式：`Physics_Q{序号:03d}_{哈希}.md`

### 提取脚本

```python
import json
import os
import hashlib

output_dir = "dt_k12_stem_physics"
os.makedirs(output_dir, exist_ok=True)

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

physics_count = 0
file_list = []

for data_file in data_files:
    with open(data_file, 'r', encoding='utf-8') as f:
        for line in f:
            if not line.strip():
                continue

            data = json.loads(line)

            # 筛选物理题目
            if data.get('subject') != 'Physics':
                continue

            physics_count += 1

            # 生成哈希ID
            question_id = data.get('question_id', '')
            hash_obj = hashlib.md5(question_id.encode())
            hash_hex = hash_obj.hexdigest()[:8]

            # 构造文件名
            filename = f"Physics_Q{physics_count:03d}_{hash_hex}.md"
            filepath = os.path.join(output_dir, filename)
            file_list.append((physics_count, question_id, filename, data))

            # 提取数据
            question_text = data.get('question_text', '').replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
            solution_steps = data.get('solution_steps', [])
            knowledge_points = data.get('knowledge_points', [])
            knowledge_points_en = data.get('knowledge_points_en', [])
            grade = data.get('grade', '')
            grade_level = data.get('grade_level', f'Grade {grade}')
            difficulty = data.get('difficulty', '')
            question_type = data.get('question_type', '')

            # 格式化解答步骤
            solution_text = ""
            for i, step in enumerate(solution_steps, 1):
                step_clean = step.replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
                solution_text += str(i) + ". " + step_clean + "\n"

            # 格式化知识点
            kp_text = "**中文**:\n"
            for kp in knowledge_points:
                kp_text += "  - " + kp + "\n"
            kp_text += "\n**English**:\n"
            for kp_en in knowledge_points_en:
                kp_text += "  - " + kp_en + "\n"

            # 写入 Markdown 文件
            header = "# Physics - 题目 " + str(physics_count) + "\n\n"
            meta = "**ID**: " + question_id + "\n"
            meta += "**年级**: " + grade_level + "\n"
            meta += "**难度**: " + difficulty + "\n"
            meta += "**题型**: " + question_type + "\n\n"

            q_section = "## 题目\n" + question_text + "\n\n"
            a_section = "## 解答\n\n" + solution_text + "\n"
            k_section = "## 知识点\n\n" + kp_text + "\n---\n"

            md_content = header + meta + q_section + a_section + k_section

            with open(filepath, 'w', encoding='utf-8') as out_f:
                out_f.write(md_content)

            if physics_count % 100 == 0:
                print(f"已提取 {physics_count} 道物理题...")

print(f"完成! 共提取 {physics_count} 道物理题目")
```

## 2. 图片下载

### 代理设置

图片托管服务器需要代理访问：
```bash
export HTTP_PROXY=http://127.0.0.1:1087
export HTTPS_PROXY=http://127.0.0.1:1087
```

### 下载脚本

```python
import json
import os
import hashlib
import urllib.request

output_dir = "dt_k12_stem_physics"
images_dir = os.path.join(output_dir, "images")
os.makedirs(images_dir, exist_ok=True)

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

physics_count = 0
image_success = 0
image_fail = 0

def download_image(url, local_path):
    """下载图片，需要添加请求头避免403错误"""
    global image_success, image_fail
    try:
        if os.path.exists(local_path):
            image_success += 1
            return True
        req = urllib.request.Request(url, headers={
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
            'Referer': 'https://huggingface.co/'
        })
        with urllib.request.urlopen(req, timeout=30) as response:
            with open(local_path, 'wb') as f:
                f.write(response.read())
        image_success += 1
        return True
    except Exception as e:
        image_fail += 1
        return False

for idx, qid, filename, data in file_list:
    question_images = data.get('question_images', [])
    for img_url in question_images:
        img_name = img_url.split('/')[-1]
        local_img_name = f"{idx:03d}_{img_name}"
        local_img_path = os.path.join(images_dir, local_img_name)
        download_image(img_url, local_img_path)

    if idx % 100 == 0:
        print(f"已处理 {idx} 道题... (已下载图片: {image_success}, 失败: {image_fail})")

print(f"\n完成! 共处理 {physics_count} 道物理题")
print(f"图片下载成功: {image_success} 张")
print(f"图片下载失败: {image_fail} 张")
```

### 添加图片引用到 Markdown

```python
import json
import os
import hashlib

output_dir = "dt_k12_stem_physics"
images_dir = os.path.join(output_dir, "images")

data_files = [
    "China-K12-STEM-10K-CoT-Reasoning/train.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/test.jsonl",
    "China-K12-STEM-10K-CoT-Reasoning/validation.jsonl"
]

physics_count = 0

for idx, qid, filename, data in file_list:
    filepath = os.path.join(output_dir, filename)
    if not os.path.exists(filepath):
        continue

    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()

    # 如果已经有图片引用，跳过
    if '![' in content:
        continue

    question_images = data.get('question_images', [])
    image_refs = []
    for img_url in question_images:
        img_name = img_url.split('/')[-1]
        local_img_name = f"{idx:03d}_{img_name}"
        local_img_path = os.path.join(images_dir, local_img_name)

        if os.path.exists(local_img_path):
            image_refs.append(f"images/{local_img_name}")

    if image_refs:
        question_text = data.get('question_text', '').replace('\\(', '$').replace('\\)', '$').replace('\\[', '$$').replace('\\]', '$$')
        new_question_section = "## 题目\n" + question_text + "\n\n"
        for img_ref in image_refs:
            new_question_section += f"![{img_ref}]({img_ref})\n"
        new_question_section += "\n"

        content = content.replace("## 题目\n" + question_text + "\n\n", new_question_section)

        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(content)

print(f"完成! 已更新文件")
```

## 3. 文件重命名

### 命名规则

从 `Physics_Q001_xxxxxxxx.md` 重命名为 `grade{年级}_{难度}_{题型}_{ID前7位}.md`

示例：`grade10_medium_hard_calculation_4a5cf47.md`

### 题型翻译规则

| 中文题型 | 英文翻译 |
|---------|---------|
| 选择题 | multiple_choice |
| 单选题 | single_choice |
| 多选题/双选题/多项选择题 | multiple_select |
| 填空题 | fill_in_the_blank |
| 计算题 | calculation |
| 解答题 | calculation |
| 实验题/实验探究题 | experiment |
| 作图题 | drawing |
| 证明题 | proof |
| 简答题 | short_answer |
| 练习 | practice |
| 达标练 | practice |
| 基础练 | basic_practice |
| 刷真题 | exam_review |
| 课时作业 | classwork |
| 课内填空 | fill_in_the_blank |
| 自我评价与作业 | self_assessment |
| 动手动脑学物理 | hands_on_physics |
| 掩体计划/技术问题 | special_topic |
| 根据要求作答 | short_answer |
| 未知 | unknown |
| 科学阅读 | science_reading |
| 综合题 | comprehensive |

### 重命名脚本

```python
import os
import re

def normalize_type(qtype):
    """将题型规范化为英文"""
    qtype = qtype.strip()

    # 已经是英文的标准类型
    english_types = {
        'Calculation': 'calculation',
        'Multiple-choice': 'multiple_choice',
        'Fill-in-the-blank': 'fill_in_the_blank',
        'Multiple-select': 'multiple_select',
        'Experiment': 'experiment',
        'Drawing': 'drawing',
        'Comprehensive': 'comprehensive',
    }
    if qtype in english_types:
        return english_types[qtype]

    # 包含证明或演算步骤的描述
    if '证明过程' in qtype or '演算步骤' in qtype or '推理过程' in qtype or '解答题' in qtype:
        return 'calculation'

    # 中文类型翻译
    translations = [
        ('单选题', 'single_choice'), ('多选题', 'multiple_select'),
        ('双选题', 'multiple_select'),
        ('多项选择题', 'multiple_select'), ('选择题', 'multiple_choice'),
        ('填空题', 'fill_in_the_blank'), ('双空题', 'double_blank'),
        ('实验题', 'experiment'), ('实验探究题', 'experiment'),
        ('计算题', 'calculation'), ('解答题', 'solution'),
        ('作图题', 'drawing'), ('证明题', 'proof'),
        ('简答题', 'short_answer'),
    ]
    for cn, en in translations:
        if cn in qtype:
            return en

    # 处理包含分数的描述
    if '分' in qtype and ('题' in qtype or '本大题' in qtype):
        return 'calculation'

    # 处理特殊描述
    if '有选错得0分' in qtype or '选对但不全' in qtype:
        return 'multiple_select'
    elif '达标练' in qtype:
        return 'practice'
    elif '基础练' in qtype:
        return 'basic_practice'
    elif '刷真题' in qtype:
        return 'exam_review'
    elif '课时作业' in qtype:
        return 'classwork'
    elif '掩体计划' in qtype or '技术问题' in qtype:
        return 'special_topic'
    elif '动手动脑学物理' in qtype:
        return 'hands_on_physics'
    elif '自我评价与作业' in qtype:
        return 'self_assessment'
    elif '根据要求作答' in qtype:
        return 'short_answer'
    elif '未知' in qtype:
        return 'unknown'
    elif '科学阅读' in qtype:
        return 'science_reading'

    # 默认清理
    result = re.sub(r'[^\w_]', '_', qtype.lower())[:30]
    return result if result else 'other'

def parse_and_rename(filepath):
    """解析并重命名单个文件"""
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()

    # 提取ID
    id_match = re.search(r'ID.*?:\s*([a-f0-9]+)', content)
    if not id_match:
        return None, None
    question_id = id_match.group(1)[:7]

    # 提取年级
    grade_match = re.search(r'年级.*?:\s*Grade (\d+)', content)
    if not grade_match:
        return None, None
    grade = f"grade{grade_match.group(1)}"

    # 提取难度
    difficulty_match = re.search(r'难度.*?:\s*(.+)', content)
    if not difficulty_match:
        return None, None
    difficulty = difficulty_match.group(1).strip().lower().replace('-', '_').replace(' ', '_')

    # 提取题型
    type_match = re.search(r'题型.*?:\s*(.+)', content)
    if not type_match:
        return None, None
    qtype_raw = type_match.group(1).strip()
    qtype = normalize_type(qtype_raw)

    new_name = f"{grade}_{difficulty}_{qtype}_{question_id}.md"
    return filepath, new_name

# 处理所有文件
md_files = sorted([f for f in os.listdir('.') if f.endswith('.md')])
renamed = 0
skipped = 0

for old_name in md_files:
    old_path, new_name = parse_and_rename(old_name)
    if not old_path or old_name == new_name or os.path.exists(new_name):
        skipped += 1
        continue
    try:
        os.rename(old_name, new_name)
        renamed += 1
    except Exception as e:
        print(f"错误: {old_name} - {e}")

print(f"完成! 重命名成功: {renamed} 个, 跳过: {skipped} 个")
```

## 4. 处理结果统计

| 项目 | 数量 |
|------|------|
| 物理题目总数 | 3005 |
| 下载图片 | 2723 张 |
| Markdown 文件 | 3005 个 |
| 包含图片的文件 | 2463 个 |

### 年级分布

| 年级 | 数量 |
|------|------|
| Grade 12 | 786 |
| Grade 11 | 688 |
| Grade 10 | 487 |
| Grade 9 | 377 |
| Grade 8 | 358 |
| Grade 7 | 309 |

### 难度分布

| 难度 | 数量 |
|------|------|
| Medium | 1565 |
| Medium-Hard | 769 |
| Easy | 336 |
| Hard | 290 |
| Medium-Easy | 45 |

### 题型分布

| 题型 | 数量 |
|------|------|
| multiple_choice | 608 |
| hard_calculation | 558 |
| experiment | 276 |
| calculation | 259 |
| fill_in_the_blank | 245 |
| hard_experiment | 238 |
| hard_multiple_choice | 224 |
| multiple_select | 159 |
| hard_multiple_select | 172 |
| hard_comprehensive | 77 |
| hard_fill_in_the_blank | 61 |
| drawing | 43 |
| comprehensive | 26 |
| short_answer | 11 |
| practice | 11 |
| science_reading | 8 |
| hard_science_reading | 7 |
| special_topic | 1 |
| hands_on_physics | 1 |
| self_assessment | 1 |
| unknown | 2 |
| classwork | 1 |
| exam_review | 1 |
| basic_practice | 1 |
