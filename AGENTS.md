# PR-Agent Development Guide

This document provides essential information for working with the pr-agent codebase.

## Project Overview

PR-Agent is an AI-powered pull request assistant that provides automated reviews, descriptions, and code suggestions. It integrates with multiple git providers (GitHub, GitLab, Bitbucket, Gitea, Azure DevOps, Gerrit) and uses various AI models through LiteLLM.

**Key Technologies:**
- Python 3.12+
- FastAPI/Uvicorn for web servers
- Dynaconf for configuration management
- LiteLLM for AI model integration
- Jinja2 for templating
- Loguru for logging
- pytest for testing

## Essential Commands

### Development
```bash
# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Run the CLI tool
pr-agent --pr_url=<PR_URL> <command>

# Install in development mode
pip install -e .
```

### Testing
```bash
# Run unit tests
pytest tests/unittest

# Run unit tests with coverage
pytest tests/unittest --cov=pr_agent --cov-report term --cov-report xml

# Run specific test file
pytest tests/unittest/test_file_filter.py

# Run with verbose output
pytest -v tests/unittest
```

### Docker
```bash
# Build test image
docker build -f docker/Dockerfile --target test -t codiumai/pr-agent:test .

# Run tests in Docker
docker run --rm codiumai/pr-agent:test pytest -v tests/unittest

# Run CLI in Docker
docker run --rm codiumai/pr-agent:latest pr-agent --pr_url=<PR_URL> review
```

### Code Quality
```bash
# Run pre-commit hooks
pre-commit run --all-files

# Run isort
isort pr_agent/

# Run ruff
ruff check pr_agent/
ruff format pr_agent/
```

## Code Organization

```
pr_agent/
├── agent/              # Core agent orchestration
│   └── pr_agent.py     # Main PRAgent class, command routing
├── algo/               # Core algorithms and utilities
│   ├── ai_handlers/    # AI model abstraction layer
│   │   ├── base_ai_handler.py      # Abstract base class
│   │   ├── litellm_ai_handler.py   # LiteLLM implementation
│   │   ├── langchain_ai_handler.py # LangChain implementation
│   │   └── openai_ai_handler.py    # Direct OpenAI implementation
│   ├── types.py        # Type definitions (FilePatchInfo, EDIT_TYPE)
│   ├── token_handler.py # Token management and limits
│   ├── utils.py        # Common utilities (ModelType, Range, etc.)
│   ├── cli_args.py     # CLI argument validation
│   ├── file_filter.py  # File filtering logic
│   └── git_patch_processing.py # Patch parsing
├── git_providers/      # Git provider integrations
│   ├── git_provider.py           # Abstract base class
│   ├── github_provider.py        # GitHub implementation
│   ├── gitlab_provider.py        # GitLab implementation
│   └── ... (other providers)
├── tools/              # PR analysis tools
│   ├── pr_reviewer.py           # /review command
│   ├── pr_description.py        # /describe command
│   ├── pr_code_suggestions.py   # /improve command
│   ├── pr_questions.py          # /ask command
│   └── ... (other tools)
├── servers/            # Server implementations
│   ├── github_app.py           # GitHub webhook server
│   ├── gitlab_webhook.py       # GitLab webhook server
│   ├── bitbucket_app.py        # Bitbucket server
│   └── ... (other servers)
├── settings/           # Configuration files
│   ├── configuration.toml      # Main configuration
│   ├── ignore.toml             # Ignore patterns
│   └── *_prompts.toml          # Prompt templates
├── secret_providers/   # Secret management
│   └── aws_secrets_manager_provider.py
└── config_loader.py    # Configuration management

tests/
├── unittest/          # Unit tests
└── e2e_tests/         # End-to-end tests
```

## Configuration System

PR-Agent uses **Dynaconf** for configuration with a multi-layered approach:

### Configuration Loading Order (highest to lowest priority)
1. CLI arguments (passed via `--key=value`)
2. Environment variables (e.g., `OPENAI__KEY`)
3. Repository-specific config: `.pr_agent.toml` in the repo being reviewed
4. Wiki settings file (if enabled)
5. Global settings: `pr_agent/settings/` directory
6. Default values in code

