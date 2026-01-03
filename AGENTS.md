# PR-Agent Development Guide

Essential information for agentic coding in this repository.

## Quick Commands

```bash
# Install
pip install -r requirements.txt -r requirements-dev.txt
pip install -e .

# Testing
pytest tests/unittest                          # Run all unit tests
pytest tests/unittest/test_file_filter.py      # Single test file
pytest tests/unittest/test_file_filter.py::TestFilterFeature::test_basic_case  # Single test
pytest -v tests/unittest                       # Verbose output

# Linting & Formatting
ruff check pr_agent/                           # Lint
ruff format pr_agent/                          # Format
isort pr_agent/                                # Sort imports
pre-commit run --all-files                     # All hooks
```

## Code Style

**Formatting**: 120 char line length, snake_case for files/functions/vars, PascalCase for classes
**Imports**: Grouped stdlib → third-party → local, sorted alphabetically, use `isort`
**Types**: Use `from __future__ import annotations`, prefer explicit typing over `Any`
**Logging**: Always use `from pr_agent.log import get_logger` (Loguru), never print()
**Async**: Most tool methods are async - use `await self.ai_handler.chat_completion()`

## Naming Conventions

- Files: `snake_case.py` (tools: `pr_<command>.py`, tests: `test_<module>.py`)
- Classes: `PascalCase` (e.g., `PRReviewer`, `GitProvider`)
- Functions/variables: `snake_case` (e.g., `get_pr_diff()`, `git_provider`)
- Constants: `UPPER_CASE` (e.g., `MAX_FILES_ALLOWED_FULL`)
- Config sections: `lowercase_with_underscores` (e.g., `pr_reviewer`)

## Key Patterns

### Tool Classes (in `pr_agent/tools/`)
```python
class MyTool:
    def __init__(self, pr_url: str, args: list = None,
                 ai_handler: partial[BaseAiHandler,] = LiteLLMAIHandler):
        self.git_provider = get_git_provider_with_context(pr_url)
        self.ai_handler = ai_handler()

    async def run(self):
        response = await self.ai_handler.chat_completion(...)
        await self.git_provider.publish_comment(response)
```

### Configuration Access
```python
from pr_agent.config_loader import get_settings, global_settings

settings = get_settings()
model = settings.config.model
num_findings = settings.pr_reviewer.num_max_findings
```

### Jinja2 Templates
```python
from jinja2 import Environment, StrictUndefined
env = Environment(undefined=StrictUndefined)
template = env.from_string(get_settings().pr_description_prompt.user)
prompt = template.render(vars_dict)
```

### Error Handling
```python
try:
    await self.ai_handler.chat_completion(...)
except Exception as e:
    get_logger().exception(f"Error in {self.__class__.__name__}: {e}")
```

## Critical Gotchas

1. **Security**: Never pass API keys/secrets via CLI arguments (base64 forbidden list in `cli_args.py:13`). Use env vars or config files.
2. **Forbidden CLI**: Many config options blocked from CLI - check `cli_args.py` for full list.
3. **Async required**: Tool `run()` methods must be async. Use `asyncio.run()` for CLI entry points.
4. **Provider context**: Always use `get_git_provider_with_context(pr_url)` to load repo-specific settings.
5. **Token limits**: Files chunked/clipped to fit `max_description_tokens`, `max_commits_tokens`, `max_model_tokens`.
6. **Settings merge**: Custom TOML loader merges by combining sections, not replacing entire sections.
7. **Secret management**: Never hardcode API keys. Use env vars or secret providers only.
8. **Capability checks**: `git_provider.is_supported("feature_name")` before using provider features.

## Testing

```python
def test_something(self, monkeypatch):
    # Use monkeypatch to modify settings without side effects
    monkeypatch.setattr(global_settings.config, 'model', 'test-model')

    # Arrange → Act → Assert pattern
    files = [...]
    result = filter_ignored(files)
    assert result == expected, "Clear failure message"
```

## Security (from .github/instructions/snyk_rules.instructions.md)

**Mandatory**: Always run `snyk_code_scan` for new first-party code in Snyk-supported languages. Fix issues until scan passes.

## Project Structure

```
pr_agent/
├── agent/              # Main orchestration (PRAgent class)
├── algo/               # Core logic (AI handlers, tokens, types)
├── git_providers/      # Provider integrations (GitHub, GitLab, etc.)
├── tools/              # PR command tools (review, describe, improve)
├── servers/            # Webhook servers
├── settings/           # Config files (TOML)
└── config_loader.py    # Dynaconf configuration management

tests/
├── unittest/          # Unit tests (test_<module>.py)
└── e2e_tests/         # End-to-end tests
```

## Config Loading Priority (high→low)

1. CLI arguments (`--key=value`) - **SECURITY RESTRICTIONS APPLY**
2. Environment variables (`OPENAI__KEY`, `LOG_LEVEL`, etc.)
3. `.pr_agent.toml` in repo being reviewed
4. Wiki settings (if enabled)
5. `pr_agent/settings/` global config
6. Code defaults
