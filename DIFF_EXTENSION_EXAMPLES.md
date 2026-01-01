# Diff 扩展算法 - 可视化示例

## 示例 1：基本扩展（固定上下文）

### 原始文件
```python
# src/utils.py
class Calculator:
    """计算器类"""
    
    def __init__(self):
        self.history = []
    
    def add(self, a, b):
        """加法运算"""
        result = a + b
        self.history.append(f"add({a}, {b}) = {result}")
        return result
    
    def subtract(self, a, b):
        """减法运算"""
        result = a - b
        self.history.append(f"subtract({a}, {b}) = {result}")
        return result
```

### 修改后的文件
```python
# src/utils.py
class Calculator:
    """计算器类"""
    
    def __init__(self):
        self.history = []
        self.cache = {}  # 新增：缓存
    
    def add(self, a, b):
        """加法运算"""
        result = a + b
        self.history.append(f"add({a}, {b}) = {result}")
        self.cache[result] = True  # 新增：缓存结果
        return result
    
    def subtract(self, a, b):
        """减法运算"""
        result = a - b
        self.history.append(f"subtract({a}, {b}) = {result}")
        return result
```

### 原始 Diff（无扩展）
```diff
diff --git a/src/utils.py b/src/utils.py
index abc123..def456 100644
--- a/src/utils.py
+++ b/src/utils.py
@@ -8,3 +8,4 @@     def __init__(self):
         self.history = []
+        self.cache = {}
 
     def add(self, a, b):
@@ -13,2 +14,3 @@     def add(self, a, b):
         result = a + b
         self.history.append(f"add({a}, {b}) = {result}")
+        self.cache[result] = True
         return result
```

### 配置
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
allow_dynamic_context = false
```

### 扩展后的 Diff
```diff
@@ -8,4 +8,5 @@     def __init__(self):
         self.history = []
         self.cache = {}                   <-- 扩展：后 1 行（类方法开始）
 
     def add(self, a, b):            <-- 扩展：前 3 行（函数开始）
         """加法运算"""
         result = a + b
         self.history.append(f"add({a}, {b}) = {result}")
         self.cache[result] = True
         return result                    <-- 扩展：后 1 行（函数结束）
```

### 可视化对比

```
原始 Diff（缺少上下文）：
  ┌─────────────────────────┐
  │  __init__ 结尾        │
  │  + cache              │ ← AI 不知道这属于哪个类
  │  add 开始            │
  └─────────────────────────┘

扩展后 Diff（完整上下文）：
  ┌─────────────────────────────────┐
  │  __init__ 结尾             │
  │  + cache                   │ ← AI 知道这是 Calculator 类
  │  空行（分隔）            │
  │  def add(...)              │ ← 完整的函数定义
  │    """加法运算"""          │
  │    result = a + b          │
  │    history.append(...)        │
  │    + cache[result] = True  │
  │    return result           │
  └─────────────────────────────────┘
```

---

## 示例 2：动态上下文（智能边界）

### 原始文件
```python
# src/data_processor.py
class DataProcessor:
    """数据处理核心类"""
    
    def __init__(self, config):
        self.config = config
        self.logger = logging.getLogger(__name__)
        self.validator = DataValidator()
        self.cache = LRUCache(max_size=1000)
    
    def process(self, data):
        """处理数据的主方法"""
        self.logger.info("Starting data processing")
        
        # 验证数据
        if not self.validator.validate(data):
            self.logger.error("Validation failed")
            return None
        
        # 缓存检查
        cache_key = self._generate_cache_key(data)
        if cache_key in self.cache:
            self.logger.debug("Cache hit")
            return self.cache[cache_key]
        
        # 处理数据
        result = self._transform(data)
        
        # 缓存结果
        self.cache[cache_key] = result
        self.logger.info("Processing complete")
        return result
```

### 修改
```python
# src/data_processor.py
# ... 其他代码不变 ...

    def process(self, data):
        """处理数据的主方法"""
        self.logger.info("Starting data processing")
        
        # 验证数据
        if not self.validator.validate(data):
            self.logger.error("Validation failed")
            return None
        
        # 缓存检查（优化：使用哈希）
        cache_key = self._generate_cache_key(data)
        if cache_key in self.cache:
            self.logger.debug("Cache hit")
            return self.cache[cache_key]
        
        # 处理数据（新增：并行处理）
        result = self._transform_parallel(data)
        
        # 缓存结果
        self.cache[cache_key] = result
        self.logger.info("Processing complete")
        return result
