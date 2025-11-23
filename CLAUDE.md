# CLAUDE.md - AI Assistant Guide for Bananalyzer

## Project Overview

Bananalyzer is an open-source AI Agent evaluation framework for **web tasks** using Playwright. It provides a standardized way to test and benchmark AI agents that interact with web pages for data extraction and navigation tasks.

**Purpose**: Evaluate AI agents on their ability to extract structured data from web pages using static snapshots (MHTML/HAR) to ensure reproducible results.

**Target Users**: AI/ML engineers building web automation agents, researchers working on web scraping AI, and developers benchmarking browser automation solutions.

**Problem Solved**: Websites change over time, have anti-bot protections, and vary in structure. Bananalyzer provides static website snapshots and standardized evaluation criteria for consistent AI agent testing.

**Current Version**: 0.12.0
**Maintainer**: Reworkd (reworkd.ai)

## Tech Stack

### Core Dependencies
| Package | Version | Purpose |
|---------|---------|---------|
| Python | >=3.11,<4.0 | Runtime |
| playwright | >=1.47.0 | Browser automation |
| pydantic | ^2.8.2 | Data validation |
| pytest | ^8.3.2 | Test framework |
| pytest-asyncio | ^0.24.0 | Async test support |
| pytest-xdist | ^3.6.1 | Parallel test execution |
| deepdiff | ^8.0.0 | JSON comparison |
| boto3 | ^1.35.8 | AWS S3 access |
| numpy | ^2.1.0 | Data processing |
| tabulate | ^0.9.0 | Results formatting |
| requests | ^2.32.3 | HTTP client |
| psutil | ^6.0.0 | Process utilities |

### Development Dependencies
- mypy ^1.11.2 (type checking)
- ruff ^0.6.2 (linting/formatting)
- pytest-mock ^3.14.0
- pytest-cov ^5.0.0
- types-requests, types-tabulate (type stubs)

## Project Structure

```
bananalyzer/
├── __init__.py           # Public API exports
├── __main__.py           # CLI entry point and argument parsing
├── __version.py          # Version from package metadata
├── schema.py             # Pydantic schemas for CLI args
├── hooks.py              # Pytest plugin for results reporting
├── junit.py              # JUnit XML report enrichment
├── data/
│   ├── example_schemas.py    # Example and Eval Pydantic models
│   ├── example_fetching.py   # Load examples from JSON files
│   ├── example_s3.py         # S3 download utilities
│   └── example_detail_schemas.py  # Schema/goal lookups
└── runner/
    ├── agent_runner.py       # AgentRunner ABC interface
    ├── runner.py             # Test file generation and execution
    ├── generator.py          # Pytest test code generator
    ├── evals.py              # Evaluation logic (JSON matching)
    ├── website_responder.py  # URL resolution for examples
    └── null_agent_wrapper.py # Reference implementation

static/                   # Example website snapshots (MHTML files)
├── examples.json         # Training examples definitions
├── test_examples.json    # Test set examples
├── schemas.json          # JSON schemas for extraction
├── goals.json            # Goal descriptions per schema
└── [id]/index.mhtml      # Static website snapshots

tests/                    # Unit tests (~1167 lines)
scripts/                  # Utility scripts
.github/workflows/        # CI/CD (lint, mypy, pytest, publish)
```

## Key Files

| File | Purpose |
|------|---------|
| `bananalyzer/__main__.py` | CLI entry point, arg parsing, test orchestration |
| `bananalyzer/runner/agent_runner.py` | `AgentRunner` ABC that users must implement |
| `bananalyzer/runner/runner.py` | Creates temp test files and runs pytest |
| `bananalyzer/runner/generator.py` | Generates pytest test code from examples |
| `bananalyzer/runner/evals.py` | JSON matching and evaluation logic |
| `bananalyzer/data/example_schemas.py` | `Example` and `Eval` Pydantic models |
| `bananalyzer/hooks.py` | Pytest plugin for result aggregation |
| `static/examples.json` | Training example definitions |
| `pyproject.toml` | Poetry configuration and dependencies |

## Architecture & Data Flow

### Execution Flow
1. User runs `bananalyze ./my_agent.py`
2. CLI parses args and loads user's `AgentRunner` implementation
3. Examples are loaded from JSON and filtered by CLI args
4. For each example, a pytest test class is dynamically generated
5. Tests are written to temp files in `.banana_cache/`
6. pytest-xdist runs tests in parallel
7. Results are aggregated and displayed via custom pytest hooks
8. JUnit XML reports are enriched with metadata

### Key Abstractions

**AgentRunner (ABC)**: Users implement this to define their agent
```python
class AgentRunner(ABC):
    @abstractmethod
    async def run(self, page: Page, eval_context: Example) -> AgentResult:
        pass
```

**Example**: Defines a test case with URL, expected output, schema
```python
class Example(BaseModel):
    id: str
    url: str
    source: Literal["html", "mhtml", "hosted", "har"]
    type: ExampleType  # "listing", "detail", "listing_detail"
    goal: str
    schema_: Dict[str, Any]
    evals: List[Eval]
```

