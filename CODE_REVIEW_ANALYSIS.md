# PR-Agent Code Review 工作流与核心技术分析

## 一、Code Review 完整工作流

### 1. 触发阶段

**Webhook 触发流程：**

```
用户提交 PR → Git Provider (GitHub/GitLab/Bitbucket)
    ↓
触发 Webhook → FastAPI 服务器
    ↓
接收 HTTP POST 请求 → 验证签名
    ↓
解析事件类型 → 判断操作类型 (opened/reopened/comment)
```

**关键代码：** `pr_agent/servers/github_app.py:38-54`
```python
@router.post("/api/v1/github_webhooks")
async def handle_github_webhooks(
    background_tasks: BackgroundTasks,
    request: Request,
    response: Response
):
    # 1. 验证 webhook 签名
    # 2. 解析请求体
    # 3. 提取 installation_id 和事件类型
    # 4. 添加后台任务处理
    background_tasks.add_task(handle_request, body, event=event_header)
```

### 2. PR 信息收集阶段

**Git Provider 抽象层：**

```
PRAgent.handle_request()
    ↓
get_git_provider_with_context(pr_url)
    ↓
GitProvider (GitHub/GitLab/Bitbucket/etc.)
    ↓
收集 PR 信息：
  - PR 标题和描述
  - 文件列表 (get_files())
  - Diff 信息 (get_diff_files())
  - 语言统计 (get_languages())
  - Commit 消息 (get_commit_messages())
  - Issue 评论 (get_issue_comments())
```

**关键代码：** `pr_agent/git_providers/git_provider.py:74-200`

抽象接口：
```python
class GitProvider(ABC):
    @abstractmethod
    def get_files(self) -> list: pass

    @abstractmethod
    def get_diff_files(self) -> list[FilePatchInfo]: pass

    @abstractmethod
    def get_pr_description_full(self) -> str: pass

    # ... 更多抽象方法
```

### 3. Diff 处理与 Token 管理阶段

**Token 管理流程：**

```
初始化 TokenHandler
    ↓
计算 prompt 基础 tokens (system + user 模板)
    ↓
get_pr_diff() - 生成扩展 diff
    ↓
检查是否超出 token 限制
    ↓
如果超出 → 裁剪策略
  - 按语言优先级裁剪
  - 按文件重要性裁剪
  - 添加未处理文件列表
    ↓
返回裁剪后的 diff 字符串
```

**关键代码：** `pr_agent/algo/pr_processing.py:38-142`

```python
def get_pr_diff(
    git_provider: GitProvider,
    token_handler: TokenHandler,
    model: str,
    add_line_numbers_to_hunks: bool = False,
    disable_extra_lines: bool = False
):
    # 1. 获取 diff 文件
    diff_files = git_provider.get_diff_files()

    # 2. 生成扩展 diff (添加上下文)
    patches_extended, total_tokens, patches_extended_tokens = \
        pr_generate_extended_diff(
            pr_languages,
            token_handler,
            add_line_numbers_to_hunks,
            patch_extra_lines_before=PATCH_EXTRA_LINES_BEFORE,
            patch_extra_lines_after=PATCH_EXTRA_LINES_AFTER
        )

    # 3. 检查 token 限制
    if total_tokens + OUTPUT_BUFFER_TOKENS_SOFT_THRESHOLD < get_max_tokens(model):
        # 在限制内，返回完整 diff
        return "\n".join(patches_extended)

    # 4. 超出限制，执行裁剪
    patches_compressed_list, total_tokens_list, ... = \
        pr_generate_compressed_diff(pr_languages, token_handler, model, ...)

    # 5. 添加未处理文件信息
    # - 添加的文件列表
    # - 修改的文件列表
    # - 删除的文件列表
```

**Diff 扩展技术：** `pr_agent/algo/git_patch_processing.py:11-31`

```python
def extend_patch(
    original_file_str: str,
    patch_str: str,
    patch_extra_lines_before: int,
    patch_extra_lines_after: int,
    filename: str = "",
    new_file_str: str = ""
) -> str:
    """
    为每个 hunk 添加额外的上下文行
    - patch_extra_lines_before: hunk 前添加的行数
    - patch_extra_lines_after: hunk 后添加的行数
    - 支持动态上下文：寻找最近的函数/类边界
    """
    # 1. 解析原始文件和补丁
    # 2. 为每个 hunk 添加上下文
    # 3. 处理 old/new code 的差异
    # 4. 返回扩展后的 patch
```

### 4. AI Prompt 构建阶段

**Jinja2 模板渲染流程：**