### Accessing Configuration
```python
from pr_agent.config_loader import get_settings

# Access nested settings
settings = get_settings()
model = settings.config.model
num_findings = settings.pr_reviewer.num_max_findings

# Dynamic update
settings.set("CONFIG.MODEL", "new-model-name")
```

### Key Configuration Files
- `settings/configuration.toml` - Main configuration with all available options
- `settings/.secrets.toml` - Secret keys (not committed, see `.secrets_template.toml`)
- `.pr_agent.toml` - Repository-specific configuration (can be in any repo)
- `pyproject.toml` - Project-specific config under `[tool.pr-agent]`

### Security: Forbidden CLI Arguments
The CLI validates against forbidden arguments (base64 encoded in `cli_args.py:13`). These include:
- API keys and secrets
- Provider URLs and tokens
- Secret provider settings
- Auto-approval settings

**Never** pass sensitive configuration via CLI arguments. Use config files or environment variables instead.

## Code Patterns and Conventions

### AI Handler Pattern
All AI interactions go through the `BaseAiHandler` interface:

```python
from functools import partial
from pr_agent.algo.ai_handlers.litellm_ai_handler import LiteLLMAIHandler

class MyTool:
    def __init__(self, ai_handler: partial[BaseAiHandler,] = LiteLLMAHandler):
        self.ai_handler = ai_handler()
        # Use self.ai_handler.chat_completion() for AI calls
```

### Tool Classes
Each PR command is implemented as a tool class in `pr_agent/tools/`:
- Constructor: Accepts `pr_url`, optional `args`, and `ai_handler`
- `run()`: Async method that executes the tool's logic
- Common pattern: Initialize git provider, get diff/PR data, call AI, publish results

### Git Provider Pattern
```python
from pr_agent.git_providers import get_git_provider_with_context

# Get provider for a PR
git_provider = get_git_provider_with_context(pr_url)

# Common methods
pr = git_provider.pr
diff_files = git_provider.get_diff_files()
pr_description = git_provider.get_pr_description()
languages = git_provider.get_languages()
```

### Async/Await Pattern
Most tool methods are async and use LiteLLM for AI calls:
```python
async def run(self):
    # Do setup
    self.ai_handler = self.ai_handler()

    # Make AI call
    response = await self.ai_handler.chat_completion(
        model=get_model(),
        system=system_prompt,
        user=user_prompt,
        temperature=temperature
    )

    # Publish result
    await self.git_provider.publish_comment(response)
```

### Token Management
The `TokenHandler` class manages token limits:
```python
from pr_agent.algo.token_handler import TokenHandler

token_handler = TokenHandler(
    git_provider.pr,
    vars_dict,
    system_prompt,
    user_prompt
)

# Check if fits in limits
if not token_handler.check_fit():
    # Handle clipping or chunking
```

### Logging
Use Loguru for all logging:
```python
from pr_agent.log import get_logger

logger = get_logger()
logger.info("Message")      # Regular message
logger.debug("Debug info")  # Debug message
logger.error("Error")       # Error message

# With structured data (JSON mode)
logger.info("Message", artifact={"key": "value"})
```

Log level controlled by `settings.config.log_level` or `LOG_LEVEL` env var.

### Jinja2 Templates
Most AI prompts use Jinja2 templates from `settings/*_prompts.toml`:
```python
from jinja2 import Environment, StrictUndefined

env = Environment(undefined=StrictUndefined)
template = env.from_string(get_settings().pr_description_prompt.user)
prompt = template.render(vars_dict)
```

### Error Handling
Use async context and logging for errors:
```python
try:
    await self.ai_handler.chat_completion(...)
except Exception as e:
    get_logger().exception(f"Error in {self.__class__.__name__}: {e}")
    # Optionally publish error comment
    await self.git_provider.publish_comment(f"Error: {str(e)}")
```

## Naming Conventions

### Files and Directories
- Snake_case for all files: `pr_reviewer.py`, `git_provider.py`
- Tools: `pr_<command>.py` (e.g., `pr_description.py`, `pr_reviewer.py`)
- Tests: `test_<module>.py` (e.g., `test_file_filter.py`)

