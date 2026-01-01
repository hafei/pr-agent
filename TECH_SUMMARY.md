# PR-Agent 技术架构总结

## 一、核心工作流程 (5分钟理解)

### 1. 触发 → 处理 → 响应

```
用户操作 (PR 打开/评论/新提交)
         ↓
Webhook (GitHub/GitLab/Bitbucket)
         ↓
FastAPI 服务器接收
    ↓
后台任务处理 (不阻塞响应)
    ↓
GitProvider 抽象层
    - 获取 PR 信息 (标题/描述/diff/commit)
    - 获取文件列表
    ↓
TokenHandler 计算和裁剪
    - 检查是否超出 token 限制
    - 智能裁剪 diff (按语言/文件优先级)
    ↓
Jinja2 渲染 Prompt
    - system prompt (角色定义)
    - user prompt (PR 信息 + diff)
    ↓
LiteLLM 调用 AI 模型
    - OpenAI (GPT-5/GPT-4o/o4-mini)
    - Anthropic (Claude 3.7 Sonnet)
    - Google (Gemini)
    - 其他 30+ 模型
    ↓
解析 YAML 响应
    - 使用 Pydantic 验证结构
    - 提取 review 内容
    ↓
格式化为 Markdown
    - 添加可点击链接
    - 添加徽章和标签
    ↓
GitProvider 发布结果
    - PR 评论
    - Labels (Review effort / Security concern)
```

### 2. 关键代码路径

**Webhook 入口：** `pr_agent/servers/github_app.py:38`
```python
@router.post("/api/v1/github_webhooks")
async def handle_github_webhooks(background_tasks, request):
    background_tasks.add_task(handle_request, body, ...)
```

**命令路由：** `pr_agent/agent/pr_agent.py:24-44`
```python
command2class = {
    "review": PRReviewer,           # /review
    "describe": PRDescription,       # /describe
    "improve": PRCodeSuggestions,   # /improve
    "ask": PRQuestions,            # /ask
    # ...
}
```

**PR Review 执行：** `pr_agent/tools/pr_reviewer.py:120-184`
```python
async def run(self):
    # 1. 获取 diff
    await retry_with_fallback_models(self._prepare_prediction)

    # 2. 调用 AI
    response = await self.ai_handler.chat_completion(...)

    # 3. 解析和格式化
    pr_review = self._prepare_pr_review()

    # 4. 发布
    self.git_provider.publish_persistent_comment(pr_review)
```

## 二、核心技术栈 (10个核心组件)

### 1. FastAPI + Uvicorn (Web 服务器)
- **作用：** 接收 webhooks，处理 HTTP 请求
- **关键：** 异步处理，后台任务
- **代码：** `pr_agent/servers/github_app.py`

### 2. Dynaconf (配置管理)
- **作用：** 多层配置系统
- **优先级：** CLI args > Env vars > .pr_agent.toml > Global settings
- **特点：** 自定义合并 loader (section 级合并)
- **代码：** `pr_agent/config_loader.py`, `pr_agent/custom_merge_loader.py`

### 3. LiteLLM (AI 统一接口)
- **作用：** 统一 30+ AI 模型接口
- **支持：** OpenAI, Anthropic, Google, AWS, Azure, HuggingFace 等
- **代码：** `pr_agent/algo/ai_handlers/litellm_ai_handler.py`

### 4. Jinja2 (Prompt 模板引擎)
- **作用：** 模板化 AI prompts
- **特点：** 条件渲染，循环，过滤器
- **代码：** `pr_agent/settings/*_prompts.toml`

### 5. Tiktoken (Token 编码和计数)
- **作用：** 准确计算 tokens
- **策略：** 模型特定编码器
- **代码：** `pr_agent/algo/token_handler.py`

### 6. GitProvider 抽象层
- **作用：** 统一 Git Provider 接口
- **实现：** GitHub, GitLab, Bitbucket, Gitea, Azure DevOps, Gerrit
- **代码：** `pr_agent/git_providers/git_provider.py`