```
加载 prompt 模板 (settings/pr_reviewer_prompts.toml)
    ↓
准备变量字典
  - title, branch, description
  - diff (裁剪后的)
  - language, num_pr_files
  - extra_instructions
  - num_max_findings
  - require_score, require_tests, require_security_review
  - commit_messages_str
  - related_tickets (如果存在)
    ↓
Environment.from_string().render(variables)
    ↓
生成 system_prompt 和 user_prompt
```

**关键代码：** `pr_agent/tools/pr_reviewer.py:213-218`

```python
async def _get_prediction(self, model: str) -> str:
    variables = copy.deepcopy(self.vars)
    variables["diff"] = self.patches_diff  # 更新 diff

    environment = Environment(undefined=StrictUndefined)

    # 渲染 system prompt
    system_prompt = environment.from_string(
        get_settings().pr_review_prompt.system
    ).render(variables)

    # 渲染 user prompt
    user_prompt = environment.from_string(
        get_settings().pr_review_prompt.user
    ).render(variables)

    # 调用 AI
    response, finish_reason = await self.ai_handler.chat_completion(
        model=model,
        temperature=get_settings().config.temperature,
        system=system_prompt,
        user=user_prompt
    )
    return response
```

**Prompt 模板结构：** `pr_agent/settings/pr_reviewer_prompts.toml`

```
system prompt:
- 定义 AI 角色 (PR-Reviewer)
- 说明任务目标 (提供简洁的建设性反馈)
- 定义 diff 格式展示方式
  - __new hunk__ : 新代码
  - __old hunk__ : 旧代码
  - 行号用于引用
- 说明符号含义
  - '+' : 新增代码
  - '-' : 删除代码
  - ' ' : 未改变代码
- 使用 Pydantic 定义 YAML 输出结构
  - Review Base Model
  - KeyIssuesComponentLink
  - TicketCompliance (如果有关联 tickets)
  - ContributionTimeCostEstimate (如果启用)
  - TodoSection (如果扫描 TODO)
  - 可配置字段 (require_score, require_tests, require_security_review 等)

user prompt:
--PR Ticket Info-- (如果存在)
- Ticket URL, Title, Labels, Description, Requirements

--PR Info--
- Today's Date, Title, Branch, PR Description
- Questions 和 Answers (如果是 answer 模式)

--PR code diff--
- 扩展的 diff 字符串

Response:
- YAML 格式的结构化输出
```

### 5. AI 调用阶段

**LiteLLM 统一接口流程：**

```
LiteLLMAIHandler.chat_completion()
    ↓
配置 AI 模型参数
  - model (gpt-5, claude-3.7-sonnet, etc.)
  - temperature (默认 0.2)
  - reasoning_effort (低/中/高，部分模型支持)
  - extended_thinking (Claude 模型支持)
  - seed (如果启用)
  - timeout
    ↓
准备 messages
  - system role
  - user role
  - (可选) image_url (如果分析图片)
    ↓
调用 LiteLLM acompletion()
    ↓
处理响应
  - 提取 content
  - 提取 finish_reason
  - 记录日志
    ↓
重试机制 (失败时使用 fallback 模型)
```

**关键代码：** `pr_agent/algo/ai_handlers/litellm_ai_handler.py:267-412`

```python
@retry(
    retry=retry_if_exception_type(openai.APIError) &
           retry_if_not_exception_type(openai.RateLimitError),
    stop=stop_after_attempt(MODEL_RETRIES),
)
async def chat_completion(
    self,
    model: str,
    system: str,
    user: str,
    temperature: float = 0.2,
    img_path: str = None
):
    # 1. 准备 messages
    messages = [
        {"role": "system", "content": system},
        {"role": "user", "content": user}
    ]

    # 2. 处理特殊模型参数
    kwargs = {
        "model": model,
        "messages": messages,
        "timeout": get_settings().config.ai_timeout,
    }

    # 添加 temperature (如果模型支持)
    if model not in self.no_support_temperature_models:
        kwargs["temperature"] = temperature

    # 添加 reasoning_effort (如果模型支持)
    if model in self.support_reasoning_models:
        kwargs["reasoning_effort"] = get_settings().config.reasoning_effort

    # Claude extended thinking
    if model in self.claude_extended_thinking_models:
        kwargs = self._configure_claude_extended_thinking(model, kwargs)

    # 3. 调用 LiteLLM
    response = await acompletion(**kwargs)

    # 4. 提取结果
    resp = response["choices"][0]['message']['content']
    finish_reason = response["choices"][0]['finish_reason']

    return resp, finish_reason
```

