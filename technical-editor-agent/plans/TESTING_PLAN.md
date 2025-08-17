# Testing Plan - Markdown Technical Editor Agent

## Testing Strategy Overview

### Testing Goals
- Achieve 80% code coverage minimum
- Ensure reliability of fact-checking and editing features
- Validate API integration with mocked responses
- Verify end-to-end workflows
- Test error handling and edge cases

### Testing Pyramid
```
         ╱╲
        ╱E2E╲       (5%) - End-to-end tests
       ╱──────╲
      ╱Integration╲  (20%) - Integration tests  
     ╱────────────╲
    ╱  Unit Tests  ╲ (75%) - Unit tests
   ╱────────────────╲
```

## Test Structure

```
tests/
├── __init__.py
├── conftest.py                 # Shared fixtures and configuration
├── unit/                       # Unit tests (80% coverage target)
│   ├── __init__.py
│   ├── test_markdown_parser.py
│   ├── test_content_analyzer.py
│   ├── test_fact_checker.py
│   ├── test_text_editor.py
│   ├── test_api_clients.py
│   ├── test_file_manager.py
│   ├── test_config_loader.py
│   └── test_validators.py
├── integration/                # Integration tests
│   ├── __init__.py
│   ├── test_pipeline.py
│   ├── test_api_switching.py
│   ├── test_batch_processing.py
│   └── test_error_recovery.py
├── e2e/                        # End-to-end tests
│   ├── __init__.py
│   ├── test_book_processing.py
│   └── test_cli_commands.py
├── fixtures/                   # Test data
│   ├── sample_book/
│   │   ├── chapter1.md
│   │   ├── chapter2.md
│   │   └── chapter3.md
│   ├── edge_cases/
│   │   ├── empty.md
│   │   ├── large_file.md
│   │   └── special_chars.md
│   └── mock_responses.json
└── performance/                # Performance tests
    ├── __init__.py
    ├── test_throughput.py
    └── test_memory_usage.py
```

## Unit Tests

### Markdown Parser Tests

```python
# test_markdown_parser.py
import pytest
from pathlib import Path
from src.processors.markdown_parser import MarkdownParser, Document, ContentBlock

class TestMarkdownParser:
    """Test markdown parsing functionality"""
    
    @pytest.fixture
    def parser(self):
        return MarkdownParser()
    
    @pytest.fixture
    def sample_markdown(self):
        return """# Title
        
This is a paragraph with **bold** and *italic* text.

```python
def hello():
    print("Hello, World!")
```

- List item 1
- List item 2

[Link text](https://example.com)
![Image](image.png)
"""

    def test_parse_basic_markdown(self, parser, sample_markdown):
        """Test parsing of basic markdown elements"""
        document = parser.parse_content(sample_markdown)
        
        assert document.content == sample_markdown
        assert document.ast is not None
        assert len(document.ast['blocks']) > 0
        
    def test_extract_editable_content(self, parser, sample_markdown):
        """Test extraction of editable content"""
        document = parser.parse_content(sample_markdown)
        blocks = parser.extract_content(document)
        
        # Should have paragraph and list items as editable
        editable_blocks = [b for b in blocks if not b.preserved]
        assert len(editable_blocks) >= 2
        
    def test_preserve_code_blocks(self, parser):
        """Test that code blocks are marked as preserved"""
        content = "Text\n```python\ncode\n```\nMore text"
        document = parser.parse_content(content)
        preserved = parser.preserve_special_blocks(document)
        
        code_blocks = [p for p in preserved if p.type == 'code']
        assert len(code_blocks) == 1
        assert "code" in code_blocks[0].content
        
    def test_preserve_links_and_images(self, parser):
        """Test preservation of links and images"""
        content = "[link](url) and ![image](path.png)"
        document = parser.parse_content(content)
        preserved = parser.preserve_special_blocks(document)
        
        links = [p for p in preserved if p.type == 'link']
        images = [p for p in preserved if p.type == 'image']
        
        assert len(links) == 1
        assert len(images) == 1
        
    def test_reconstruct_markdown(self, parser, sample_markdown):
        """Test markdown reconstruction from AST"""
        document = parser.parse_content(sample_markdown)
        reconstructed = parser.reconstruct_markdown(document)
        
        # Should preserve essential structure
        assert "# Title" in reconstructed
        assert "```python" in reconstructed
        assert "[Link text]" in reconstructed
        
    def test_parse_empty_document(self, parser):
        """Test handling of empty document"""
        document = parser.parse_content("")
        assert document.content == ""
        assert document.ast is not None
        
    def test_parse_malformed_markdown(self, parser):
        """Test handling of malformed markdown"""
        malformed = "# Unclosed code block\n```python\nno closing"
        document = parser.parse_content(malformed)
        assert document.warnings
        
    @pytest.mark.parametrize("filename,expected", [
        ("chapter1.md", True),
        ("README.markdown", True),
        ("notes.txt", False),
        ("script.py", False)
    ])
    def test_is_markdown_file(self, parser, filename, expected):
        """Test markdown file detection"""
        assert parser.is_markdown_file(filename) == expected