### 7. Diff 处理引擎
- **作用：** 扩展、裁剪、格式化 git diff
- **特点：**
  - 添加上下文行 (前后 N 行)
  - 动态上下文 (函数/类边界)
  - 行号添加
  - Hunks 解耦 (new/old 分离)
- **代码：** `pr_agent/algo/git_patch_processing.py`

### 8. Pydantic (数据验证)
- **作用：** 定义 AI 响应结构
- **代码：** Prompt templates 中的 Pydantic 定义

### 9. Loguru (结构化日志)
- **作用：** JSON 格式日志，结构化输出
- **代码：** `pr_agent/log/__init__.py`

### 10. Pytest (测试框架)
- **作用：** 单元测试和 E2E 测试
- **代码：** `tests/unittest/`, `tests/e2e_tests/`

## 三、关键算法和策略

### 1. Token 管理策略

**问题：** AI 模型有 token 限制 (32K-1M)，大 PR 的 diff 可能超出

**解决方案：**
```python
# 步骤 1: 计算基础 tokens (system + user prompt)
prompt_tokens = count_tokens(system) + count_tokens(user)

# 步骤 2: 添加 diff，检查是否超限
total_tokens = prompt_tokens + count_tokens(diff)
if total_tokens > model_max_tokens:
    # 超出限制，执行裁剪
    diff = prune_diff(diff, model_max_tokens - prompt_tokens - buffer)

# 步骤 3: 裁剪策略
# - 按语言优先级：主要语言优先
# - 按文件重要性：修改 > 新增
# - 添加未处理文件列表
```

**配置：**
```toml
[config]
max_model_tokens = 32000  # 硬性限制
patch_extra_lines_before = 5
patch_extra_lines_after = 1
max_extra_lines_before_dynamic_context = 10
```

### 2. Diff 扩展算法

**问题：** 原始 diff 只有 3 行上下文，AI 理解困难

**解决方案：**
```python
# 为每个 hunk 添加额外上下文
@@ -10,3 +10,4 @@
def foo():  <-- hunk start
-   old_code()
+   new_code()
```

**扩展后：**
```python
@@ -7,10 +7,11 @@
class MyClass:          <-- 动态上下文：找到 class 边界
    def __init__(self):
        self.value = 10   <-- 添加前 3 行

@@ -10,3 +10,4 @@   <-- hunk start
    def foo():
-       old_code()
+       new_code()
        return self.value   <-- 添加后 1 行
```

**动态上下文：**
- `allow_dynamic_context=true`
- 向前寻找最近的函数/类定义
- 最多 10 行

### 3. Prompt 模板化

**问题：** 不同 review 需要不同输出结构，硬编码不灵活

**解决方案：** Jinja2 模板 + Pydantic

```jinja2
# system prompt
You are PR-Reviewer, a language model designed to review a Pull Request.

{%- if require_score %}
Output format includes:
  - score: 0-100 rating
{%- endif %}

{%- if require_security_review %}
  - security_concerns: yes/no + details
{%- endif %}

# Pydantic 定义
class Review(BaseModel):
    {%- if require_score %}
    score: int = Field(description="Rate this PR on a scale of 0-100")
    {%- endif %}
    {%- if require_security_review %}
    security_concerns: str = Field(description="...")
    {%- endif %}
```

**优点：**
- 条件字段 (根据配置启用/禁用)
- 可复用模板
- 类型安全 (Pydantic 验证)

### 4. AI 模型 Fallback 策略

**问题：** 主模型可能失败 (API 错误/限流)

**解决方案：**
```python
# 配置 fallback 模型
[config]
model = "gpt-5-2025-08-07"
fallback_models = ["o4-mini"]

# 自动重试
@retry(stop_after_attempt(2))
async def chat_completion(...):
    try:
        response = await acompletion(model=primary_model, ...)
    except Exception:
        # 使用 fallback 模型
        response = await acompletion(model=fallback_model, ...)
```