**支持的 AI 模型：**
- OpenAI: GPT-5, GPT-4o, GPT-4 Turbo
- Anthropic: Claude 3.7 Sonnet (支持 extended thinking)
- Google: Gemini
- AWS Bedrock
- Azure OpenAI
- HuggingFace
- Ollama
- DeepSeek, Mistral, Groq, Replicate, xAI

### 6. 响应解析与格式化阶段

**YAML 解析流程：**

```
AI 返回 YAML 字符串
    ↓
load_yaml() (自定义解析器)
    ↓
处理 Pydantic 模型定义
  - Review (主结构)
  - KeyIssuesComponentLink (关键问题)
  - TicketCompliance (ticket 合规检查)
  - ContributionTimeCostEstimate (时间成本估算)
  - TodoSection (TODO 扫描)
    ↓
convert_to_markdown_v2()
    ↓
生成 Markdown 格式的 review
  - 添加标题和徽章
  - 格式化 key_issues (可点击链接)
  - 添加安全审查部分
  - 添加测试检查
  - 添加评分 (如果启用)
  - 添加努力估算 (如果启用)
  - 添加 labels (如果启用)
```

**关键代码：** `pr_agent/tools/pr_reviewer.py:229-279`

```python
def _prepare_pr_review(self) -> str:
    """
    准备 PR review，处理 AI 预测并生成 markdown 格式的反馈
    """
    # 1. 解析 YAML
    data = load_yaml(
        self.prediction.strip(),
        keys_fix_yaml=[...],  # 修复常见格式问题
        first_key='review',
        last_key='security_concerns'
    )

    # 2. 检查必需字段
    if 'review' not in data:
        get_logger().exception("Failed to parse review data")
        return ""

    # 3. 转换为 Markdown
    markdown_text = convert_to_markdown_v2(
        data,
        self.git_provider.is_supported("gfm_markdown"),  # GitHub Flavored Markdown
        incremental_review_markdown_text,  # 增量 review 信息
        git_provider=self.git_provider,
        files=self.git_provider.get_diff_files()
    )

    # 4. 添加帮助文本 (如果启用)
    if get_settings().pr_reviewer.enable_help_text:
        markdown_text += HelpMessage.get_review_usage_guide()

    # 5. 输出相关配置 (如果启用)
    if get_settings().config.output_relevant_configurations:
        markdown_text += show_relevant_configurations('pr_reviewer')

    # 6. 设置 labels (如果启用)
    self.set_review_labels(data)

    return markdown_text
```

### 7. 结果发布阶段

**发布流程：**

```
准备发布
    ↓
检查是否应该发布
  - publish_output=true
  - publish_output_no_suggestions=true 或有实际建议
    ↓
publish_persistent_comment() 或 publish_comment()
    ↓
Git Provider 发布到 PR
  - GitHub: PR 评论 + Labels
  - GitLab: MR 评论 + Labels
  - Bitbucket: PR 评论
    ↓
移除临时 "Preparing review..." 评论
    ↓
更新持久评论 (如果启用)
  - 添加 header: "## PR Reviewer Guide 🔍"
  - 支持多次更新同一个评论
```

**关键代码：** `pr_agent/tools/pr_reviewer.py:172-182`

```python
# 发布 review
if get_settings().pr_reviewer.persistent_comment and \
   not self.incremental.is_incremental:
    final_update_message = get_settings().pr_reviewer.final_update_message
    self.git_provider.publish_persistent_comment(
        pr_review,
        initial_header=f"{PRReviewHeader.REGULAR.value} 🔍",
        update_header=True,
        final_update_message=final_update_message
    )
else:
    self.git_provider.publish_comment(pr_review)

self.git_provider.remove_initial_comment()
```

### 8. Label 设置阶段

**Label 逻辑：**

```
解析 review 数据
    ↓
提取标签信息
  - Review effort [1-5]/5 (如果启用)
  - Possible security concern (如果启用且发现安全问题)
    ↓
更新 PR labels
  - 获取当前 labels
  - 过滤掉旧的 review labels
  - 添加新的 review labels
  - 发布更新后的 labels
```

**关键代码：** `pr_agent/tools/pr_reviewer.py:365-415`