**Eval**: Defines success criteria
```python
class Eval(BaseModel):
    type: Literal["json_match", "end_url_match"]
    expected: AllowedJSON | None
    options: Optional[list[AllowedJSON]]
```

### Website Sources
- **mhtml**: Static MHTML snapshots (most common)
- **har**: HTTP Archive recordings played back via Playwright
- **hosted**: Live websites (less deterministic)
- **html**: Static HTML files

## Development Setup

### Prerequisites
- Python 3.11+
- Poetry
- Chromium (installed via Playwright)

### Installation
```bash
# Clone repository
git clone https://github.com/reworkd/bananalyzer.git
cd bananalyzer

# Install dependencies
poetry install

# Install Playwright browsers
poetry run playwright install chromium

# Run tests
poetry run pytest -vv .
```

### Running the Evaluation Suite
```bash
# Basic usage
bananalyze ./my_agent.py

# With options
bananalyze ./my_agent.py --headless -n 4 -c software -i detail

# Download examples first
bananalyze --download ./my_agent.py
```

## Common Commands

```bash
# Run all tests
poetry run pytest -vv .

# Type checking
poetry run mypy .

# Format code
poetry run ruff format .

# Lint check
poetry run ruff format --check

# Run specific test file
poetry run pytest tests/test_evals.py -v

# Run with coverage
poetry run pytest --cov=bananalyzer
```

## Testing

### Test Structure
- `tests/test_evals.py` - Evaluation function tests
- `tests/test_runner.py` - Test runner tests
- `tests/test_generator.py` - Code generation tests
- `tests/test_examples.py` - Example validation tests
- `tests/test_example_eval.py` - Full evaluation tests
- `tests/test_website_responder.py` - URL resolution tests
- `tests/test_junit.py` - Report enrichment tests

### Running Tests
```bash
pytest                           # All tests
pytest tests/test_evals.py      # Single file
pytest -x                        # Stop on first failure
pytest -k "test_sanitize"       # Pattern match
```

## Dependency Issues & Solutions

### Critical Issues

#### 1. NumPy 2.1.0 Compatibility
**Issue**: NumPy 2.x has breaking changes from 1.x
**Solution**: Pin to `numpy<2.0` if compatibility issues arise, or ensure all consumers support NumPy 2.x

#### 2. Playwright Browser Installation
**Issue**: Playwright requires browser binaries that aren't installed with pip
**Solution**: Always run `playwright install chromium` after package installation
```bash
pip install bananalyzer && playwright install chromium
```

#### 3. pytest-xdist Worker Isolation
**Issue**: Tests may bleed state when using `--single_browser_instance`
**Solution**: Default to separate browser instances per test (slower but reliable)

#### 4. MHTML CRLF Line Endings
**Issue**: Git normalizes CRLF to LF, breaking MHTML files
**Solution**: The `convert_to_crlf()` function in `example_fetching.py` handles this, or run:
```bash
unix2dos static/*/*.mhtml  # On macOS/Linux
```

### Moderate Issues

#### 5. AWS Credentials for S3
**Issue**: `boto3` uses default credential chain; may fail without AWS config
**Solution**: Set environment variables or use `--examples_bucket` for public buckets
```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
```

#### 6. GitHub Actions Deprecated Syntax
**Issue**: `set-output` command is deprecated
**Solution**: Update `.github/workflows/python.yml` to use `$GITHUB_OUTPUT`:
```yaml
# Old (deprecated)
echo "::set-output name=should_publish::true"
# New
echo "should_publish=true" >> $GITHUB_OUTPUT
```

#### 7. Pydantic V2 Migration
**Issue**: Uses Pydantic V2 syntax; incompatible with V1
**Solution**: Ensure `pydantic>=2.0.0` is installed; no V1 fallback

### Minor Issues

#### 8. Type Annotations
**Issue**: Some functions use `type()` instead of `isinstance()` for type checking
**Location**: `bananalyzer/data/example_schemas.py:176`
**Solution**: Replace `type(values.get("schema_")) == str` with `isinstance()`

#### 9. Broad Exception Handling
**Issue**: Several places catch `Exception` too broadly
**Locations**: `hooks.py:114`, `junit.py:20`
**Solution**: Catch specific exceptions for better error messages

## Code Quality Issues

### Security Concerns

1. **Subprocess Execution**: `download_examples()` runs git clone without sanitization
   - Risk: Low (hardcoded repo URL)
   - Location: `example_fetching.py:64`

2. **Dynamic Code Generation**: Test files are generated from example data
   - Risk: Medium if examples.json is compromised
   - Location: `runner.py:41-93`

### Performance Bottlenecks

1. **Sequential Example Loading**: Examples loaded and validated one at a time
   - Location: `example_fetching.py:97-109`
   - Improvement: Batch validation with asyncio

2. **Full File Read for CRLF Conversion**: Reads entire MHTML into memory
   - Location: `example_fetching.py:44-51`
   - Improvement: Stream processing for large files

