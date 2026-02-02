# Academic MCP Server

An MCP (Model Context Protocol) server for academic paper search, analysis, and BibTeX retrieval.

Aggregates data from **OpenAlex**, **DBLP**, **Semantic Scholar**, **arXiv**, and **CrossRef** to provide a unified academic research interface.

## Features

### 🔍 Multi-Source Paper Search
- **OpenAlex** - Primary source with 100K+ free API calls/day
- **DBLP** - Computer science papers with native BibTeX export
- **Semantic Scholar** - AI-powered recommendations
- **arXiv** - Preprints in physics, math, CS
- **CrossRef** - DOI resolution and publisher metadata

### 🧠 Smart ID Resolution
Automatically detects and handles various paper identifiers:
- **DOI**: `10.1038/nature12345`
- **arXiv DOI**: `10.48550/arXiv.1706.03762` (auto-resolved to arXiv ID)
- **arXiv ID**: `2401.12345` or `arxiv:2401.12345`
- **OpenAlex ID**: `W2741809807`
- **Semantic Scholar ID**: `40-character-hex-string`
- **DBLP Key**: `journals/nature/Smith2024`

### 📚 BibTeX Export
- Native BibTeX from DBLP (highest quality for CS papers)
- Automatic generation from metadata for other sources
- Batch export for multiple papers (comma-separated IDs)
- Proper LaTeX escaping for special characters

### 📊 Citation Analysis
- Citation counts from OpenAlex
- List of citing papers with metadata
- Citation network visualization data (nodes and edges)

### 🤖 AI-Powered Recommendations
- Related paper suggestions via Semantic Scholar
- Based on embedding similarity and citation graphs

## Installation

This project uses [uv](https://github.com/astral-sh/uv) for dependency management.

```bash
# Clone the repository
git clone https://github.com/your-org/academic-mcp.git
cd academic-mcp

# Install dependencies
uv sync
```

## Usage

### As MCP Server

```bash
# Run the MCP server
uv run academic-mcp

# Or with Python directly
uv run python -m academic_mcp.server
```

### Configuration

Set optional environment variables for improved rate limits:

```bash
# For OpenAlex and CrossRef polite pool (recommended)
export ACADEMIC_MCP_EMAIL="your.email@example.com"

# For Semantic Scholar higher rate limits
export SEMANTIC_SCHOLAR_API_KEY="your-api-key"
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `academic_search_papers` | Search papers by keywords, title, author, DOI, date, venue. Supports markdown/json output. |
| `academic_get_paper_details` | Get full metadata for a paper using any supported ID format. |
| `academic_get_bibtex` | Export BibTeX citations (single or batch). Prioritizes DBLP. |
| `academic_get_citations` | Get papers that cite a given paper (via OpenAlex). |
| `academic_search_author` | Find all papers by an author name. |
| `academic_get_related_papers` | AI-powered related paper recommendations (via Semantic Scholar). |
| `academic_get_citation_network` | Get citation network data (nodes/edges) for visualization. |
| `academic_cache_stats` | View cache hit rates and statistics. |

## Examples

### Search for Papers

```
Use academic_search_papers with:
- query: "attention is all you need"
- limit: 5
```

### Get BibTeX (Batch)

```
Use academic_get_bibtex with:
- paper_ids: "10.1038/nature12345,10.48550/arXiv.1706.03762"
```

### Find Related Papers

```
Use academic_get_related_papers with:
- paper_id: "10.48550/arXiv.1706.03762"
- limit: 10
```

## Development

### Run Tests

```bash
# Run all tests
uv run pytest

# Run with coverage
uv run pytest --cov=academic_mcp
```

### Project Structure

```
academic-mcp/
├── src/academic_mcp/
│   ├── __init__.py
│   ├── server.py          # MCP server entry point & lifespan management
│   ├── models.py          # Pydantic data models
│   ├── cache.py           # In-memory cache with TTL
│   ├── bibtex.py          # BibTeX generation logic
│   ├── aggregator.py      # Multi-source orchestrator
│   └── clients/
│       ├── base.py        # Base client class
│       ├── openalex.py    # OpenAlex API client
│       ├── dblp.py        # DBLP API client
│       ├── semantic.py    # Semantic Scholar API client
│       ├── arxiv_client.py # arXiv API client
│       └── crossref.py    # CrossRef API client
├── tests/
│   ├── conftest.py
│   ├── test_bibtex.py
│   ├── test_cache.py
│   └── test_models.py
├── pyproject.toml
└── README.md
```

## API Sources

| Source | Rate Limits | Authentication | Best For |
|--------|-------------|----------------|----------|
| **OpenAlex** | 100K/day with email | Email (optional) | General search, citations, author data |
| **DBLP** | Reasonable use | None | CS papers, high-quality BibTeX |
| **Semantic Scholar** | 100/5min (higher with key) | API key (optional) | AI Recommendations |
| **arXiv** | Unlimited (polite) | None | Preprints (CS, Math, Physics) |
| **CrossRef** | Dynamic | Email (optional) | DOI resolution |

## License

MIT License