```python
def set_review_labels(self, data):
    if not get_settings().config.publish_output:
        return

    # 1. 检查是否生成相关输出
    if not get_settings().pr_reviewer.require_estimate_effort_to_review:
        get_settings().pr_reviewer.enable_review_labels_effort = False

    if not get_settings().pr_reviewer.require_security_review:
        get_settings().pr_reviewer.enable_review_labels_security = False

    # 2. 准备 labels
    review_labels = []

    # Review effort label
    if get_settings().pr_reviewer.enable_review_labels_effort:
        estimated_effort = data['review']['estimated_effort_to_review_[1-5]']
        estimated_effort_number = int(str(estimated_effort).split(',')[0])
        if 1 <= estimated_effort_number <= 5:
            review_labels.append(f'Review effort {estimated_effort_number}/5')

    # Security concern label
    if get_settings().pr_reviewer.enable_review_labels_security:
        security_concerns = data['review']['security_concerns']
        if 'yes' in security_concerns.lower():
            review_labels.append('Possible security concern')

    # 3. 更新 PR labels
    current_labels = self.git_provider.get_pr_labels(update=True)
    current_labels_filtered = [
        label for label in current_labels
        if not label.lower().startswith('review effort') and
           not label.lower().startswith('possible security concern')
    ]
    new_labels = review_labels + current_labels_filtered

    if sorted(new_labels) != sorted(current_labels):
        self.git_provider.publish_labels(new_labels)
```

---

## 二、核心技术架构

### 1. 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                  Webhook/CLI 入口层                  │
│  (FastAPI Server, CLI, GitHub Actions)              │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  PRAgent 核心协调层              │
│  - 命令路由 (review/describe/improve/etc.)    │
│  - 参数解析和验证                               │
│  - 配置管理 (Dynaconf)                         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   Tool 层 (工具类)                │
│  - PRReviewer (/review)                         │
│  - PRDescription (/describe)                     │
│  - PRCodeSuggestions (/improve)                 │
│  - PRQuestions (/ask)                           │
│  - PRAddDocs (/add_docs)                        │
│  - PRUpdateChangelog (/update_changelog)           │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼─────┐  │  ┌────────▼────────┐
│ Algo 层      │  │  │ AI Handler 层  │
│ - Diff 处理  │  │  │ - LiteLLM       │
│ - Token 管理  │  │  │ - OpenAI        │
│ - 文件过滤   │  │  │ - LangChain     │
│ - Prompt 渲染│  │  └────────────────┘
└───────┬─────┘  │
        │          │
┌───────▼─────┐  │
│ Git Provider  │  │
│ 层           │  │
│ - GitHub      │  │
│ - GitLab      │  │
│ - Bitbucket   │  │
│ - Gitea       │  │
│ - Azure DevOps│  │
│ - Gerrit      │  │
└──────────────┘  │
                  │
        ┌─────────▼─────────┐
        │  Configuration   │
        │  层              │
        │  - TOML files    │
        │  - Env vars      │
        │  - CLI args      │
        │  - Secret stores  │
        └──────────────────┘
```

### 2. 异步编程模型

**全面采用 async/await：**

```python
# Webhook 处理
@router.post("/api/v1/github_webhooks")
async def handle_github_webhooks(
    background_tasks: BackgroundTasks,
    request: Request
):
    # 使用 await 处理异步 I/O
    body = await request.json()

    # 添加后台任务
    background_tasks.add_task(handle_request, body, ...)

# Tool 执行
async def run(self) -> None:
    # AI 调用是异步的
    await retry_with_fallback_models(self._prepare_prediction)

# AI Handler
async def chat_completion(self, model: str, ...):
    # LiteLLM 异步调用
    response = await acompletion(**kwargs)
```

**优势：**
- 高并发处理多个 webhook
- 非阻塞 AI API 调用
- 高效处理 Git Provider API 请求
- 后台任务不阻塞响应

### 3. 配置管理系统

**Dynaconf 多层配置：**

```
优先级 (从高到低):
1. CLI 参数 (--key=value)
   - 安全限制：禁止传递敏感配置
   - 实时更新设置

2. 环境变量 (OPENAI__KEY, ANTHROPIC__KEY)
   - 容器化部署友好
   - 机密信息安全存储

3. 仓库配置 (.pr_agent.toml)
   - 仓库特定设置
   - 动态加载（扫描 .git root）

4. Wiki 设置文件
   - 从 repo wiki 加载配置
   - 团队共享配置

5. 全局设置 (pr_agent/settings/)
   - configuration.toml (主配置)
   - ignore.toml (忽略模式)
   - language_extensions.toml (语言扩展)
   - pr_reviewer_prompts.toml (review prompts)
   - ... (其他 prompt 文件)

6. 默认值 (代码中)
```

**自定义合并加载器：** `pr_agent/custom_merge_loader.py`

```python
# 合并 TOML 文件的特殊逻辑
# - 合并 sections
# - 重写 overlapping values
# - 而不是完全替换 section
```

**配置访问：** `pr_agent/config_loader.py`

```python
from dynaconf import Dynaconf