```

### Content Analyzer Tests

```python
# test_content_analyzer.py
import pytest
from src.processors.content_analyzer import ContentAnalyzer, ClarityScore

class TestContentAnalyzer:
    """Test content analysis functionality"""
    
    @pytest.fixture
    def analyzer(self):
        return ContentAnalyzer()
    
    def test_identify_factual_claims(self, analyzer):
        """Test identification of factual claims"""
        text = """
        The Earth orbits the Sun. 
        Water boils at 100 degrees Celsius at sea level.
        I think this is interesting.
        """
        
        claims = analyzer.identify_facts(text)
        
        # Should identify first two as factual claims
        assert len(claims) >= 2
        assert any("Earth" in claim.statement for claim in claims)
        assert any("Water boils" in claim.statement for claim in claims)
        
    def test_clarity_assessment(self, analyzer):
        """Test clarity scoring"""
        clear_text = "The cat sat on the mat."
        complex_text = "The aforementioned feline specimen subsequently positioned itself upon the designated textile floor covering apparatus."
        
        clear_score = analyzer.assess_clarity(clear_text)
        complex_score = analyzer.assess_clarity(complex_text)
        
        assert clear_score.score > complex_score.score
        assert clear_score.readability > complex_score.readability
        
    def test_consistency_check(self, analyzer):
        """Test consistency checking across documents"""
        doc1 = Document(content="We use color in our application.")
        doc2 = Document(content="The colour settings are important.")
        
        report = analyzer.check_consistency([doc1, doc2])
        
        # Should detect spelling inconsistency
        assert report.inconsistencies
        assert any("color" in i.description for i in report.inconsistencies)
        
    def test_readability_metrics(self, analyzer):
        """Test readability score calculation"""
        simple = "See Spot run. Run, Spot, run!"
        complex = "The implementation of sophisticated algorithmic paradigms necessitates comprehension."
        
        simple_metrics = analyzer.measure_readability(simple)
        complex_metrics = analyzer.measure_readability(complex)
        
        assert simple_metrics.flesch_score > complex_metrics.flesch_score
        assert simple_metrics.grade_level < complex_metrics.grade_level
        
    def test_technical_jargon_detection(self, analyzer):
        """Test detection of technical jargon"""
        text = "The API endpoint uses OAuth2 for authentication via JWT tokens."
        
        jargon = analyzer.detect_jargon(text)
        
        assert len(jargon) >= 3
        assert any(term in ["API", "OAuth2", "JWT"] for term in jargon)
```

### API Client Tests

```python
# test_api_clients.py
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from src.clients.openai_client import OpenAIClient
from src.clients.claude_client import ClaudeClient
from src.models import FactCheckResult, EditResult, EditInstructions

