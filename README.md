# Serapeum

[![Documentations](https://img.shields.io/badge/Documentations-blue?logo=github&logoColor=white)](https://serapeum-org.github.io/serapeum/main/)
[![Python Versions](https://img.shields.io/pypi/pyversions/statista.svg)](https://pypi.org/project/serapeum/)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://serapeum-org.github.io/serapeum/latest/)
[![codecov](https://codecov.io/gh/serapeum-org/serapeum/branch/main/graph/badge.svg?token=GQKhcj2pFK)](https://codecov.io/gh/serapeum-org/serapeum)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)](https://github.com/pre-commit/pre-commit)
[![GitHub last commit](https://img.shields.io/github/last-commit/serapeum-org/serapeum)](https://github.com/serapeum-org/serapeum/commits/main)
[![GitHub issues](https://img.shields.io/github/issues/serapeum-org/serapeum)](https://github.com/serapeum-org/serapeum/issues)

[//]: # ([![GitHub stars]&#40;https://img.shields.io/github/stars/serapeum-org/serapeum&#41;]&#40;https://github.com/serapeum-org/serapeum/stargazers&#41;)

[//]: # ([![GitHub forks]&#40;https://img.shields.io/github/forks/serapeum-org/serapeum&#41;]&#40;https://github.com/serapeum-org/serapeum/network/members&#41;)

Serapeum is a modular LLM toolkit. The repo contains a core package with shared
LLM abstractions plus provider integrations (OpenAI, Azure OpenAI, Ollama,
llama.cpp) and supporting docs and examples.

## What is in this repo

- `libs/core/`: Provider-agnostic core (LLM interfaces, prompts, parsers,
  tools, structured outputs).
- `libs/providers/`: Provider adapters (OpenAI, Azure OpenAI, Ollama,
  llama.cpp).
- `docs/`: Architecture and reference docs (MkDocs).
- `examples/`: Usage examples and notebooks.

## Packages

| Package | Location | Description |
|---------|----------|-------------|
| `serapeum-core` | `libs/core/` | Shared LLM interfaces, prompt templates, output parsers, tool schemas |
| `serapeum-openai` | `libs/providers/openai/` | OpenAI Chat Completions and Responses API adapter |
| `serapeum-azure-openai` | `libs/providers/azure-openai/` | Azure OpenAI deployment adapter (extends serapeum-openai) |
| `serapeum-ollama` | `libs/providers/ollama/` | Ollama-backed LLM and embedding adapter |
| `serapeum-llama-cpp` | `libs/providers/llama-cpp/` | Local GGUF model inference via llama-cpp-python |

Each package has its own README with details and examples.

## Quick start

This project uses [uv](https://docs.astral.sh/uv/) for dependency management
and workspace orchestration.

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Sync all workspace dependencies from root
uv sync --dev --active

# Or sync a specific provider
uv sync --package serapeum-openai --active
```

## Testing

The workspace venv lives outside the project, so use `python -m pytest`
directly (not `uv run pytest`):

```bash
# Run all tests
python -m pytest

# Run tests for a specific package
python -m pytest libs/core/tests
python -m pytest libs/providers/openai/tests
python -m pytest libs/providers/ollama/tests
python -m pytest libs/providers/llama-cpp/tests

# Skip end-to-end tests
python -m pytest -m "not e2e"
```

Markers are defined in each package's `pyproject.toml`.

## Links

- Homepage: https://github.com/serapeum-org/serapeum
- Docs: https://serapeum-org.github.io/serapeum
- Security: `SECURITY.md`