### Classes
- PascalCase: `PRReviewer`, `PRDescription`, `PRAgent`, `TokenHandler`

### Functions and Variables
- Snake_case: `get_pr_diff()`, `filter_ignored()`, `git_provider`

### Configuration
- Sections: lowercase with underscores: `pr_reviewer`, `pr_description`
- Keys: lowercase with underscores: `num_max_findings`, `extra_instructions`
- CLI args: hyphen-separated: `--pr-url`, `--max-findings`

### Constants
- UPPER_CASE: `MAX_FILES_ALLOWED_FULL`, `OUTPUT_BUFFER_TOKENS_HARD_THRESHOLD`

## Testing Patterns

### Unit Test Structure
```python
import pytest
from pr_agent.algo.file_filter import filter_ignored

class TestFilterFeature:
    def test_basic_case(self):
        """Test description."""
        # Arrange
        files = [...]

        # Act
        result = filter_ignored(files)

        # Assert
        assert result == expected, "Message on failure"

    def test_with_monkeypatch(self, monkeypatch):
        """Modify settings for test."""
        monkeypatch.setattr(global_settings.ignore, 'glob', ['*.py'])
        # ... rest of test
```

### Test Configuration
Tests use `monkeypatch` to modify settings without affecting global state:
```python
from pr_agent.config_loader import global_settings

def test_something(self, monkeypatch):
    monkeypatch.setattr(global_settings.config, 'model', 'test-model')
    # Test with test-model
```

### Test Location
- Unit tests: `tests/unittest/test_<module>.py`
- E2E tests: `tests/e2e_tests/test_<integration>.py`
- Follow test naming: `test_<what_is_tested>`

### Fixtures
Create reusable test fixtures in `tests/conftest.py` if needed:
```python
@pytest.fixture
def mock_pr():
    return type('', (object,), {
        'title': 'Test PR',
        'get_files': lambda: [...]
    })()
```

## Important Gotchas

### 1. Configuration Merging
The custom merge loader (`pr_agent/custom_merge_loader.py`) merges TOML files by combining sections and overwriting overlapping values. This is different from standard TOML behavior where entire sections are replaced.

### 2. Forbidden CLI Arguments
Many configuration options cannot be passed via CLI (security). Check `cli_args.py:13` for the full list. Use config files or environment variables instead.

### 3. Async/Await Required
Tool `run()` methods must be async because AI calls are async. Use `asyncio.run()` for CLI entry points.

### 4. Token Limits
PR files are chunked and clipped to fit within token limits defined in configuration:
- `max_description_tokens`: 500
- `max_commits_tokens`: 500
- `max_model_tokens`: 32000 (or model-specific)

Large patches use `large_patch_policy: "clip"` or `"skip"`.

### 5. Git Provider Context
Always use `get_git_provider_with_context(pr_url)` instead of directly instantiating providers. This ensures settings from the repo being reviewed are loaded.

### 6. Settings File Loading
The project looks for `.pr_agent.toml` in the repository root being reviewed (not the pr-agent repo itself). Use `_find_repository_root()` logic from `config_loader.py`.

### 7. Provider-Specific Features
Check capability before using features:
```python
if git_provider.is_supported("get_issue_comments"):
    # Use feature
```

Capabilities include: `get_issue_comments`, `gfm_markdown`, `publish_inline_comments`, etc.

### 8. Retry with Fallback Models
AI calls automatically retry with fallback models if they fail:
```python
from pr_agent.algo.pr_processing import retry_with_fallback_models

response = await retry_with_fallback_models(
    lambda: self.ai_handler.chat_completion(...)
)
```

### 9. Secret Management
Never hardcode API keys. Use:
- Environment variables (preferred)
- AWS Secrets Manager or Google Cloud Storage (if `secret_provider` is configured)
- `.secrets.toml` (local development only, never commit)

### 10. Auto-Best Practices
The codebase has an auto-best-practices feature that detects patterns in the repo and uses them for suggestions. Controlled by `[auto_best_practices]` configuration.

### 11. AI Metadata
When enabled (`enable_ai_metadata=true`), metadata is added to PR files and passed to AI. Used for context enhancement.