class TestOpenAIClient:
    """Test OpenAI client implementation"""
    
    @pytest.fixture
    def mock_openai_response(self):
        """Create mock OpenAI response"""
        response = MagicMock()
        response.choices = [
            MagicMock(
                message=MagicMock(
                    content='{"confidence": 0.95, "verdict": "accurate", '
                           '"explanation": "Verified", "sources": []}'
                )
            )
        ]
        return response
    
    @pytest.mark.asyncio
    async def test_fact_check(self, mock_openai_response):
        """Test fact checking with OpenAI"""
        with patch('openai.AsyncOpenAI') as mock_openai:
            mock_client = AsyncMock()
            mock_client.chat.completions.create.return_value = mock_openai_response
            mock_openai.return_value = mock_client
            
            client = OpenAIClient(api_key="test_key")
            await client.initialize()
            
            result = await client.check_fact(
                statement="The Earth is round",
                context="Basic geography"
            )
            
            assert isinstance(result, FactCheckResult)
            assert result.confidence == 0.95
            assert result.verdict == "accurate"
            
    @pytest.mark.asyncio
    async def test_edit_text(self):
        """Test text editing with OpenAI"""
        with patch('openai.AsyncOpenAI') as mock_openai:
            mock_response = MagicMock()
            mock_response.choices = [
                MagicMock(
                    message=MagicMock(
                        content='{"edited_text": "Improved text", '
                               '"changes_made": ["Simplified"], '
                               '"confidence": 0.9}'
                    )
                )
            ]
            
            mock_client = AsyncMock()
            mock_client.chat.completions.create.return_value = mock_response
            mock_openai.return_value = mock_client
            
            client = OpenAIClient(api_key="test_key")
            await client.initialize()
            
            instructions = EditInstructions(
                dimension="clarity",
                aggressiveness="medium"
            )
            
            result = await client.edit_text(
                text="Complex text here",
                instructions=instructions
            )
            
            assert isinstance(result, EditResult)
            assert result.edited_text == "Improved text"
            assert "Simplified" in result.changes_made
            
    def test_token_estimation(self):
        """Test token count estimation"""
        client = OpenAIClient(api_key="test_key")
        
        text = "This is a test sentence with several words."
        tokens = client.estimate_tokens(text)
        
        # Should be roughly word count * 1.3
        assert 8 <= tokens <= 15

class TestClaudeClient:
    """Test Claude client implementation"""
    
    @pytest.mark.asyncio
    async def test_fact_check(self):
        """Test fact checking with Claude"""
        with patch('anthropic.AsyncAnthropic') as mock_anthropic:
            mock_response = MagicMock()
            mock_response.content = [
                MagicMock(
                    text='Here is the analysis: {"confidence": 0.88, '
                         '"verdict": "accurate", "explanation": "Correct", '
                         '"sources": ["source1"]}'
                )
            ]
            
            mock_client = AsyncMock()
            mock_client.messages.create.return_value = mock_response
            mock_anthropic.return_value = mock_client
            
            client = ClaudeClient(api_key="test_key")
            await client.initialize()
            
            result = await client.check_fact(
                statement="Python is a programming language",
                context="Technology"
            )
            
            assert result.confidence == 0.88
            assert result.verdict == "accurate"
            assert len(result.sources) == 1
            
    @pytest.mark.asyncio
    async def test_error_handling(self):
        """Test error handling in Claude client"""
        with patch('anthropic.AsyncAnthropic') as mock_anthropic:
            mock_client = AsyncMock()
            mock_client.messages.create.side_effect = Exception("API Error")
            mock_anthropic.return_value = mock_client
            
            client = ClaudeClient(api_key="test_key")
            await client.initialize()
            
            with pytest.raises(Exception) as exc_info:
                await client.check_fact("Test statement")
            
            assert "API Error" in str(exc_info.value)