# 全局设置
global_settings = Dynaconf(
    core_loaders=[],
    loaders=['pr_agent.custom_merge_loader', ...],
    merge_enabled=True,
    settings_files=[...],
    **dynconf_kwargs
)

# 带上下文的设置（每个 PR 请求）
def get_settings(use_context=False):
    try:
        return context["settings"]  # 从 starlette_context
    except Exception:
        return global_settings  # 回退到全局
```

### 4. Token 管理策略

**智能 Token 计数：** `pr_agent/algo/token_handler.py`

```python
class TokenHandler:
    def __init__(self, pr, vars, system, user):
        # 1. 获取编码器（模型特定）
        self.encoder = TokenEncoder.get_token_encoder()

        # 2. 计算 prompt 基础 tokens
        self.prompt_tokens = self._get_system_user_tokens(
            pr, self.encoder, vars, system, user
        )

    def count_tokens(self, patch: str, force_accurate: bool = False):
        # 估算 token 数
        encoder_estimate = len(self.encoder.encode(patch))

        if not force_accurate:
            return encoder_estimate

        # 准确计算（使用模型特定 API）
        return self._get_token_count_by_model_type(patch, encoder_estimate)
```

**编码器选择策略：**

```python
class TokenEncoder:
    @classmethod
    def get_token_encoder(cls):
        model = get_settings().config.model

        # OpenAI 模型：使用 tiktoken encoding_for_model
        if "gpt" in cls._model:
            cls._encoder_instance = encoding_for_model(cls._model)
        else:
            # 其他模型：使用 o200k_base (通用)
            cls._encoder_instance = get_encoding("o200k_base")

        return cls._encoder_instance
```

**Token 限制配置：**

```python
# pr_agent/algo/__init__.py
MAX_TOKENS = {
    "gpt-5-2025-08-07": 1000000,
    "o4-mini": 200000,
    "claude-3-7-sonnet-20250219": 200000,
    "gemini-2.5-pro": 1000000,
    # ... 更多模型
}

# pr_agent/settings/configuration.toml
[config]
max_description_tokens = 500
max_commits_tokens = 500
max_model_tokens = 32000  # 硬性限制
model_token_count_estimate_factor = 0.3  # 增加安全余量
```

### 5. Diff 处理技术

**统一 diff 格式：**

```
## File: 'src/example.py'

__new hunk__
11  unchanged code line0
12  unchanged code line1
13 +new code line2 added
14  unchanged code line3
__old hunk__
 unchanged code line0
 unchanged code line1
-old code line2 removed
 unchanged code line3
```

**关键技术：** `pr_agent/algo/git_patch_processing.py`

1. **Patch 扩展：**
   - 在 hunk 前后添加上下文行
   - `patch_extra_lines_before`: hunk 前行数
   - `patch_extra_lines_after`: hunk 后行数

2. **动态上下文：**
   - `allow_dynamic_context=true`：智能寻找函数/类边界
   - `max_extra_lines_before_dynamic_context`: 最多 10 行
   - 向前寻找最近的 section header（class/function）

3. **行号添加：**
   - `add_line_numbers_to_hunks=true`
   - 为 new hunk 添加行号
   - 便于 AI 引用特定行

4. **Hunk 解耦：**
   - `decouple_and_convert_to_hunks_with_line_numbers`
   - 分离 __new_hunk__ 和 __old_hunk__
   - 添加行号标识

**扩展算法：** `pr_agent/algo/git_patch_processing.py:56-200`

```python
def process_patch_lines(
    patch_str,
    original_file_str,
    patch_extra_lines_before,
    patch_extra_lines_after,
    new_file_str=""
):
    # 1. 解析文件
    file_original_lines = original_file_str.splitlines()
    file_new_lines = new_file_str.splitlines()
    patch_lines = patch_str.splitlines()

    # 2. 解析 hunk headers
    # @@ -old_start,old_size +new_start,new_size @@
    RE_HUNK_HEADER = re.compile(
        r"^@@ -(\d+)(?:,(\d+))? \+(\d+)(?:,(\d+))? @@[ ]?(.*)"
    )

    # 3. 处理每个 hunk
    for i, line in enumerate(patch_lines):
        if line.startswith('@@'):
            # 提取 hunk 信息
            start1, size1, start2, size2 = extract_hunk_headers(match)

            # 4. 扩展上下文
            if patch_extra_lines_before > 0:
                # 添加 hunk 前的行
                delta_lines_before = file_original_lines[
                    extended_start1 - 1 : start1 - 1
                ]

            if patch_extra_lines_after > 0:
                # 添加 hunk 后的行
                delta_lines_after = file_original_lines[
                    start1 + size1 - 1 : start1 + size1 - 1 + patch_extra_lines_after
                ]

            # 5. 动态上下文（如果启用）
            if allow_dynamic_context:
                # 寻找最近的 section header
                # 调整上下文范围到函数/类边界