```

### 原始 Diff
```diff
@@ -27,1 +27,1 @@     def process(self, data):
         # 处理数据
-        result = self._transform(data)
+        result = self._transform_parallel(data)
```

### 配置
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
allow_dynamic_context = true
max_extra_lines_before_dynamic_context = 10
```

### 扩展过程

#### 步骤 1：解析 Hunk
```
@@ -27,1 +27,1 @@ 
start1 = 27, size1 = 1
start2 = 27, size2 = 1
section_header = ""
```

#### 步骤 2：动态上下文扫描
```
向前扫描 10 行，寻找 section_header：

  行 17:     def __init__(self, config):
  行 18:         self.config = config
  行 19:         self.logger = logging.getLogger(__name__)
  行 20:         self.validator = DataValidator()
  行 21:         self.cache = LRUCache(max_size=1000)
  行 22:     
  行 23:     def process(self, data):  <-- 找到！这是 section_header
  行 24:         """处理数据的主方法"""
  行 25:         self.logger.info("Starting data processing")
  行 26:         ...
```

**找到匹配！**
- section_header = "def process(self, data):"
- 调整扩展范围到第 23 行（函数定义）

#### 步骤 3：验证 Old/New 文件
```
Original file lines 23-26:
  23:     def process(self, data):
  24:         """处理数据的主方法"""
  25:         self.logger.info("Starting data processing")
  26:         self.logger.info("Starting data processing")

New file lines 23-26:
  23:     def process(self, data):
  24:         """处理数据的主方法"""
  25:         self.logger.info("Starting data processing")
  26:         self.logger.info("Starting data processing")

比较：完全相同！✓
可以应用动态上下文
```

#### 步骤 4：生成扩展 Diff
```diff
@@ -23,6 +23,6 @@     def process(self, data):
     def process(self, data):           <-- 动态上下文：函数定义
         """处理数据的主方法"""           <-- 动态上下文：文档字符串
         self.logger.info("Starting data processing")  <-- 动态上下文：日志
         
         # 处理数据
-        result = self._transform(data)
+        result = self._transform_parallel(data)
         
         return None                        <-- 扩展：后 1 行
```

### 可视化对比

```
固定扩展（3 行前）：
  ┌────────────────────────────┐
  │  Starting processing    │ ← 从这里开始
  │  Validation check     │
  │  Cache check          │
  │  # 处理数据           │
  │  - transform(...)      │ ← AI 不知道这个方法的完整上下文
  └────────────────────────────┘

动态上下文扩展：
  ┌────────────────────────────────┐
  │  def process(self, data):   │ ← 从函数定义开始
  │    """处理数据的主方法"""    │
  │    Starting processing       │
  │    Validation check        │
  │    Cache check             │
  │    # 处理数据              │
  │    - transform(...)         │ ← AI 知道这是 process 方法
  └────────────────────────────────┘
```

---

## 示例 3：处理文件变更（Old vs New）

### Old File（修改前）
```python
# src/api/client.py
class APIClient:
    """API 客户端"""
    
    def __init__(self, base_url, timeout):
        self.base_url = base_url
        self.timeout = timeout
        self.session = None
        self.retry_count = 3
    
    def request(self, method, endpoint, data=None):
        """发送请求"""
        url = f"{self.base_url}/{endpoint}"
        
        # 重试逻辑
        for attempt in range(self.retry_count):
            try:
                response = requests.request(method, url, timeout=self.timeout, json=data)
                if response.status_code == 200:
                    return response.json()
                elif response.status_code >= 500:
                    self._handle_error(response)
                    time.sleep(2 ** attempt)  # 指数退避
                else:
                    return None
            except Exception as e:
                if attempt == self.retry_count - 1:
                    raise
                time.sleep(2 ** attempt)
        
        return None
```