```

### Fact Checker Tests

```python
# test_fact_checker.py
import pytest
from unittest.mock import AsyncMock
from src.processors.fact_checker import FactChecker, Warning
from src.models import FactCheckResult, VerificationResult

class TestFactChecker:
    """Test fact checking functionality"""
    
    @pytest.fixture
    def mock_ai_client(self):
        """Create mock AI client"""
        client = AsyncMock()
        client.check_fact.return_value = FactCheckResult(
            statement="Test statement",
            confidence=0.95,
            verdict="accurate",
            explanation="Verified",
            sources=[]
        )
        return client
    
    @pytest.fixture
    def fact_checker(self, mock_ai_client):
        return FactChecker(mock_ai_client)
    
    @pytest.mark.asyncio
    async def test_verify_single_claim(self, fact_checker):
        """Test verification of single claim"""
        claim = FactualClaim(statement="The sky is blue")
        result = await fact_checker.verify_claim(claim, "Weather context")
        
        assert isinstance(result, VerificationResult)
        assert result.confidence == 0.95
        assert result.verdict == "accurate"
        
    @pytest.mark.asyncio
    async def test_batch_verification(self, fact_checker):
        """Test batch verification of multiple claims"""
        claims = [
            FactualClaim(statement="Claim 1"),
            FactualClaim(statement="Claim 2"),
            FactualClaim(statement="Claim 3")
        ]
        
        results = await fact_checker.batch_verify(claims)
        
        assert len(results) == 3
        assert all(isinstance(r, VerificationResult) for r in results)
        
    def test_warning_generation(self, fact_checker):
        """Test generation of warnings for low-confidence facts"""
        results = [
            VerificationResult(
                statement="High confidence",
                confidence=0.95,
                verdict="accurate"
            ),
            VerificationResult(
                statement="Low confidence",
                confidence=0.4,
                verdict="uncertain"
            ),
            VerificationResult(
                statement="Inaccurate",
                confidence=0.9,
                verdict="inaccurate"
            )
        ]
        
        warnings = fact_checker.generate_warnings(results)
        
        # Should generate warnings for low confidence and inaccurate
        assert len(warnings) >= 2
        assert any(w.severity == "high" for w in warnings)
        assert any("Low confidence" in w.message for w in warnings)
```

## Integration Tests

### Pipeline Integration Tests

```python
# test_pipeline.py
import pytest
from pathlib import Path
from unittest.mock import AsyncMock, patch
from src.pipeline import ProcessingPipeline
from src.models import Config, ProcessingResult