### 12. CLI Mode vs Server Mode
- CLI mode: `settings.config.cli_mode = True`, output printed to stdout
- Server mode: Output published as comments/labels via git provider

### 13. Language Detection
PR language is determined from file extensions in `get_main_pr_language()`. Extensions defined in `settings/language_extensions.toml`.

### 14. Incremental Reviews
Use `-i` flag for incremental reviews. Requires provider support and tracks commit history.

### 15. Self-Reflection
Some tools use self-reflection where the AI critiques its own suggestions. Controlled by `self_reflect_on_custom_suggestions` config.

## Common Workflows

### Adding a New Tool
1. Create `pr_agent/tools/pr_my_tool.py`
2. Implement tool class with `__init__` and `async run()`
3. Add to `command2class` in `pr_agent/agent/pr_agent.py`
4. Create prompt template in `settings/pr_my_tool_prompts.toml`
5. Add config section in `settings/configuration.toml`
6. Write tests in `tests/unittest/test_pr_my_tool.py`

### Adding a New Git Provider
1. Create `pr_agent/git_providers/my_provider.py`
2. Inherit from `GitProvider` base class
3. Implement required abstract methods
4. Add to provider factory in `pr_agent/git_providers/__init__.py`
5. Test with E2E tests

### Modifying Configuration
1. Update `settings/configuration.toml` with new options
2. Add validation if needed in `config_loader.py`
3. Update tools to use new configuration
4. Update documentation in `docs/`

### Debugging AI Prompts
1. Enable `verbosity_level=2` in config
2. Set `log_level="DEBUG"`
3. Check logs for prompt content and responses
4. Use `show_relevant_configurations=True` to see active config

## Project-Specific Context

### Legacy Status
This is the legacy open-source version. The commercial Qodo Merge product has additional features.

### Documentation
Docs are built with MkDocs in the `docs/` directory. View at: https://qodo-merge-docs.qodo.ai/

### Community Resources
- Discord: https://discord.com/channels/1057273017547378788/1126104260430528613
- Issues: https://github.com/qodo-ai/pr-agent/issues
- Docs: https://qodo-merge-docs.qodo.ai/

### Deployment Targets
- GitHub App: `github_app` Docker target
- GitLab Webhook: `gitlab_webhook` Docker target
- Bitbucket App: `bitbucket_app` Docker target
- CLI: `cli` Docker target
- Test: `test` Docker target (includes pytest)

### License
Open source, see LICENSE file.

## Environment Variables

Common environment variables:
- `LOG_LEVEL`: Logging level (DEBUG, INFO, WARNING, ERROR)
- `OPENAI__KEY`: OpenAI API key (if using OpenAI directly)
- `ANTHROPIC_API_KEY`: Anthropic API key
- `ANALYTICS_FOLDER`: Path for analytics log files
- `LOG_SANE`: Set to "1" to disable JSON logging format
- `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `GIT_SSL_CAINFO`: SSL certificate paths for git operations

Provider-specific environment variables:
- `GITHUB__USER_TOKEN`: GitHub token
- `GITLAB__PERSONAL_ACCESS_TOKEN`: GitLab token
- `BITBUCKET__USERNAME`, `BITBUCKET__PASSWORD`: Bitbucket credentials

## Additional Notes

### Code Style
- Line length: 120 characters (ruff config)
- Use `isort` for import sorting
- Pre-commit hooks enforce formatting (currently commented out, see `.pre-commit-config.yaml`)

### Dependencies
All dependencies in `requirements.txt`. Dev dependencies in `requirements-dev.txt`.

### Python Version
Requires Python 3.12 or higher (see `pyproject.toml:17`).

### Multi-Stage Docker Builds
Dockerfile uses multi-stage builds for different deployment targets. See `docker/Dockerfile`.

### CI/CD
GitHub Actions workflows in `.github/workflows/`:
- `build-and-test.yaml`: Main CI, runs pytest
- `code_coverage.yaml`: Code coverage with codecov
- `e2e_tests.yaml`: End-to-end tests
- `docs-ci.yaml`: Documentation build
- `pre-commit.yml`: Pre-commit hooks

### Conventional Commits
Commit messages should follow conventional commit format (see CONTRIBUTING.md).