### Anti-patterns

1. **Circular Import Avoidance**: Inline imports in `Example.get_static_url()`
   - Location: `example_schemas.py:139`
   - Improvement: Restructure module dependencies

2. **Global State in `example_detail_schemas.py`**: Module-level `get_examples_path()` call
   - Location: `example_detail_schemas.py:10`
   - Improvement: Lazy initialization

## Architecture Notes for AI Assistants

### Key Extension Points

1. **Custom Agent Implementation**: Extend `AgentRunner` class
2. **Custom Evaluation Types**: Add to `Eval.type` literal and implement in `eval_results()`
3. **New Source Types**: Add to `Example.source` and create new `WebsiteResponder`

### Important Patterns

1. **Test Generation**: Tests are Python code strings written to temp files
2. **Fixture Scoping**: `session` scope for browser, `class` scope for page
3. **Marker System**: Custom pytest markers with `bananalyzer_` prefix for result grouping

### Common Modifications

**Adding a new test type**:
1. Add to `ExampleType` literal in `example_schemas.py`
2. Update filtering in `__main__.py`
3. Add evaluation logic if needed

**Adding a new eval type**:
1. Add to `Eval.type` literal
2. Implement validation in `Eval.eval_results()`
3. Add tests in `test_evals.py`

**Adding CLI arguments**:
1. Add to `argparse` in `__main__.py:parse_args()`
2. Add to `Args` schema in `schema.py`
3. Handle in filtering/execution logic

### Data Locations
- Training examples: `~/.bananalyzer_data/examples.json` (downloaded) or `static/examples.json` (local)
- Test cache: `.banana_cache/` in working directory
- Example data: `~/.bananalyzer_data/` or `static/`

## Prioritized Improvements

### Critical (Should fix immediately)

1. **Fix deprecated GitHub Actions syntax** [Easy]
   - Update `set-output` to `$GITHUB_OUTPUT` in `.github/workflows/python.yml`
   - Estimated: 15 minutes

2. **Add Playwright installation to CLI help** [Easy]
   - Users frequently miss browser installation step
   - Add to README and CLI error messages
   - Estimated: 30 minutes

### High Priority

3. **Improve error messages for missing examples** [Medium]
   - Current error is cryptic when examples aren't downloaded
   - Add clear instructions to download
   - Estimated: 1 hour

4. **Add type safety for generated test code** [Medium]
   - Test code is generated as strings; could use AST
   - Would catch errors earlier
   - Estimated: 4 hours

5. **Fix circular import pattern** [Medium]
   - Refactor `example_schemas.py` to avoid inline imports
   - Estimated: 2 hours

### Medium Priority

6. **Add integration tests for full pipeline** [Medium]
   - Currently lacks end-to-end tests
   - Should test CLI -> results flow
   - Estimated: 4 hours

7. **Improve S3 error handling** [Easy]
   - Add specific exceptions for auth, network, not found
   - Estimated: 2 hours

8. **Add async batch validation for examples** [Medium]
   - Speed up example loading for large datasets
   - Estimated: 3 hours

9. **Document all CLI arguments in README** [Easy]
   - Some arguments only documented in `--help`
   - Estimated: 1 hour

### Low Priority

10. **Add progress bar for downloads** [Easy]
    - Use tqdm for large S3 downloads
    - Estimated: 1 hour

11. **Support custom schemas directory** [Medium]
    - Allow users to define schemas outside static/
    - Estimated: 2 hours

12. **Add retry logic for flaky tests** [Medium]
    - Some web tests may flake due to timing
    - Estimated: 2 hours

13. **Create example gallery/browser** [High Complexity]
    - Web UI to browse and test examples
    - Estimated: 8+ hours

## CI/CD

- **GitHub Actions**: `.github/workflows/python.yml`
  - Lint (ruff)
  - Type check (mypy)
  - Tests (pytest)
  - Auto-publish to PyPI on version bump

- **Auto-publish**: Compares local version to PyPI; publishes if different

## External Resources

- **PyPI**: https://pypi.org/project/bananalyzer/
- **GitHub**: https://github.com/reworkd/bananalyzer
- **Reworkd**: https://reworkd.ai/
- **Discord**: https://discord.gg/gcmNyAAFfV

## When Making Changes

1. **Follow existing patterns** - Check similar code in the same module
2. **Run type checker** - `poetry run mypy .`
3. **Format code** - `poetry run ruff format .`
4. **Add tests** - Create tests for new functionality
5. **Update examples** - If adding new example types, update `static/examples.json`
6. **Check CI locally** - Run full test suite before pushing

## Common Issues

- **"No examples found"**: Run `bananalyze --download ./agent.py` first
- **Blank MHTML pages**: Run `unix2dos static/*/*.mhtml` to fix line endings
- **Browser not found**: Run `playwright install chromium`
- **Tests bleeding state**: Don't use `--single_browser_instance` for reliability
- **S3 access denied**: Check AWS credentials or use public bucket
- **Type errors**: Ensure Pydantic V2 is installed