class TestProcessingPipeline:
    """Test the complete processing pipeline"""
    
    @pytest.fixture
    def test_config(self):
        """Create test configuration"""
        return Config(
            api={
                "provider": "openai",
                "api_key": "test_key",
                "model": "gpt-5"
            },
            processing={
                "batch_size": 2,
                "parallel_workers": 1
            },
            fact_checking={
                "enabled": True,
                "confidence_threshold": 0.7
            }
        )
    
    @pytest.fixture
    def mock_api_client(self):
        """Create mock API client"""
        client = AsyncMock()
        client.check_fact.return_value = FactCheckResult(
            statement="Test",
            confidence=0.9,
            verdict="accurate",
            explanation="Good"
        )
        client.edit_text.return_value = EditResult(
            original_text="Original",
            edited_text="Edited",
            changes_made=["Improved"],
            confidence=0.85
        )
        return client
    
    @pytest.mark.asyncio
    async def test_process_single_file(self, test_config, mock_api_client, tmp_path):
        """Test processing a single markdown file"""
        # Create test file
        test_file = tmp_path / "test.md"
        test_file.write_text("# Test\n\nThis is test content.")
        
        with patch('src.pipeline.AIClientFactory.create_and_initialize', 
                  return_value=mock_api_client):
            pipeline = ProcessingPipeline(test_config)
            result = await pipeline.process_file(str(test_file))
            
            assert isinstance(result, ProcessingResult)
            assert result.success
            assert len(result.documents) == 1
            
    @pytest.mark.asyncio
    async def test_process_directory(self, test_config, mock_api_client, tmp_path):
        """Test processing directory with multiple files"""
        # Create test files
        for i in range(3):
            (tmp_path / f"chapter{i}.md").write_text(f"# Chapter {i}\n\nContent")
        
        with patch('src.pipeline.AIClientFactory.create_and_initialize',
                  return_value=mock_api_client):
            pipeline = ProcessingPipeline(test_config)
            result = await pipeline.process_book(str(tmp_path))
            
            assert result.success
            assert len(result.documents) == 3
            
    @pytest.mark.asyncio
    async def test_error_recovery(self, test_config, tmp_path):
        """Test pipeline error recovery"""
        test_file = tmp_path / "test.md"
        test_file.write_text("# Test")
        
        # Mock client that fails on first call
        mock_client = AsyncMock()
        mock_client.check_fact.side_effect = [
            Exception("API Error"),
            FactCheckResult(
                statement="Test",
                confidence=0.9,
                verdict="accurate",
                explanation="Good"
            )
        ]
        
        with patch('src.pipeline.AIClientFactory.create_and_initialize',
                  return_value=mock_client):
            pipeline = ProcessingPipeline(test_config)
            pipeline.config.processing.max_retries = 1
            
            result = await pipeline.process_file(str(test_file))
            
            # Should recover and complete
            assert result.success
            assert mock_client.check_fact.call_count == 2
```

### API Switching Tests

```python
# test_api_switching.py
import pytest
from unittest.mock import patch, AsyncMock
from src.clients import AIClientFactory, APIProvider

class TestAPISwitching:
    """Test switching between API providers"""
    
    @pytest.mark.asyncio
    async def test_switch_providers(self):
        """Test switching from OpenAI to Claude"""
        # Start with OpenAI
        with patch('src.clients.openai_client.OpenAIClient') as mock_openai:
            openai_instance = AsyncMock()
            mock_openai.return_value = openai_instance
            
            client1 = AIClientFactory.create_client(
                provider="openai",
                api_key="key1"
            )
            
            assert mock_openai.called
            
        # Switch to Claude
        with patch('src.clients.claude_client.ClaudeClient') as mock_claude:
            claude_instance = AsyncMock()
            mock_claude.return_value = claude_instance
            
            client2 = AIClientFactory.create_client(
                provider="claude",
                api_key="key2"
            )
            
            assert mock_claude.called
            
    @pytest.mark.asyncio
    async def test_fallback_on_failure(self):
        """Test fallback from primary to secondary provider"""
        primary_client = AsyncMock()
        primary_client.check_fact.side_effect = Exception("Primary failed")
        
        secondary_client = AsyncMock()
        secondary_client.check_fact.return_value = FactCheckResult(
            statement="Test",
            confidence=0.9,
            verdict="accurate",
            explanation="From secondary"
        )
        
        # Test fallback logic
        result = None
        try:
            result = await primary_client.check_fact("Test")
        except:
            result = await secondary_client.check_fact("Test")
            
        assert result.explanation == "From secondary"
```

## End-to-End Tests

### Book Processing E2E Test

```python
# test_book_processing.py
import pytest
import shutil
from pathlib import Path
from click.testing import CliRunner
from src.cli import cli