### 5. 文件过滤策略

**问题：** 某些文件不应被 review (生成代码、文档、配置)

**解决方案：**
```python
# 配置忽略规则
[ignore]
glob = ["*.md", "*.json", "generated/*"]
regex = ["^generated/"]
language_framework = ["protobuf", "go_gen"]

# 过滤流程
for file in files:
    # 1. Glob 匹配
    if any(fnmatch(file.filename, pattern) for pattern in glob):
        continue

    # 2. Regex 匹配
    if any(re.match(pattern, file.filename) for pattern in regex):
        continue

    # 3. 语言框架匹配
    if file.language in language_framework:
        continue

    # 通过过滤
    filtered_files.append(file)
```

## 四、架构模式

### 1. 工厂模式 (Factory Pattern)

**代码：** `pr_agent/git_providers/__init__.py`
```python
def get_git_provider_with_context(pr_url: str):
    # 根据检测到的 provider 返回相应实例
    if 'github.com' in pr_url:
        return GithubProvider(pr_url)
    elif 'gitlab.com' in pr_url:
        return GitLabProvider(pr_url)
    # ...
```

### 2. 策略模式 (Strategy Pattern)

**代码：** `pr_agent/algo/ai_handlers/base_ai_handler.py`
```python
# 不同的 AI 实现策略
class LiteLLMAIHandler(BaseAiHandler): ...
class LangChainAIHandler(BaseAiHandler): ...
class OpenAIHandler(BaseAiHandler): ...

# 运行时选择
ai_handler = LiteLLMAIHandler  # 或其他实现
```

### 3. 模板方法模式 (Template Method)

**代码：** `pr_agent/tools/pr_reviewer.py`
```python
class PRReviewer:
    async def run(self):
        # 模板方法定义流程
        self._get_pr_diff()
        self._call_ai()
        self._parse_response()
        self._publish_result()
```

### 4. 装饰器模式 (Decorator Pattern)

**代码：** `pr_agent/algo/ai_handlers/litellm_ai_handler.py:263-266`
```python
@retry(
    retry=retry_if_exception_type(openai.APIError),
    stop=stop_after_attempt(2),
)
async def chat_completion(self, ...):
    # 自动重试逻辑
```

### 5. 单例模式 (Singleton Pattern)

**代码：** `pr_agent/algo/token_handler.py:27-39`
```python
class TokenEncoder:
    _encoder_instance = None
    _lock = Lock()

    @classmethod
    def get_token_encoder(cls):
        # 线程安全的单例
        if cls._encoder_instance is None:
            with cls._lock:
                if cls._encoder_instance is None:
                    cls._encoder_instance = encoding_for_model(...)
        return cls._encoder_instance
```

## 五、性能优化点

### 1. 异步 I/O
- 所有 AI 调用异步 (`await`)
- 所有 Git API 调用异步
- Webhook 处理不阻塞响应

### 2. Token 缓存
- Encoder 单例模式 (避免重复初始化)
- Token 估算缓存 (可选)

### 3. Diff 裁剪优化
- 按语言优先级 (主要语言优先)
- 多批次处理 (大 PR 分多个 patch)

### 4. 重复触发防护
```python
_duplicate_push_triggers = DefaultDictWithTimeout(ttl=300)
# 5 分钟内忽略相同 PR 的重复触发
```

### 5. 超时控制
```python
# AI API 超时
ai_timeout = 120  # 2 分钟

# Git 操作超时
CLONE_TIMEOUT_SEC = 20
```

## 六、安全机制

### 1. Webhook 签名验证
```python
# GitHub HMAC-SHA256 签名
verify_signature(body_bytes, webhook_secret, signature_header)
```

### 2. 禁止的 CLI 参数
```python
# Base64 编码的禁止参数
# - API keys (openai.key, anthropic.key)
# - 系统配置 (git_provider, app_name)
# - 安全设置 (secret_provider, enable_auto_approval)

# 运行时验证
CliArgs.validate_user_args(args)
```

