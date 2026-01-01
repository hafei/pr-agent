# PR-Agent Diff 扩展算法详解

## 目录
1. [算法概述](#1-算法概述)
2. [Git Diff 基础知识](#2-git-diff-基础知识)
3. [算法设计目标](#3-算法设计目标)
4. [核心函数分析](#4-核心函数分析)
5. [算法流程详解](#5-算法流程详解)
6. [动态上下文机制](#6-动态上下文机制)
7. [边缘情况处理](#7-边缘情况处理)
8. [配置选项](#8-配置选项)
9. [性能考虑](#9-性能考虑)
10. [示例演示](#10-示例演示)

---

## 1. 算法概述

### 1.1 问题背景

**原始 Git Diff 的局限性：**
- Git diff 默认只显示 3 行上下文
- 代码片段可能缺少重要的上下文信息
- AI 模型难以理解没有上下文的代码片段

**示例 - 原始 diff：**
```diff
@@ -42,3 +42,4 @@
     def process(self):
-        return old_result()
+        return new_result()
         self.log.info("Done")
```

**问题：** AI 不知道这个函数属于哪个类、接收什么参数、在什么上下文中被调用。

### 1.2 解决方案

**Diff 扩展算法的核心思想：**
1. 在每个 hunk（代码块）前后添加额外的上下文行
2. 智能识别代码边界（函数/类定义）
3. 处理文件变更（old vs new）带来的上下文差异
4. 更新 hunk header 的行号以反映扩展后的范围

**扩展后的 diff 示例：**
```diff
@@ -38,7 +38,8 @@ class DataProcessor:
     def __init__(self, config):
         self.config = config
         self.log = Logger()
+    def process(self):          <-- 动态上下文：找到函数定义
-        return old_result()
+        return new_result()
         self.log.info("Done")
+        return self              <-- 添加后 1 行
```

---

## 2. Git Diff 基础知识

### 2.1 Diff 格式

**标准 Unified Diff 格式：**
```diff
diff --git a/src/example.py b/src/example.py
index abc123..def456 100644
--- a/src/example.py (old)
+++ b/src/example.py (new)
@@ -old_start,old_lines +new_start,new_lines @@ section_header
 context_line1
 context_line2
-deleted_line
 context_line3
+added_line
+another_added_line
 context_line4
```

**Hunk Header 解析：**
- `@@ -10,5 +10,6 @@`:
  - `-10,5`: old file 从第 10 行开始，共 5 行
  - `+10,6`: new file 从第 10 行开始，共 6 行
  - `@@ section_header`: 可选的函数/类声明

### 2.2 Diff 符号含义

| 符号 | 含义 |
|------|------|
| ` ` (空格) | 上下文行（未改变的行） |
| `-` | 删除的行 |
| `+` | 添加的行 |
| `@@` | Hunk header 标记 |

---

## 3. 算法设计目标

### 3.1 核心目标

1. **提高上下文完整性**
   - 在 hunk 前后添加固定数量的上下文行
   - 可配置：`patch_extra_lines_before`, `patch_extra_lines_after`

2. **智能代码边界识别**
   - 动态上下文：寻找最近的函数/类定义
   - 避免截断逻辑代码块

3. **处理文件变更**
   - 比较 old 和 new file 的上下文行
   - 找到第一个匹配点（mini-match）

4. **维护 diff 正确性**
   - 验证 hunk 是否有效
   - 更新 hunk header 的行号

5. **性能优化**
   - 跳过不需要扩展的文件（配置）
   - 处理边界情况（文件开头/结尾）

### 3.2 约束条件

- **不能超出文件边界**
  ```python
  if extended_start1 - 1 + extended_size1 > len_original_lines:
      # 调整扩展范围
  ```

- **必须保持 diff 可应用性**
  - 验证 hunk 行与文件内容匹配
  - 处理编码差异

- **向后兼容**
  - 如果扩展失败，返回原始 patch

---

## 4. 核心函数分析

### 4.1 extend_patch() - 主入口函数

**位置：** `pr_agent/algo/git_patch_processing.py:11-31`

**函数签名：**
```python
def extend_patch(
    original_file_str,      # 原始文件内容
    patch_str,              # Git diff 字符串
    patch_extra_lines_before=0,  # hunk 前添加的行数
    patch_extra_lines_after=0,   # hunk 后添加的行数
    filename: str = "",     # 文件名（用于跳过检查）
    new_file_str = ""       # 新文件内容（用于处理文件变更）
) -> str
```

**执行流程：**
```python
1. 前置检查
   - patch_str 是否为空？
   - extra_lines 是否都为 0？
   - original_file_str 是否为空？
   - 如果任一条件为真，直接返回 patch_str

2. 解码处理
   - 如果是 bytes，解码为字符串
   - 尝试多种编码：utf-8, iso-8859-1, latin-1, ascii, utf-16

3. 跳过检查
   - 根据配置检查文件扩展名
   - 例如：.pb, .grpc 文件可能不需要扩展

4. 执行扩展
   - 调用 process_patch_lines()
   - 捕获异常，失败时返回原始 patch

5. 返回结果
   - 成功：返回扩展后的 patch
   - 失败：返回原始 patch（降级处理）
```

**代码示例：**
```python
def extend_patch(original_file_str, patch_str, 
             patch_extra_lines_before=0,
             patch_extra_lines_after=0, 
             filename: str = "", 
             new_file_str="") -> str:
    # 前置检查
    if not patch_str or \
       (patch_extra_lines_before == 0 and patch_extra_lines_after == 0) or \
       not original_file_str:
        return patch_str

    # 解码
    original_file_str = decode_if_bytes(original_file_str)
    new_file_str = decode_if_bytes(new_file_str)
    
    # 跳过检查
    if should_skip_patch(filename):
        return patch_str

    # 执行扩展
    try:
        extended_patch_str = process_patch_lines(
            patch_str, original_file_str,
            patch_extra_lines_before, 
            patch_extra_lines_after, 
            new_file_str
        )
    except Exception as e:
        get_logger().warning(f"Failed to extend patch: {e}")
        return patch_str  # 降级：返回原始 patch

    return extended_patch_str
```

---

### 4.2 process_patch_lines() - 核心算法

**位置：** `pr_agent/algo/git_patch_processing.py:56-185`

**函数签名：**
```python
def process_patch_lines(
    patch_str,                 # Git diff 字符串
    original_file_str,         # 原始文件内容
    patch_extra_lines_before,   # hunk 前添加的行数
    patch_extra_lines_after,    # hunk 后添加的行数
    new_file_str = ""         # 新文件内容
) -> str
```

**核心数据结构：**
```python
file_original_lines = original_file_str.splitlines()  # 原始文件行列表
file_new_lines = new_file_str.splitlines()          # 新文件行列表
patch_lines = patch_str.splitlines()                # patch 行列表
extended_patch_lines = []                          # 扩展后的 patch 行列表

# Hunk header 正则
RE_HUNK_HEADER = re.compile(
    r"^@@ -(\d+)(?:,(\d+))? \+(\d+)(?:,(\d+))? @@[ ]?(.*)"
)
# 捕获组：
# 1: old_start (必需)
# 2: old_lines (可选)
# 3: new_start (必需)
# 4: new_lines (可选)
# 5: section_header (可选)
```

**状态变量：**
```python
is_valid_hunk = True       # 当前 hunk 是否有效
start1, size1 = -1, -1   # old file: 起始行, 行数
start2, size2 = -1, -1   # new file: 起始行, 行数
section_header = ""       # 函数/类声明
```

---

## 5. 算法流程详解

### 5.1 完整流程图

```
开始
  ↓
解析 patch 行列表
  ↓
遍历每一行
  ↓
┌─────────────────────────────────┐
│ 当前行是否是 hunk header?    │
└─────────────────────────────────┘
  ↓ 是                    ↓ 否
┌────────────────┐      ┌──────────────────┐
│ 结束上一个 hunk │      │ 添加到输出      │
│ 添加后 1 行     │      └──────────────────┘
└────────────────┘
  ↓
┌─────────────────────────────────┐
│ 解析 hunk header           │
│ 提取：start1, size1,      │
│       start2, size2,        │
│       section_header       │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 验证 hunk 是否有效          │
│ - 检查行是否匹配文件内容    │
│ - 处理编码差异              │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 需要 扩展？                 │
│ (extra_lines > 0)           │
└─────────────────────────────────┘
  ↓ 是                    ↓ 否
┌─────────────────────────────────┐
│ 计算扩展范围               │
│ _calc_context_limits()       │
│                         │
│ 1. 向前扩展：              │
│    max(1, start - before)  │
│                         │
│ 2. 向后扩展：              │
│    size + before + after    │
│                         │
│ 3. 检查文件边界          │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 启用动态上下文？             │
│ (allow_dynamic_context)      │
└─────────────────────────────────┘
  ↓ 是                    ↓ 否
┌─────────────────────────────────┐
│ 动态上下文逻辑             │
│                         │
│ 1. 向前 10 行寻找         │
│    section_header           │
│                         │
│ 2. 如果找到：              │
│    - 调整范围到 header    │
│    - 验证 old/new 文件   │
│      这些行相同           │
│                         │
│ 3. 如果未找到：          │
│    - 回退到固定扩展       │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 处理文件变更（old vs new）  │
│                         │
│ 1. 比较 extra lines        │
│    original == new?        │
│                         │
│ 2. 如果不同：             │
│    - mini-match 算法       │
│    - 找第一个匹配点       │
│    - 调整扩展范围         │
│                         │
│ 3. 如果无匹配：           │
│    - 放弃扩展             │
│    - 使用原始 hunk        │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 移除重复的 section_header   │
│ (如果在 extra lines 中)      │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 添加新的 hunk header       │
│ 更新行号：                │
│ @@ -extended_start1,      │
│     extended_size1         │
│    +extended_start2,      │
│     extended_size2         │
│   @@ section_header        │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 添加 extra lines (前)      │
│ (prefix with ' ')           │
└─────────────────────────────────┘
  ↓
┌─────────────────────────────────┐
│ 继续处理 hunk 内容行       │
└─────────────────────────────────┘
  ↓
遍历完成？
  ↓ 否 ────→ 返回循环开始
  ↓ 是
┌─────────────────────────────────┐
│ 处理最后一个 hunk          │
│ 添加后 1 行               │
└─────────────────────────────────┘
  ↓
拼接扩展后的行
  ↓
返回扩展后的 patch 字符串
  ↓
结束
```

### 5.2 关键步骤详解

#### 步骤 1：处理上一个 hunk 的后置扩展

**位置：** `pr_agent/algo/git_patch_processing.py:77-79`

```python
if is_valid_hunk and (start1 != -1 and patch_extra_lines_after > 0):
    # 添加 hunk 后的 extra lines
    delta_lines_original = [f' {line}' for line in 
        file_original_lines[start1 + size1 - 1:start1 + size1 - 1 + patch_extra_lines_after]]
    extended_patch_lines.extend(delta_lines_original)
```

**说明：**
- 当遇到新的 hunk header 时，先处理上一个 hunk 的后置扩展
- `start1 + size1 - 1`: hunk 最后一行的索引
- 添加 `patch_extra_lines_after` 行的上下文
- 每行前添加空格 `' '`（表示是上下文行）

---

#### 步骤 2：解析 hunk header

**位置：** `pr_agent/algo/git_patch_processing.py:81`

```python
section_header, size1, size2, start1, start2 = extract_hunk_headers(match)
```

**extract_hunk_headers() 函数：**
```python
def extract_hunk_headers(match):
    res = list(match.groups())
    # 处理 None 值
    for i in range(len(res)):
        if res[i] is None:
            res[i] = 0
    
    # 解析数值
    try:
        start1, size1, size2, start1, start2 = map(int, res[:4])
    except:  # 特殊情况：'@@ -0,0 +1 @@'
        start1, size1, size2 = map(int, res[:3])
        start2 = 0
    
    section_header = res[4]
    return section_header, size1, size2, start1, start2
```

**示例：**
```diff
@@ -10,5 +10,6 @@ def process_data():
```
- `start1 = 10`: old file 从第 10 行开始
- `size1 = 5`: old file 共 5 行
- `start2 = 10`: new file 从第 10 行开始
- `size2 = 6`: new file 共 6 行
- `section_header = "def process_data():"`: 函数声明

---

#### 步骤 3：验证 hunk 有效性

**位置：** `pr_agent/algo/git_patch_processing.py:83`
**函数：** `check_if_hunk_lines_matches_to_file()`

```python
def check_if_hunk_lines_matches_to_file(i, original_lines, patch_lines, start1):
    """
    检查 hunk 行是否与原始文件内容匹配
    
    原因：我们遇到过 hunk header 行与原始文件内容不匹配的情况，
    这会导致扩展 hunk 时产生无效的 patch
    """
    is_valid_hunk = True
    try:
        # 检查 hunk 后的第一行（如果是上下文行）
        if i + 1 < len(patch_lines) and patch_lines[i + 1][0] == ' ':
            if patch_lines[i + 1].strip() != original_lines[start1 - 1].strip():
                # 检查是否需要不同的编码
                original_line = original_lines[start1 - 1].strip()
                for encoding in ['iso-8859-1', 'latin-1', 'ascii', 'utf-16']:
                    try:
                        if original_line.encode(encoding).decode().strip() == patch_lines[i + 1].strip():
                            get_logger().info(
                                f"检测到不同编码，hunk header 行 {start1}，"
                                f"需要的编码: {encoding}"
                            )
                            return False  # 仍然避免扩展 hunk
                    except:
                        pass
                
                is_valid_hunk = False
                get_logger().info(
                    f"PR 中的无效 hunk，hunk header 行 {start1} "
                    f"与原始文件内容不匹配"
                )
    except:
        pass
    
    return is_valid_hunk
```

**验证逻辑：**
1. 获取 hunk 后的第一行
2. 如果是上下文行（以空格开头）
3. 与原始文件对应行比较
4. 如果不匹配，尝试不同的编码
5. 如果仍然不匹配，标记为无效（不扩展此 hunk）

---

#### 步骤 4：计算扩展范围

**位置：** `pr_agent/algo/git_patch_processing.py:86-96`

```python
def _calc_context_limits(patch_lines_before):
    # 1. 向前扩展的起始行
    extended_start1 = max(1, start1 - patch_lines_before)
    
    # 2. 向前扩展后的总行数
    extended_size1 = size1 + (start1 - extended_start1) + patch_extra_lines_after
    
    # 3. new file 同理
    extended_start2 = max(1, start2 - patch_lines_before)
    extended_size2 = size2 + (start2 - extended_start2) + patch_extra_lines_after
    
    # 4. 检查文件边界
    if extended_start1 - 1 + extended_size1 > len_original_lines:
        # 不能超出原始文件
        delta_cap = extended_start1 - 1 + extended_size1 - len_original_lines
        extended_size1 = max(extended_size1 - delta_cap, size1)
        extended_size2 = max(extended_size2 - delta_cap, size2)
    
    return extended_start1, extended_size1, extended_start2, extended_size2
```

**数学公式：**
```
扩展后的起始行：
  extended_start = max(1, hunk_start - extra_lines_before)

扩展后的总行数：
  extended_size = hunk_size + (hunk_start - extended_start) + extra_lines_after

边界检查：
  if extended_start - 1 + extended_size > file_length:
      调整 extended_size 到文件边界
```

**示例：**
```
原始 hunk:
  @@ -10,5 +10,6 @@
  start1 = 10, size1 = 5

配置:
  patch_extra_lines_before = 3
  patch_extra_lines_after = 1

计算:
  extended_start1 = max(1, 10 - 3) = 7
  extended_size1 = 5 + (10 - 7) + 1 = 9

结果:
  @@ -7,9 +7,10 @@
  (向前扩展 3 行，向后扩展 1 行)
```

---

## 6. 动态上下文机制

### 6.1 什么是动态上下文？

**问题：** 固定数量的上下文可能不够
- 可能在函数中间截断
- 看不到完整的逻辑块
- AI 难以理解代码结构

**解决方案：** 智能识别代码边界
- 寻找最近的函数/类定义
- 向前调整到定义行
- 提供完整的代码上下文

**配置：**
```toml
[config]
allow_dynamic_context = true
max_extra_lines_before_dynamic_context = 10
```

### 6.2 动态上下文算法

**位置：** `pr_agent/algo/git_patch_processing.py:98-123`

**步骤 1：向前扫描寻找 section header**

```python
if allow_dynamic_context and file_new_lines:
    # 1. 先计算动态范围（最多 10 行）
    extended_start1, extended_size1, extended_start2, extended_size2 = \
        _calc_context_limits(patch_extra_lines_before_dynamic)
    
    # 2. 提取这些行的内容
    lines_before_original = file_original_lines[extended_start1 - 1:start1 - 1]
    lines_before_new = file_new_lines[extended_start2 - 1:start2 - 1]
    
    # 3. 寻找 section_header
    found_header = False
    for i, line in enumerate(lines_before_original):
        if section_header in line:
            # 找到！调整范围
            extended_start1, extended_start2 = \
                extended_start1 + i, extended_start2 + i
            extended_size1, extended_size2 = \
                extended_size1 - i, extended_size2 - i
            
            # 4. 验证 old/new 文件这些行相同
            lines_before_original_dynamic = lines_before_original[i:]
            lines_before_new_dynamic = lines_before_new[i:]
            
            if lines_before_original_dynamic == lines_before_new_dynamic:
                found_header = True
                section_header = ''  # 已找到，移除
            
            break
```

**示例：**

```
文件内容（行号）：
  1: class DataProcessor:
  2:     """数据处理类"""
  3:     
  4:     def __init__(self, config):
  5:         self.config = config
  6:         self.log = Logger()
  7:     
  8:     def process_data(self, data):  <-- hunk 从这里开始
  9:         result = self.transform(data)
 10:         self.log.info("Done")
 11:         return result
 12:
 13: # other code...

原始 hunk:
  @@ -8,3 +8,4 @@ def process_data(self, data):
  -        return old_result()
  +        return new_result()
           self.log.info("Done")

动态上下文（向前 10 行）：
  扫描到第 8 行：
    "class DataProcessor:" 不在第 8 行前
    "def __init__(self, config):" 不在
    "def process_data(self, data):" 在 section_header 中！
  
  结果：调整到函数定义行（第 8 行）
  
扩展后的 hunk:
  @@ -8,4 +8,5 @@ def process_data(self, data):
      def process_data(self, data):  <-- 添加函数定义
 -         return old_result()
 +         return new_result()
          self.log.info("Done")
```

**步骤 2：如果未找到，回退到固定扩展**

```python
if not found_header:
    # 未找到 section header，使用固定扩展
    extended_start1, extended_size1, extended_start2, extended_size2 = \
        _calc_context_limits(patch_extra_lines_before)
```

**步骤 3：验证 old/new 文件一致性**

```python
# 只有当 old 和 new 文件的这些行完全相同时才应用动态上下文
if lines_before_original_dynamic_context == lines_before_new_dynamic_context:
    found_header = True
    section_header = ''
```

**原因：** 如果这些行在 old 和 new 文件中不同，说明它们也被修改了，不应该作为上下文。

---

### 6.3 处理文件变更（Old vs New）

**问题：** 文件在 PR 中被修改了，old 和 new 版本的对应行可能不同

**位置：** `pr_agent/algo/git_patch_processing.py:128-153`

**步骤 1：提取 extra lines**

```python
# 提取 hunk 前的 extra lines
delta_lines_original = [f' {line}' for line in 
    file_original_lines[extended_start1 - 1:start1 - 1]]

if file_new_lines:
    delta_lines_new = [f' {line}' for line in 
        file_new_lines[extended_start2 - 1:start2 - 1]]
```

**步骤 2：比较 old 和 new**

```python
if delta_lines_original != delta_lines_new:
    # 不同！需要处理
    found_mini_match = False
    
    # Mini-match 算法：找第一个匹配点
    for i in range(len(delta_lines_original)):
        if delta_lines_original[i:] == delta_lines_new[i:]:
            # 找到匹配点！
            delta_lines_original = delta_lines_original[i:]
            delta_lines_new = delta_lines_new[i:]
            
            # 调整扩展范围
            extended_start1 += i
            extended_size1 -= i
            extended_start2 += i
            extended_size2 -= i
            
            found_mini_match = True
            break
    
    if not found_mini_match:
        # 完全不匹配，放弃扩展
        extended_start1 = start1
        extended_size1 = size1
        extended_start2 = start2
        extended_size2 = size2
        delta_lines_original = []
```

**示例：**

```
Old file (修改前):
  1: class DataProcessor:
  2:     def process(self):
  3:         self.log.debug("Start")
  4:         result = self.transform()
  5:         return result

New file (修改后):
  1: class DataProcessor:
  2:     def process(self):
  3:         self.log.info("Start")      <-- 第 3 行改了
  4:         result = self.transform()
  5:         return result

Hunk:
  @@ -3,1 +3,1 @@ 
  -        self.log.debug("Start")
  +        self.log.info("Start")

尝试扩展前 2 行：
  delta_lines_original (old):
    line 1: "class DataProcessor:"
    line 2: "    def process(self):"
  
  delta_lines_new (new):
    line 1: "class DataProcessor:"
    line 2: "    def process(self):"

  比较：相同！✓

如果 old file 第 2 行也被改了：
  delta_lines_original:
    line 1: "class DataProcessor:"
    line 2: "    def process_old(self):"
  
  delta_lines_new:
    line 1: "class DataProcessor:"
    line 2: "    def process_new(self):"

  比较：不同！✗
  
  Mini-match:
    i=0: 不匹配 (class DataProcessor vs class DataProcessor) ✓
    i=1: 不匹配 (def process_old vs def process_new) ✗
    ...
    所有都不匹配 → 放弃扩展
```

---

## 7. 边缘情况处理

### 7.1 文件开头/结尾

**文件开头：**
```python
extended_start1 = max(1, start1 - patch_extra_lines_before)
# max(1, ...) 确保不会小于 1
```

**文件结尾：**
```python
if extended_start1 - 1 + extended_size1 > len_original_lines:
    # 超出文件长度，调整
    delta_cap = extended_start1 - 1 + extended_size1 - len_original_lines
    extended_size1 = max(extended_size1 - delta_cap, size1)
    extended_size2 = max(extended_size2 - delta_cap, size2)
```

### 7.2 编码问题

**位置：** `pr_agent/algo/git_patch_processing.py:34-46`

```python
def decode_if_bytes(original_file_str):
    if isinstance(original_file_str, (bytes, bytearray)):
        try:
            # 首先尝试 utf-8
            return original_file_str.decode('utf-8')
        except UnicodeDecodeError:
            # 依次尝试其他编码
            encodings_to_try = ['iso-8859-1', 'latin-1', 'ascii', 'utf-16']
            for encoding in encodings_to_try:
                try:
                    return original_file_str.decode(encoding)
                except UnicodeDecodeError:
                    continue
            return ""  # 所有编码都失败
    return original_file_str
```

### 7.3 空文件/删除文件

**位置：** `pr_agent/algo/git_patch_processing.py:267-297`

```python
def handle_patch_deletions(patch, original_file_content_str, 
                        new_file_content_str, file_name, edit_type):
    """
    处理整个文件或删除的 patch
    """
    if not new_file_content_str and (edit_type == EDIT_TYPE.DELETED or 
                                  edit_type == EDIT_TYPE.UNKNOWN):
        # 文件被删除，不显示 patch
        patch = None
    else:
        # 过滤删除的 hunks
        patch_lines = patch.splitlines()
        patch_new = omit_deletion_hunks(patch_lines)
        if patch != patch_new:
            patch = patch_new
    return patch
```

### 7.4 无效 hunk

**情况：** hunk header 的行号与文件内容不匹配

**处理：**
```python
# 验证 hunk
is_valid_hunk = check_if_hunk_lines_matches_to_file(...)

if not is_valid_hunk:
    # 不扩展此 hunk，使用原始范围
    extended_start1 = start1
    extended_size1 = size1
    extended_start2 = start2
    extended_size2 = size2
```

### 7.5 特殊 hunk header

**情况：** `'@@ -0,0 +1 @@'` (新文件)

**处理：**
```python
try:
    start1, size1, size2, start1, start2 = map(int, res[:4])
except:  # 特殊情况
    start1, size1, size2 = map(int, res[:3])
    start2 = 0
```

---

## 8. 配置选项

### 8.1 核心配置

```toml
[config]
# 扩展上下文行数
patch_extra_lines_before = 5   # hunk 前添加 5 行
patch_extra_lines_after = 1    # hunk 后添加 1 行

# 动态上下文
allow_dynamic_context = true
max_extra_lines_before_dynamic_context = 10  # 最多向前 10 行

# 跳过特定文件扩展
patch_extension_skip_types = [".pb", ".grpc"]
```

### 8.2 配置建议

**保守配置（节省 token）：**
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
allow_dynamic_context = false
```

**平衡配置（推荐）：**
```toml
patch_extra_lines_before = 5
patch_extra_lines_after = 1
allow_dynamic_context = true
max_extra_lines_before_dynamic_context = 10
```

**激进配置（更好的上下文）：**
```toml
patch_extra_lines_before = 10
patch_extra_lines_after = 3
allow_dynamic_context = true
max_extra_lines_before_dynamic_context = 20
```

---

## 9. 性能考虑

### 9.1 时间复杂度

**分析：**
- 解析 patch: O(n)，n = patch 行数
- 处理每个 hunk: O(m)，m = hunk 行数
- 动态上下文扫描: O(k)，k = max_extra_lines_before_dynamic_context (固定)
- Mini-match 算法: O(p)，p = extra_lines_before (固定)

**总复杂度：** O(n + m) ≈ O(patch_size)

**结论：** 线性时间复杂度，性能良好

### 9.2 空间复杂度

**分析：**
- 存储原始文件行: O(file_lines)
- 存储新文件行: O(file_lines)
- 存储 patch 行: O(patch_lines)
- 存储扩展后的 patch: O(patch_lines + extra_lines)

**总复杂度：** O(file_lines + patch_lines)

**结论：** 合理的内存使用

### 9.3 优化技巧

1. **跳过不需要扩展的文件**
   ```python
   if should_skip_patch(filename):
       return patch_str  # 直接返回，不处理
   ```

2. **限制动态扫描范围**
   ```python
   max_extra_lines_before_dynamic_context = 10  # 固定上限
   ```

3. **惰性处理**
   - 只在需要时解码 bytes
   - 只在需要时处理 new file

---

## 10. 示例演示

### 10.1 完整示例

**原始文件：**
```python
# src/data_processor.py
import logging

class DataProcessor:
    """数据处理核心类"""
    
    def __init__(self, config):
        self.config = config
        self.logger = logging.getLogger(__name__)
        self.cache = {}
    
    def process(self, data):
        """处理数据"""
        self.logger.debug("Starting processing")
        result = self._transform(data)
        self.logger.debug("Processing complete")
        return result
    
    def _transform(self, data):
        """内部转换"""
        # 转换逻辑
        return data.upper()
```

**修改后的文件：**
```python
# src/data_processor.py
import logging

class DataProcessor:
    """数据处理核心类"""
    
    def __init__(self, config):
        self.config = config
        self.logger = logging.getLogger(__name__)
        self.cache = {}
    
    def process(self, data):
        """处理数据"""
        self.logger.info("Starting processing")      # 改了
        result = self._transform(data)
        self.logger.info("Processing complete")      # 改了
        return result + "_processed"              # 改了
    
    def _transform(self, data):
        """内部转换"""
        # 转换逻辑
        return data.upper()
```

**原始 diff：**
```diff
@@ -16,3 +16,3 @@
     def process(self, data):
         """处理数据"""
-        self.logger.debug("Starting processing")
+        self.logger.info("Starting processing")
         result = self._transform(data)
-        self.logger.debug("Processing complete")
+        self.logger.info("Processing complete")
```

### 10.2 扩展过程（patch_extra_lines_before=3, after=1）

**步骤 1：解析 hunk header**
```diff
@@ -16,3 +16,3 @@
```
- start1 = 16, size1 = 3
- start2 = 16, size2 = 3
- section_header = ""

**步骤 2：计算扩展范围**
```
extended_start1 = max(1, 16 - 3) = 13
extended_size1 = 3 + (16 - 13) + 1 = 7

extended_start2 = max(1, 16 - 3) = 13
extended_size2 = 3 + (16 - 13) + 1 = 7
```

**步骤 3：提取 extra lines（前）**
```
Original file lines 13-15:
  13:         self.logger = logging.getLogger(__name__)
  14:         self.cache = {}
  15:     
  16:     def process(self, data):  <-- hunk start

New file lines 13-15:
  13:         self.logger = logging.getLogger(__name__)
  14:         self.cache = {}
  15:     
  16:     def process(self, data):  <-- hunk start
```

**比较：相同！可以扩展**

**步骤 4：提取 extra lines（后）**
```
Original file lines 19-19 (start1 + size1 - 1 + 1):
  19:         return result

New file lines 19-19:
  19:         return result + "_processed"
```

**比较：不同！不添加后置扩展**

**步骤 5：生成扩展后的 diff**
```diff
@@ -13,7 +13,7 @@
         self.logger = logging.getLogger(__name__)
         self.cache = {}
     
     def process(self, data):
         """处理数据"""
-        self.logger.debug("Starting processing")
+        self.logger.info("Starting processing")
         result = self._transform(data)
-        self.logger.debug("Processing complete")
+        self.logger.info("Processing complete")
```

### 10.3 动态上下文示例

**原始文件：**
```python
# src/calculator.py
class Calculator:
    """计算器类"""
    
    def add(self, a, b):
        """加法"""
        return a + b
    
    def subtract(self, a, b):
        """减法"""
        return a - b
```

**修改：**
```python
# src/calculator.py
class Calculator:
    """计算器类"""
    
    def add(self, a, b):
        """加法"""
        return a + b
    
    def subtract(self, a, b):
        """减法"""
        return max(a, b) - min(a, b)  # 改了
```

**原始 diff：**
```diff
@@ -13,1 +13,1 @@ def subtract(self, a, b):
     """减法"""
-        return a - b
+        return max(a, b) - min(a, b)
```

**动态上下文（allow_dynamic_context=true）：**
```
向前扫描（最多 10 行）：
  行 11: "    def add(self, a, b):"
  行 12: '        """加法"""'
  行 13: "        return a + b"
  行 14: ""
  行 15: "    def subtract(self, a, b):"  <-- 匹配 section_header！

找到函数定义！
调整扩展范围到行 15
```

**扩展后的 diff：**
```diff
@@ -15,3 +15,3 @@ def subtract(self, a, b):
     def subtract(self, a, b):
         """减法"""
-        return a - b
+        return max(a, b) - min(a, b)
```

### 10.4 文件变更处理示例

**Old file：**
```python
class Processor:
    def step1(self):
        print("Step 1")
    
    def step2(self):  # 修改点
        print("Step 2 old")
    
    def step3(self):
        print("Step 3")
```

**New file：**
```python
class Processor:
    def step1(self):
        print("Step 1")
    
    def step2_new(self):  # 修改点（函数名也改了）
        print("Step 2 new")
    
    def step3(self):
        print("Step 3")
```

**原始 diff：**
```diff
@@ -5,1 +5,1 @@
-    def step2(self):
+    def step2_new(self):
         print("Step 2 old")
```

**尝试扩展前 2 行：**
```
delta_lines_original (old):
  "    def step1(self):"
  "    def step2(self):"

delta_lines_new (new):
  "    def step1(self):"
  "    def step2_new(self):"

比较：不同！✗

Mini-match 算法：
  i=0: "def step1" vs "def step1" ✓
  i=1: "def step2" vs "def step2_new" ✗
  
找到第一个匹配点 i=0，调整：
  delta_lines_original = ["    def step2(self):"]
  delta_lines_new = ["    def step2_new(self):"]
  
  extended_start1 += 1, extended_size1 -= 1
  extended_start2 += 1, extended_size2 -= 1

比较剩余：仍然不同！✗

放弃扩展，使用原始 hunk
```

---

## 总结

### 核心思想
1. **增加上下文**：在 hunk 前后添加额外行
2. **智能边界**：动态识别函数/类定义
3. **处理变更**：比较 old/new 文件，找到匹配点
4. **保持正确性**：验证 hunk，更新行号

### 关键算法
1. **扩展范围计算**：`max(1, start - before)` + `size + before + after`
2. **动态上下文**：向前扫描寻找 section_header
3. **Mini-match**：找到第一个匹配点
4. **Hunk 验证**：检查行是否匹配文件内容

### 优势
- 更好的 AI 理解（完整上下文）
- 灵活的配置（可调参数）
- 健壮的错误处理（降级到原始 patch）
- 高效的实现（线性复杂度）

这个算法是 PR-Agent 的核心组件之一，显著提升了 AI 对代码变更的理解能力！