### New File（修改后）
```python
# src/api/client.py
class APIClient:
    """API 客户端"""
    
    def __init__(self, base_url, timeout):
        self.base_url = base_url
        self.timeout = timeout
        self.session = None
        self.retry_count = 3
        self.max_backoff = 60  # 新增：最大退避时间
    
    def request(self, method, endpoint, data=None):
        """发送请求"""
        url = f"{self.base_url}/{endpoint}"
        
        # 重试逻辑（修改：使用退避上限）
        for attempt in range(self.retry_count):
            try:
                response = requests.request(method, url, timeout=self.timeout, json=data)
                if response.status_code == 200:
                    return response.json()
                elif response.status_code >= 500:
                    self._handle_error(response)
                    backoff = min(2 ** attempt, self.max_backoff)  # 修改
                    time.sleep(backoff)
                else:
                    return None
            except Exception as e:
                if attempt == self.retry_count - 1:
                    raise
                time.sleep(2 ** attempt)
        
        return None
```

### 原始 Diff
```diff
@@ -9,2 +9,3 @@     def __init__(self, base_url, timeout):
         self.session = None
         self.retry_count = 3
+        self.max_backoff = 60

@@ -23,1 +23,1 @@             else:
-                    time.sleep(2 ** attempt)
+                    time.sleep(min(2 ** attempt, self.max_backoff))
```

### 配置
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
allow_dynamic_context = false
```

### 扩展过程（第二个 Hunk）

#### 步骤 1：解析 Hunk
```
@@ -23,1 +23,1 @@ 
start1 = 23, size1 = 1
start2 = 23, size2 = 1
```

#### 步骤 2：计算扩展范围
```
extended_start1 = max(1, 23 - 3) = 20
extended_size1 = 1 + (23 - 20) + 1 = 5

extended_start2 = max(1, 23 - 3) = 20
extended_size2 = 1 + (23 - 20) + 1 = 5
```

#### 步骤 3：提取 Extra Lines

```
Original file lines 20-22:
  20:                 elif response.status_code >= 500:
  21:                     self._handle_error(response)
  22:                     time.sleep(2 ** attempt)

New file lines 20-22:
  20:                 elif response.status_code >= 500:
  21:                     self._handle_error(response)
  22:                     backoff = min(2 ** attempt, self.max_backoff)  <-- 改了！
```

#### 步骤 4：比较 Old/New
```
delta_lines_original:
  "elif response.status_code >= 500:"
  "self._handle_error(response)"
  "time.sleep(2 ** attempt)"

delta_lines_new:
  "elif response.status_code >= 500:"
  "self._handle_error(response)"
  "backoff = min(2 ** attempt, self.max_backoff)"

比较：第 3 行不同！✗
```

#### 步骤 5：Mini-Match 算法
```
i = 0:
  original[0:] = ["elif status >= 500:", "handle_error", "sleep"]
  new[0:]      = ["elif status >= 500:", "handle_error", "backoff = ..."]
  比较：不同 ✗

i = 1:
  original[1:] = ["handle_error", "sleep"]
  new[1:]      = ["handle_error", "backoff = ..."]
  比较：不同 ✗

i = 2:
  original[2:] = ["sleep"]
  new[2:]      = ["backoff = ..."]
  比较：不同 ✗

所有都不匹配！放弃扩展
```

#### 步骤 6：生成扩展后的 Diff
```diff
@@ -23,1 +23,1 @@
-                    time.sleep(2 ** attempt)
+                    time.sleep(min(2 ** attempt, self.max_backoff))
```

**结果：** 无法扩展，因为 extra lines 中有变更

### 可视化

```
尝试扩展（3 行前）：
  ┌────────────────────────────┐
  │  elif status >= 500:     │
  │    handle_error()         │
  │    time.sleep(...)       │ ← Old
  │    time.sleep(min(...))   │ ← New（改了！）
  └────────────────────────────┘
  
  问题：extra lines 本身被修改了
  结果：放弃扩展，使用原始 hunk

正确做法：
  ┌────────────────────────────┐
  │  # 发送请求             │
  │  for attempt in ...:    │
  │    try:               │
  │      response = ...     │
  │    elif status >= 500:  │
  │      handle_error()     │
  │      sleep(min(...))    │ ← 如果从更前面开始，可能会不同
  └────────────────────────────┘
```

---

## 示例 4：多层嵌套代码

### 原始文件
```python
# src/parser.py
class Parser:
    def __init__(self):
        self.rules = {}
        self.handlers = {}
    
    def parse(self, data):
        """解析数据"""
        # 预处理
        cleaned = self._preprocess(data)
        
        # 解析
        for token in self._tokenize(cleaned):
            if token.type == 'KEYWORD':
                self._handle_keyword(token)
            elif token.type == 'OPERATOR':
                self._handle_operator(token)
            elif token.type == 'LITERAL':
                self._handle_literal(token)
            else:
                self._handle_unknown(token)
        
        # 后处理
        return self._postprocess(tokens)