```

### 6. AI 模型统一接口

**BaseAiHandler 抽象：** `pr_agent/algo/ai_handlers/base_ai_handler.py`

```python
class BaseAiHandler(ABC):
    @abstractmethod
    def __init__(self):
        pass

    @property
    @abstractmethod
    def deployment_id(self):
        pass

    @abstractmethod
    async def chat_completion(
        self,
        model: str,
        system: str,
        user: str,
        temperature: float = 0.2,
        img_path: str = None
    ):
        """
        AI 模型调用接口
        """
        pass
```

**LiteLLM 实现：** `pr_agent/algo/ai_handlers/litellm_ai_handler.py`

**支持的功能：**

1. **多模型提供商：**
   - OpenAI: `OPENAI.KEY`
   - Anthropic: `ANTHROPIC.KEY`
   - Google: `GOOGLE_AI_STUDIO.GEMINI_API_KEY`
   - AWS: `AWS.AWS_ACCESS_KEY_ID`, `AWS.AWS_SECRET_ACCESS_KEY`
   - Azure: `OPENAI.API_TYPE=azure`, `OPENAI.API_VERSION`
   - HuggingFace: `HUGGINGFACE.KEY`
   - Ollama: `OLLAMA.API_BASE`
   - 更多...

2. **高级模型参数：**
   - `reasoning_effort`: 推理强度
   - `extended_thinking`: Claude 扩展思考
   - `temperature`: 温度控制
   - `seed`: 随机种子
   - `timeout`: API 超时
   - `max_tokens`: 最大输出 tokens

3. **重试机制：**
   ```python
   @retry(
       retry=retry_if_exception_type(openai.APIError) &
              retry_if_not_exception_type(openai.RateLimitError),
       stop=stop_after_attempt(MODEL_RETRIES),  # 2 次
   )
   ```

4. **Fallback 模型：**
   - 主模型失败时自动切换
   - `fallback_models=["o4-mini"]`

5. **流式响应：**
   - 某些模型需要流式调用
   - `STREAMING_REQUIRED_MODELS`
   - 自动检测和处理

**模型特定配置：**

```python
# GPT-5 (reasoning models)
if model.startswith('gpt-5'):
    if model.endswith('_thinking'):
        kwargs["reasoning_effort"] = 'low'
    else:
        kwargs["reasoning_effort"] = 'minimal'
    kwargs["allowed_openai_params"] = ["reasoning_effort"]

# Claude extended thinking
if model in self.claude_extended_thinking_models:
    kwargs["thinking"] = {
        "type": "enabled",
        "budget_tokens": extended_thinking_budget_tokens
    }
    kwargs["max_tokens"] = extended_thinking_max_output_tokens
    kwargs["temperature"] = 1  # 必须为 1
```

### 7. 模板引擎 (Jinja2)

**Prompt 模板化：**

```python
from jinja2 import Environment, StrictUndefined

environment = Environment(undefined=StrictUndefined)

# 渲染模板
system_prompt = environment.from_string(template_str).render(variables)
```

**条件渲染：**

```jinja2
{%- if require_score %}
  score: str = Field(description="Rate this PR on a scale of 0-100")
{%- endif %}

{%- if related_tickets %}
  ticket_compliance_check: List[TicketCompliance]
{%- endif %}

{%- for ticket in related_tickets %}
Ticket: {{ ticket.title }}
{%- endfor %}
```

**模板特点：**
- 使用 `StrictUndefined`：未定义变量抛出错误
- 支持条件判断 (`if`)
- 支持循环 (`for`)
- 过滤 (`|trim`, `|upper` 等)
- 字符串拼接
- YAML 输出结构

### 8. 文件过滤与忽略

**文件过滤流程：** `pr_agent/algo/file_filter.py`

```python
def filter_ignored(files):
    # 1. 获取忽略规则
    glob_patterns = global_settings.ignore.glob
    regex_patterns = global_settings.ignore.regex

    # 2. 过滤文件
    filtered_files = []
    for file in files:
        filename = file.filename

        # Glob 模式匹配
        if any(fnmatch(filename, pattern) for pattern in glob_patterns):
            continue

        # Regex 模式匹配
        if any(re.match(pattern, filename) for pattern in regex_patterns):
            continue

        # 语言框架忽略
        language = file.language
        if language in global_settings.ignore.language_framework:
            continue

        filtered_files.append(file)

    return filtered_files