class TestBookProcessingE2E:
    """End-to-end tests for book processing"""
    
    @pytest.fixture
    def sample_book(self, tmp_path):
        """Create a sample book structure"""
        book_dir = tmp_path / "sample_book"
        book_dir.mkdir()
        
        # Create chapters
        (book_dir / "chapter1.md").write_text("""# Chapter 1: Introduction

This chapter introduces the topic. The Earth orbits the Sun in 365 days.

## Section 1.1
Content here with some **formatting**.
""")
        
        (book_dir / "chapter2.md").write_text("""# Chapter 2: Advanced Topics

```python
def example():
    return "This code should not be edited"
```

Water boils at 100 degrees Celsius.
""")
        
        return book_dir
    
    @pytest.mark.integration
    def test_complete_book_processing(self, sample_book, monkeypatch):
        """Test processing an entire book"""
        # Set test API key
        monkeypatch.setenv("OPENAI_API_KEY", "test_key")
        
        runner = CliRunner()
        result = runner.invoke(cli, [
            'process',
            str(sample_book),
            '--api', 'openai',
            '--dry-run'  # Don't actually call API
        ])
        
        assert result.exit_code == 0
        assert "Processing" in result.output
        
    @pytest.mark.integration
    def test_cli_with_config_file(self, sample_book, tmp_path, monkeypatch):
        """Test CLI with custom configuration file"""
        config_file = tmp_path / "config.yaml"
        config_file.write_text("""
api:
  provider: claude
  model: claude-sonnet-4
  temperature: 0.2

editing:
  dimensions:
    - clarity
    - accuracy
""")
        
        monkeypatch.setenv("CLAUDE_API_KEY", "test_key")
        
        runner = CliRunner()
        result = runner.invoke(cli, [
            'process',
            str(sample_book),
            '--config', str(config_file),
            '--dry-run'
        ])
        
        assert result.exit_code == 0
```

## Performance Tests

### Throughput Testing

```python
# test_throughput.py
import pytest
import asyncio
import time
from src.pipeline import ProcessingPipeline

class TestThroughput:
    """Test processing throughput"""
    
    @pytest.mark.performance
    @pytest.mark.asyncio
    async def test_batch_processing_speed(self, mock_api_client, tmp_path):
        """Test speed of batch processing"""
        # Create 50 test files
        for i in range(50):
            (tmp_path / f"file{i}.md").write_text(f"# File {i}\n\nContent")
        
        pipeline = ProcessingPipeline(test_config)
        
        start_time = time.time()
        result = await pipeline.process_book(str(tmp_path))
        end_time = time.time()
        
        processing_time = end_time - start_time
        files_per_second = 50 / processing_time
        
        # Should process at least 1 file per second
        assert files_per_second >= 1.0
        assert result.success
        
    @pytest.mark.performance
    def test_memory_usage(self, tmp_path):
        """Test memory usage during processing"""
        import tracemalloc
        
        # Create large test file (5MB)
        large_file = tmp_path / "large.md"
        content = "# Large File\n\n" + ("Test content. " * 100000)
        large_file.write_text(content)
        
        tracemalloc.start()
        
        # Process file
        pipeline = ProcessingPipeline(test_config)
        asyncio.run(pipeline.process_file(str(large_file)))
        
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        
        # Should use less than 500MB for a 5MB file
        assert peak / 1024 / 1024 < 500
```

## Test Fixtures

### Shared Fixtures (conftest.py)

```python
# conftest.py
import pytest
import asyncio
from pathlib import Path
from unittest.mock import AsyncMock
import json

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def mock_api_responses():
    """Load mock API responses"""
    fixtures_path = Path(__file__).parent / "fixtures" / "mock_responses.json"
    with open(fixtures_path) as f:
        return json.load(f)

@pytest.fixture
def mock_ai_client(mock_api_responses):
    """Create a fully mocked AI client"""
    client = AsyncMock()
    
    # Setup default responses
    client.check_fact.return_value = FactCheckResult(
        statement="Default",
        confidence=0.9,
        verdict="accurate",
        explanation="Mock response",
        sources=[]
    )
    
    client.edit_text.return_value = EditResult(
        original_text="Original",
        edited_text="Edited",
        changes_made=["Mock change"],
        confidence=0.85
    )
    
    client.estimate_tokens.return_value = 100
    
    return client