```

### 修改
```python
# src/parser.py
class Parser:
    def __init__(self):
        self.rules = {}
        self.handlers = {}
        self.stats = {'parsed': 0, 'errors': 0}  # 新增
    
    def parse(self, data):
        """解析数据"""
        # 预处理
        cleaned = self._preprocess(data)
        
        # 解析
        for token in self._tokenize(cleaned):
            if token.type == 'KEYWORD':
                self._handle_keyword(token)
            elif token.type == 'OPERATOR':
                self._handle_operator(token)
            elif token.type == 'LITERAL':
                self._handle_literal(token)
            elif token.type == 'COMMENT':  # 新增：处理注释
                continue
            else:
                self._handle_unknown(token)
        
        # 后处理（修改：统计）
        self.stats['parsed'] += 1
        return self._postprocess(tokens)
```

### 原始 Diff
```diff
@@ -4,2 +4,3 @@ class Parser:
         self.rules = {}
         self.handlers = {}
+        self.stats = {'parsed': 0, 'errors': 0}

@@ -18,6 +18,8 @@     def parse(self, data):
                 self._handle_literal(token)
+            elif token.type == 'COMMENT':
+                continue
             else:
                 self._handle_unknown(token)
+        self.stats['parsed'] += 1
```

### 配置
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
allow_dynamic_context = true
```

### 扩展后的 Diff（动态上下文）

#### 第一个 Hunk（__init__）
```diff
@@ -4,3 +4,4 @@ class Parser:
     def __init__(self):                 <-- 动态上下文：找到函数定义
         self.rules = {}
         self.handlers = {}
         self.stats = {'parsed': 0, 'errors': 0}  <-- 添加
```

#### 第二个 Hunk（parse 方法）
```diff
@@ -13,12 +13,14 @@     def parse(self, data):  <-- 动态上下文：找到函数定义
         """解析数据"""                    <-- 动态上下文：文档字符串
         
         # 预处理
         cleaned = self._preprocess(data)
         
         # 解析
         for token in self._tokenize(cleaned):
             if token.type == 'KEYWORD':
                 self._handle_keyword(token)
             elif token.type == 'OPERATOR':
                 self._handle_operator(token)
             elif token.type == 'LITERAL':
                 self._handle_literal(token)
+            elif token.type == 'COMMENT':    <-- 新增
+                continue
             else:
                 self._handle_unknown(token)
         
         # 后处理
+        self.stats['parsed'] += 1          <-- 新增
         return self._postprocess(tokens)    <-- 扩展：后 1 行
```

### 可视化

```
动态上下文识别代码结构：

原始 Hunk（从中间开始）：
  ┌──────────────────────────┐
  │  cleaned = ...          │
  │  for token in ...:      │
  │    if KEYWORD:          │
  │      handle_keyword()     │
  │    elif OPERATOR:        │
  │      handle_operator()     │
  │    elif LITERAL:         │
  │      handle_literal()      │
  │    + elif COMMENT:       │
  │    +   continue         │
  └──────────────────────────┘

扩展后（从函数定义开始）：
  ┌──────────────────────────────────┐
  │  def parse(self, data):      │ ← 完整的函数签名
  │    """解析数据"""             │ ← 完整的文档字符串
  │    # 预处理                  │
  │    cleaned = ...              │
  │    # 解析                    │
  │    for token in ...:          │
  │      if KEYWORD:              │
  │        handle_keyword()         │
  │      elif OPERATOR:            │
  │        handle_operator()        │
  │      elif LITERAL:           │
  │        handle_literal()         │
  │      + elif COMMENT:          │
  │      +   continue            │
  │      else:                   │
  │        handle_unknown()        │
  │    # 后处理                  │
  │    + stats['parsed'] += 1    │
  │    return postprocess()       │
  └──────────────────────────────────┘

AI 理解度提升：
  ✓ 知道这是 Parser 类的 parse 方法
  ✓ 知道函数的用途（文档字符串）
  ✓ 知道整体结构（预处理 → 解析 → 后处理）
  ✓ 知道添加的注释处理逻辑的上下文
```