```

**忽略配置：** `pr_agent/settings/ignore.toml`

```toml
[ignore]
# Glob 模式
glob = [
    "*.md",
    "*.txt",
    "*.json",
    "generated/*"
]

# Regex 模式
regex = [
    "^generated/",
    ".*\\.(pb|grpc)$"
]

# 语言框架（自动生成的代码）
language_framework = [
    "protobuf",
    "go_gen"
]
```

### 9. 安全机制

**1. 签名验证：** `pr_agent/servers/utils.py`

```python
def verify_signature(body_bytes, secret, signature_header):
    """
    验证 GitHub webhook 签名
    """
    import hmac
    import hashlib

    if not secret:
        return

    # 计算 HMAC-SHA256
    hmac_obj = hmac.new(
        secret.encode('utf-8'),
        body_bytes,
        hashlib.sha256
    )
    digest = "sha256=" + hmac_obj.hexdigest()

    # 比较签名
    if not hmac.compare_digest(digest, signature_header):
        raise ValueError("Invalid signature")
```

**2. 禁止的 CLI 参数：** `pr_agent/algo/cli_args.py`

```python
class CliArgs:
    @staticmethod
    def validate_user_args(args: list) -> (bool, str):
        # 解码禁止的参数（Base64 编码）
        _encoded_args = 'c2hhcmVkX3NlY3JldDpkdXNlc3RfaW5hYmxlX2FhcHByb3ZhbDpZW5hYmxlX2Z0cnV0aW9uOmludmlzaWJsZV9hdXRvX2FwcHJvdmFsOmVuYWJsZV9tYW51YWxfYXBwcm92YWw6ZW5hYmxlX2F1dG9fYXBwcm92YWw='

        forbidden_cli_args = []
        for e in _encoded_args.split(':'):
            forbidden_cli_args.append(b64decode(e).decode())

        # 验证参数
        for arg in args:
            if arg.startswith('--'):
                arg_word = arg.lower()
                for forbidden_arg in forbidden_cli_args:
                    if forbidden_arg in arg_word:
                        return False, forbidden_arg

        return True, ""
```

**禁止的参数包括：**
- `openai.key`, `anthropic.key` (API 密钥)
- `git_provider`, `app_name` (系统配置)
- `enable_auto_approval`, `approve_pr_on_self_review` (关键配置)
- `secret_provider`, `base_url` (安全设置)

**3. 机密管理：** `pr_agent/secret_providers/`

```python
# AWS Secrets Manager
class AWSSecretsManagerProvider:
    def get_secret(self, secret_name: str):
        client = boto3.client('secretsmanager')
        response = client.get_secret_value(SecretId=secret_name)
        return response['SecretString']

# Google Cloud Storage
class GoogleCloudStorageSecretProvider:
    def get_secret(self, bucket_name: str, object_name: str):
        client = storage.Client()
        bucket = client.bucket(bucket_name)
        blob = bucket.blob(object_name)
        return blob.download_as_text()
```

**4. 身份验证：** `pr_agent/identity_providers/`

```python
class IdentityProvider(ABC):
    @abstractmethod
    def verify_eligibility(
        self,
        provider: str,
        user_id: str,
        pr_url: str
    ) -> Eligibility:
        """
        验证用户是否有资格使用 PR-Agent
        """
        pass
```

---

## 三、性能优化策略

### 1. Token 优化

**动态裁剪：**
- 按语言优先级：主要语言优先
- 按文件重要性：修改 > 新增
- 多批次：大 PR 分多个 patch 处理

**配置：**
```toml
[config]
patch_extra_lines_before = 5
patch_extra_lines_after = 1
max_extra_lines_before_dynamic_context = 10
large_patch_policy = "clip"  # 或 "skip"
```

### 2. 并发处理

**后台任务：**
```python
# FastAPI BackgroundTasks
background_tasks.add_task(handle_request, body, event=event)
```

**异步 I/O：**
- 所有 AI 调用异步
- 所有 Git Provider API 调用异步
- 不阻塞 webhook 响应

### 3. 缓存策略

**配置缓存：**
```python
from starlette_context import context

# 每个 PR 请求的独立设置
context["settings"] = copy.deepcopy(global_settings)
```

**重复触发防护：**
```python
_duplicate_push_triggers = DefaultDictWithTimeout(ttl=300)
```

### 4. 超时控制

```python
# AI API 超时
ai_timeout = 120  # 2 分钟

# Git 克隆超时
CLONE_TIMEOUT_SEC = 20