### 3. 机密管理
- 环境变量 (容器化部署)
- AWS Secrets Manager
- Google Cloud Storage
- `.secrets.toml` (本地开发，不提交)

### 4. 身份验证
- Identity Provider 抽象层
- 用户资格验证 (`verify_eligibility`)

## 七、可扩展性设计

### 1. 添加新的 Git Provider

```python
# 1. 继承 GitProvider
class MyProvider(GitProvider):
    def get_files(self): ...
    def get_diff_files(self): ...
    def publish_comment(self, comment): ...

# 2. 在工厂中注册
# pr_agent/git_providers/__init__.py
```

### 2. 添加新的 Tool

```python
# 1. 创建 tool 类
class PRMyTool:
    def __init__(self, pr_url, args):
        self.git_provider = get_git_provider_with_context(pr_url)

    async def run(self):
        diff = get_pr_diff(self.git_provider, ...)
        response = await self.ai_handler.chat_completion(...)
        self.git_provider.publish_comment(response)

# 2. 注册命令
# pr_agent/agent/pr_agent.py
command2class = {"my_tool": PRMyTool}
```

### 3. 添加新的 Prompt 模板

```toml
# pr_agent/settings/pr_my_tool_prompts.toml
[pr_my_tool_prompt]
system = """
You are an AI assistant that...
"""

user = """
PR Info:
{{ title }}
{{ description }}

Diff:
{{ diff }}

Please analyze...
"""
```

## 八、配置最佳实践

### 1. 生产环境
```toml
[config]
publish_output = true
publish_output_progress = true
log_level = "WARNING"
model = "gpt-5-2025-08-07"
fallback_models = ["o4-mini"]
ai_timeout = 120
```

### 2. 开发环境
```toml
[config]
publish_output = false
log_level = "DEBUG"
verbosity_level = 2
```

### 3. 仓库特定配置
```toml
# .pr_agent.toml (在被 review 的仓库中)
[pr_reviewer]
require_security_review = true
require_tests_review = true

[pr_code_suggestions]
focus_only_on_problems = true
```

## 九、调试技巧

### 1. 启用详细日志
```toml
[config]
verbosity_level = 2  # 输出完整 prompt 和响应
log_level = "DEBUG"
```

### 2. 输出相关配置
```toml
[config]
output_relevant_configurations = true
```

### 3. 本地 CLI 测试
```bash
# 安装依赖
pip install -r requirements.txt

# 设置环境变量
export OPENAI_API_KEY="sk-..."

# 运行 review
pr-agent --pr_url="https://github.com/owner/repo/pull/1" review
```

### 4. Docker 调试
```bash
# 构建 test 镜像
docker build -f docker/Dockerfile --target test -t pr-agent:test .

# 运行测试
docker run --rm pr-agent:test pytest -v tests/unittest/test_pr_reviewer.py
```

## 十、关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Webhook 服务器 | `pr_agent/servers/github_app.py` |
| 命令路由 | `pr_agent/agent/pr_agent.py` |
| PR Review 工具 | `pr_agent/tools/pr_reviewer.py` |
| AI Handler | `pr_agent/algo/ai_handlers/litellm_ai_handler.py` |
| Token 管理 | `pr_agent/algo/token_handler.py` |
| Diff 处理 | `pr_agent/algo/git_patch_processing.py` |
| PR 处理 | `pr_agent/algo/pr_processing.py` |
| Git Provider 抽象 | `pr_agent/git_providers/git_provider.py` |
| GitHub Provider | `pr_agent/git_providers/github_provider.py` |
| 配置管理 | `pr_agent/config_loader.py` |
| 自定义合并加载器 | `pr_agent/custom_merge_loader.py` |
| 日志系统 | `pr_agent/log/__init__.py` |
| Review Prompt | `pr_agent/settings/pr_reviewer_prompts.toml` |
| 主配置 | `pr_agent/settings/configuration.toml` |
| 忽略配置 | `pr_agent/settings/ignore.toml` |