---

## 示例 5：边界情况（文件开头/结尾）

### 原始文件
```python
# src/config.py
# Application configuration
DEBUG = True
DATABASE_URL = "postgresql://localhost:5432/mydb"
CACHE_TTL = 3600
```

### 修改
```python
# src/config.py
# Application configuration
DEBUG = False  # 修改：关闭调试模式
DATABASE_URL = "postgresql://localhost:5432/mydb"
CACHE_TTL = 1800  # 修改：缓存时间减半
```

### 原始 Diff
```diff
@@ -2,1 +2,1 @@
-DEBUG = True
+DEBUG = False

@@ -4,1 +4,1 @@
-CACHE_TTL = 3600
+CACHE_TTL = 1800
```

### 配置
```toml
patch_extra_lines_before = 3
patch_extra_lines_after = 1
```

### 扩展过程

#### 第一个 Hunk（文件开头）

```
解析：
  @@ -2,1 +2,1 @@
  start1 = 2, size1 = 1
  start2 = 2, size2 = 1

计算扩展：
  extended_start1 = max(1, 2 - 3) = 1  ← 不能小于 1
  extended_size1 = 1 + (2 - 1) + 1 = 3

结果：
  @@ -1,3 +1,3 @@  # Application configuration
  # Application configuration
- DEBUG = True
+ DEBUG = False
```

#### 第二个 Hunk（文件结尾）

```
解析：
  @@ -4,1 +4,1 @@
  start1 = 4, size1 = 1
  start2 = 4, size2 = 1

计算扩展：
  extended_start1 = max(1, 4 - 3) = 1
  extended_size1 = 1 + (4 - 1) + 1 = 5

边界检查：
  extended_start1 - 1 + extended_size1 = 0 + 5 = 5
  len_original_lines = 5
  5 > 5? 否 ✓

结果：
  @@ -1,5 +1,5 @@
  # Application configuration
  DEBUG = False
  DATABASE_URL = "postgresql://localhost:5432/mydb"
- CACHE_TTL = 3600
+ CACHE_TTL = 1800
```

### 可视化

```
文件开头边界：
  ┌────────────────────────────┐
  │  line 0 (不存在)       │ ← 试图扩展到 -1，被限制到 1
  │  # Application config   │ ← 从这里开始
  │  - DEBUG = True       │
  │  + DEBUG = False      │
  └────────────────────────────┘

文件结尾边界：
  ┌────────────────────────────┐
  │  # Application config   │
  │  DEBUG = False        │
  │  DATABASE_URL = ...   │
  │  - CACHE_TTL = 3600   │
  │  + CACHE_TTL = 1800   │
  │  (文件结束)           │ ← 试图扩展到 +1，超出文件，被限制
  └────────────────────────────┘
```

---

## 总结：算法效果对比

### 配置影响

| 配置 | Token 开销 | 上下文质量 | 适用场景 |
|------|-----------|------------|----------|
| patch_extra_lines_before=0, after=0 | 低 | 差 | 只看变更 |
| patch_extra_lines_before=3, after=1 | 中 | 好 | 平衡性能和质量 |
| patch_extra_lines_before=10, after=3 | 高 | 优 | 复杂代码 |
| allow_dynamic_context=false | 固定 | 一般 | 可预测的开销 |
| allow_dynamic_context=true | 可变 | 优 | 需要完整上下文 |

### 实际效果

**示例：真实 PR 的 diff**
- 原始：15 行，500 tokens
- 扩展后（before=5, after=1）：25 行，850 tokens
- 增加：70% tokens，提升 3 倍上下文

**AI 质量提升：**
- 更准确的 bug 检测
- 更好的代码建议
- 减少误报（"不知道这个变量在哪里定义"）

### 性能权衡

```
上下文 vs Token 成本

更多上下文:
  ✓ AI 理解更准确
  ✓ 减少"不清楚上下文"的建议
  ✗ 更高的 token 成本
  ✗ 更长的处理时间

更少上下文:
  ✓ 更低的 token 成本
  ✓ 更快的处理
  ✗ 可能遗漏重要细节
  ✗ 更多"需要更多信息"的回答

推荐：平衡配置
  patch_extra_lines_before = 5
  patch_extra_lines_after = 1
  allow_dynamic_context = true
```