# Webhook 处理超时
push_trigger_pending_tasks_ttl = 300  # 5 分钟
```

---

## 四、可观测性

### 1. 日志系统

**Loguru 配置：** `pr_agent/log/__init__.py`

```python
def setup_logger(level="INFO", fmt=LoggingFormat.CONSOLE):
    # JSON 格式（生产环境）
    logger.add(
        sys.stdout,
        filter=inv_analytics_filter,
        level=level,
        format="{message}",
        serialize=True,  # JSON
    )

    # 控制台格式（开发环境）
    logger.add(
        sys.stdout,
        level=level,
        colorize=True,
        filter=inv_analytics_filter,
    )

    # Analytics 日志（文件）
    if log_folder:
        logger.add(
            log_file,
            filter=analytics_filter,
            level=level,
            serialize=True,
        )
```

**结构化日志：**
```python
get_logger().info(
    "Reviewing PR",
    artifact={
        "pr_url": pr_url,
        "num_files": len(files),
        "model": model
    }
)
```

### 2. 追踪集成

**LiteLLM Callbacks：**

```toml
[litellm]
enable_callbacks = false
success_callback = ["langfuse", "langsmith"]
failure_callback = []
service_callback = []
```

**元数据：**
```python
metadata = {
    "trace_name": command,
    "tags": [git_provider, command, f'version:{get_version()}'],
    "trace_metadata": {
        "command": command,
        "pr_url": pr_url,
    }
}
```

### 3. GitHub Actions 输出

```python
def github_action_output(data, key):
    """
    写入 GitHub Actions 输出
    """
    if os.getenv('GITHUB_OUTPUT'):
        with open(os.getenv('GITHUB_OUTPUT'), 'a') as f:
            f.write(f"{key}={json.dumps(data)}\n")
```

---

## 五、扩展性设计

### 1. 添加新的 Git Provider

**步骤：**
1. 继承 `GitProvider`
2. 实现抽象方法
3. 在工厂中注册

**示例：**
```python
# pr_agent/git_providers/my_provider.py
class MyProvider(GitProvider):
    def __init__(self, pr_url: str, token: str = None):
        # 初始化 API 客户端
        pass

    def get_files(self) -> list:
        # 获取 PR 文件列表
        pass

    def get_diff_files(self) -> list[FilePatchInfo]:
        # 获取 diff 信息
        pass

    def publish_comment(self, comment: str):
        # 发布评论
        pass

    # ... 其他抽象方法
```

### 2. 添加新的 Tool

**步骤：**
1. 创建 tool 类
2. 实现 `__init__` 和 `async run()`
3. 添加到 `command2class`

**示例：**
```python
# pr_agent/tools/pr_my_tool.py
class PRMyTool:
    def __init__(self, pr_url: str, args: list = None):
        self.git_provider = get_git_provider_with_context(pr_url)
        self.vars = {...}
        self.token_handler = TokenHandler(...)

    async def run(self):
        # 1. 获取 diff
        self.patches_diff = get_pr_diff(...)

        # 2. 调用 AI
        response = await self.ai_handler.chat_completion(...)

        # 3. 发布结果
        self.git_provider.publish_comment(response)

# pr_agent/agent/pr_agent.py
command2class = {
    "my_tool": PRMyTool,
    # ... 其他命令
}
```

### 3. 添加新的 Prompt 模板

**步骤：**
1. 创建 `.toml` 文件
2. 定义 system 和 user prompts
3. 使用 Jinja2 模板语法

**示例：**
```toml
# pr_agent/settings/pr_my_tool_prompts.toml
[pr_my_tool_prompt]
system = """
You are an AI assistant that...
"""

user = """
Here is the PR information:
Title: {{ title }}
Description: {{ description }}
Diff:
{{ diff }}

Please analyze and provide...
"""
```

---

## 六、关键技术栈总结

### 核心框架
- **Python 3.12+**: 主要编程语言
- **FastAPI/Uvicorn**: Web 框架和 ASGI 服务器
- **asyncio**: 异步编程支持

### AI 集成
- **LiteLLM**: 统一 AI 模型接口
- **tiktoken**: Token 编码和计数
- **Jinja2**: Prompt 模板引擎

### Git 集成
- **PyGithub**: GitHub API
- **python-gitlab**: GitLab API
- **atlassian-python-api**: Bitbucket API
- **GitPython**: Git 操作

### 配置管理
- **Dynaconf**: 多层配置系统
- **pydantic**: 数据验证

### 日志和追踪
- **Loguru**: 结构化日志
- **Langfuse/Langsmith**: 可选追踪集成

### 测试
- **pytest**: 单元测试
- **pytest-cov**: 代码覆盖率

### 部署
- **Docker**: 容器化部署
- **GitHub Actions**: CI/CD
- **Gunicorn**: 生产服务器（uvicorn workers）