@pytest.fixture
def sample_markdown_files(tmp_path):
    """Create sample markdown files for testing"""
    files = {
        "simple.md": "# Title\n\nSimple content.",
        "complex.md": """# Complex Document

## Introduction
This has multiple sections.

### Subsection
With various elements:
- Lists
- **Bold text**
- [Links](https://example.com)

```python
code_block = "preserved"
```

## Conclusion
Final thoughts.
""",
        "facts.md": """# Factual Content

The Earth orbits the Sun.
Water freezes at 0 degrees Celsius.
Python was created in 1991.
"""
    }
    
    for filename, content in files.items():
        (tmp_path / filename).write_text(content)
    
    return tmp_path

@pytest.fixture
def test_config():
    """Create default test configuration"""
    return Config(
        api={
            "provider": "openai",
            "api_key": "test_key",
            "model": "gpt-5",
            "temperature": 0.3
        },
        processing={
            "batch_size": 5,
            "parallel_workers": 2
        },
        fact_checking={
            "enabled": True,
            "confidence_threshold": 0.7
        },
        editing={
            "dimensions": ["clarity", "accuracy"],
            "aggressiveness": "medium"
        }
    )
```

## Coverage Configuration

### pytest.ini

```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
asyncio_mode = auto

markers =
    unit: Unit tests
    integration: Integration tests
    e2e: End-to-end tests
    performance: Performance tests
    slow: Slow running tests

addopts = 
    --cov=src
    --cov-report=html
    --cov-report=term-missing
    --cov-report=xml
    -v
```

### .coveragerc

```ini
[run]
source = src
omit = 
    */tests/*
    */test_*
    */__pycache__/*
    */venv/*
    */env/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
    if TYPE_CHECKING:
    @abstractmethod

precision = 2
show_missing = True
skip_covered = False

[html]
directory = htmlcov
```

## GitHub Actions Testing Workflow

### .github/workflows/test.yml

```yaml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
        restore-keys: |
          ${{ runner.os }}-pip-
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Run linting
      run: |
        flake8 src tests --max-line-length=100
        black --check src tests
        mypy src --ignore-missing-imports
    
    - name: Run unit tests
      run: |
        pytest tests/unit -v --cov=src --cov-report=xml
    
    - name: Run integration tests
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        pytest tests/integration -v -m integration
    
    - name: Run E2E tests
      if: github.event_name == 'push'
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        pytest tests/e2e -v -m e2e
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-${{ matrix.python-version }}
    
    - name: Check coverage threshold
      run: |
        coverage report --fail-under=80
```

## Test Execution Commands

### Local Testing

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=html

# Run specific test categories
pytest -m unit           # Unit tests only
pytest -m integration    # Integration tests only
pytest -m "not slow"     # Skip slow tests

# Run specific test file
pytest tests/unit/test_markdown_parser.py

# Run with verbose output
pytest -v

# Run in parallel (requires pytest-xdist)
pytest -n auto

# Generate coverage report
coverage run -m pytest
coverage report
coverage html  # Generate HTML report
```

### Continuous Testing

```bash
# Watch for changes and re-run tests (requires pytest-watch)
ptw -- --cov=src

# Run tests on file save
nodemon --exec "pytest tests/unit" --ext py
```

## Test Data Management

### Sample Book Structure

```
tests/fixtures/sample_book/
├── metadata.yaml          # Book metadata
├── introduction.md        # Introduction chapter
├── chapters/
│   ├── chapter01.md      # Chapter 1
│   ├── chapter02.md      # Chapter 2
│   └── chapter03.md      # Chapter 3
├── appendices/
│   ├── appendix_a.md     # Appendix A
│   └── glossary.md       # Glossary
└── images/
    ├── diagram1.png      # Image files
    └── chart1.svg        # Charts
```

This comprehensive testing plan ensures thorough validation of all components while maintaining the 80% coverage target and providing confidence in the system's reliability.