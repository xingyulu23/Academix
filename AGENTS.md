# AGENTS.md - Academic MCP Server

## Project Overview

MCP (Model Context Protocol) server for academic paper search, analysis, and BibTeX retrieval.
Aggregates multiple academic APIs: OpenAlex, DBLP, Semantic Scholar, arXiv, CrossRef.

**Stack**: Python 3.11+, Pydantic v2, httpx (async), FastMCP, pytest-asyncio

## Build & Development Commands

This project uses **uv** for dependency management.

```bash
# Install dependencies (creates .venv automatically)
uv sync

# Run the MCP server
uv run academic-mcp
# or
uv run python -m academic_mcp.server

# Run all tests
uv run pytest

# Run single test file
uv run pytest tests/test_models.py

# Run single test class
uv run pytest tests/test_models.py::TestPaper

# Run single test
uv run pytest tests/test_models.py::TestPaper::test_minimal_paper

# Run tests with verbose output
uv run pytest -v

# Run tests with coverage
uv run pytest --cov=academic_mcp

# Skip integration tests (default - they hit real APIs)
uv run pytest -m "not integration"

# Run ONLY integration tests
uv run pytest -m integration tests/test_clients/test_integration.py

# Linting
uv run ruff check src/ tests/
uv run ruff check --fix src/ tests/

# Type checking
uv run mypy src/

# Add a new dependency
uv add <package>

# Add a dev dependency
uv add --group dev <package>

# Update dependencies
uv lock --upgrade
uv sync
```

## Code Style Guidelines

### Formatting
- **Line length**: 100 characters (configured in pyproject.toml)
- **Formatter**: ruff
- **Linter rules**: E, F, I, N, W, UP, B, C4, SIM (see pyproject.toml)

### Imports
- Standard library first, then third-party, then local
- Use `from __future__ import annotations` is NOT needed (Python 3.11+)
- Prefer absolute imports from `academic_mcp`
- Use TYPE_CHECKING guard for typing-only imports:
```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pydantic import ValidationInfo
```

### Type Hints
- **mypy strict mode** - all functions must have type hints
- Use `str | None` not `Optional[str]` (Python 3.10+ union syntax)
- Use `list[str]` not `List[str]` (builtin generics)
- Use `dict[str, Any]` not `Dict[str, Any]`
- Return type `-> None` for functions that don't return
- Annotate class variables: `_client: httpx.AsyncClient | None = None`

### Naming Conventions
- **Classes**: PascalCase (`OpenAlexClient`, `SearchResult`)
- **Functions/methods**: snake_case (`get_paper`, `search_by_author`)
- **Constants**: UPPER_SNAKE_CASE (`BASE_URL`, `DEFAULT_TIMEOUT`)
- **Private methods**: single underscore (`_parse_work`, `_get_client`)
- **Module-level singletons**: underscore prefix (`_aggregator`, `_search_cache`)

### Docstrings
- Use triple-quoted docstrings for all public functions/classes
- Format: Brief description, then Args/Returns sections
```python
def get_paper(self, paper_id: str) -> Paper | None:
    """Get paper details by ID.

    Args:
        paper_id: Paper identifier (DOI, arXiv ID, etc.)

    Returns:
        Paper details or None if not found
    """
```

### Pydantic Models
- Use `Field(...)` for required fields, `Field(default=...)` for optional
- Use `model_config = ConfigDict(...)` not class-level `Config`
- Add descriptions to all fields
- Validators use `@field_validator` decorator with `@classmethod`
```python
class Paper(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    id: str = Field(..., description="Unique identifier")
    title: str = Field(..., description="Paper title")
    year: int | None = Field(default=None, ge=1900, le=2100)

    @field_validator("doi")
    @classmethod
    def normalize_doi(cls, v: str | None) -> str | None:
        # validation logic
        return v
```

### Async Patterns
- All API clients use async/await
- Use `httpx.AsyncClient` for HTTP requests
- Use `asyncio.gather()` for parallel operations
- Client pattern: lazy initialization in `_get_client()` method

### Error Handling
- Use specific exception types, not bare `except:`
- Log errors with context before re-raising
- API clients return `None` for not-found, raise for actual errors
- Use `_handle_api_error()` for user-friendly error messages
```python
try:
    response = await self._get(url)
except httpx.HTTPStatusError as e:
    logger.error(f"HTTP error {e.response.status_code} for {url}: {e}")
    raise
except httpx.TimeoutException as e:
    logger.error(f"Timeout for {url}: {e}")
    raise
```

### Caching
- Global cache instances via getter functions (`get_search_cache()`)
- Different TTLs: search (10min), paper (1hr), bibtex (24hr)
- Cache keys use MD5 hashes of normalized inputs

### Testing
- pytest-asyncio with `asyncio_mode = "auto"`
- Test classes named `TestClassName`
- Test methods named `test_what_it_tests`
- Use respx for mocking HTTP in unit tests
- Fixtures in `conftest.py` for shared mock data
- Integration tests marked with `@pytest.mark.integration`

## Project Structure

```
src/academic_mcp/
    __init__.py
    server.py           # MCP server entry point, tool definitions
    models.py           # Pydantic data models
    cache.py            # TTL cache implementations
    bibtex.py           # BibTeX generation logic
    aggregator.py       # Multi-source orchestrator
    clients/
        __init__.py
        base.py         # Abstract base client
        openalex.py     # OpenAlex API client
        dblp.py         # DBLP API client
        semantic.py     # Semantic Scholar client
        arxiv_client.py # arXiv API client
        crossref.py     # CrossRef API client
tests/
    conftest.py         # Shared fixtures
    test_models.py
    test_bibtex.py
    test_cache.py
    test_clients/
        test_integration.py  # Real API tests
```

## Key Patterns

### Adding a New API Client
1. Create `src/academic_mcp/clients/newclient.py`
2. Inherit from `BaseClient`
3. Implement `search()` and `get_paper()` abstract methods
4. Add to `clients/__init__.py` exports
5. Wire into `AcademicAggregator`

### Adding a New MCP Tool
1. Add to `server.py` using `@mcp.tool()` decorator
2. Use `Field()` for parameter definitions with descriptions
3. Return markdown for human-readable output, JSON for structured

### Client Implementation Pattern
```python
class NewClient(BaseClient):
    BASE_URL = "https://api.example.com"
    SOURCE = PaperSource.NEW_SOURCE

    async def search(self, query: str, limit: int = 10, ...) -> SearchResult:
        cache_key = self._search_cache.search_key("new", query, limit)
        cached = self._search_cache.get(cache_key)
        if cached:
            return cached

        response = await self._get(f"{self.BASE_URL}/search", params={...})
        # Parse response...
        self._search_cache.set(cache_key, result)
        return result
```

## Environment Variables
- `ACADEMIC_MCP_EMAIL` - Email for polite pool access (OpenAlex, CrossRef)
- `SEMANTIC_SCHOLAR_API_KEY` - API key for higher rate limits
