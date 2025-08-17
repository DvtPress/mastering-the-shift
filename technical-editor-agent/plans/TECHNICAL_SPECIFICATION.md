# Technical Specification - Markdown Technical Editor Agent

## 1. System Components

### 1.1 Core Modules

#### Markdown Parser Module
```python
class MarkdownParser:
    """Parses markdown files into AST for manipulation"""
    
    def parse_file(self, filepath: str) -> Document:
        """Parse a single markdown file"""
        
    def parse_directory(self, dirpath: str) -> List[Document]:
        """Parse all markdown files in directory"""
        
    def extract_content(self, document: Document) -> ContentBlocks:
        """Extract editable content from document"""
        
    def preserve_special_blocks(self, document: Document) -> PreservedContent:
        """Mark code blocks, links, images for preservation"""
        
    def reconstruct_markdown(self, document: Document) -> str:
        """Convert modified AST back to markdown"""
```

#### Content Analyzer Module
```python
class ContentAnalyzer:
    """Analyzes content for editing needs"""
    
    def identify_facts(self, text: str) -> List[FactualClaim]:
        """Extract factual claims from text"""
        
    def assess_clarity(self, text: str) -> ClarityScore:
        """Calculate clarity metrics"""
        
    def check_consistency(self, documents: List[Document]) -> ConsistencyReport:
        """Check consistency across documents"""
        
    def measure_readability(self, text: str) -> ReadabilityMetrics:
        """Calculate readability scores"""
```

#### Fact Checker Module
```python
class FactChecker:
    """Verifies factual accuracy of claims"""
    
    def __init__(self, ai_client: AIClient):
        self.client = ai_client
    
    async def verify_claim(self, claim: FactualClaim, context: str) -> VerificationResult:
        """Verify a single factual claim"""
        
    async def batch_verify(self, claims: List[FactualClaim]) -> List[VerificationResult]:
        """Verify multiple claims efficiently"""
        
    def generate_warnings(self, results: List[VerificationResult]) -> List[Warning]:
        """Create warnings for low-confidence facts"""
```

#### Text Editor Module
```python
class TextEditor:
    """Performs text editing operations"""
    
    def __init__(self, ai_client: AIClient):
        self.client = ai_client
    
    async def edit_for_clarity(self, text: str) -> EditResult:
        """Improve text clarity"""
        
    async def edit_for_accuracy(self, text: str, facts: List[VerificationResult]) -> EditResult:
        """Correct factual errors"""
        
    async def edit_for_consistency(self, text: str, style_guide: StyleGuide) -> EditResult:
        """Ensure consistent style"""
        
    async def reorganize_content(self, blocks: List[ContentBlock]) -> List[ContentBlock]:
        """Reorganize content for better flow"""
```

### 1.2 API Clients

#### Base Client Interface
```python
from abc import ABC, abstractmethod

class AIClient(ABC):
    """Abstract base class for AI clients"""
    
    @abstractmethod
    async def check_fact(self, statement: str, context: str) -> FactCheckResult:
        """Verify a factual statement"""
        
    @abstractmethod
    async def edit_text(self, text: str, instructions: EditInstructions) -> EditResult:
        """Edit text based on instructions"""
        
    @abstractmethod
    async def analyze_content(self, content: str) -> ContentAnalysis:
        """Analyze content characteristics"""
```

#### OpenAI Client Implementation
```python
class OpenAIClient(AIClient):
    """OpenAI API client implementation"""
    
    def __init__(self, api_key: str, model: str = "gpt-5"):
        self.client = openai.AsyncOpenAI(api_key=api_key)
        self.model = model
        
    async def check_fact(self, statement: str, context: str) -> FactCheckResult:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": FACT_CHECK_PROMPT},
                {"role": "user", "content": f"Statement: {statement}\nContext: {context}"}
            ],
            temperature=0.3
        )
        return self._parse_fact_check_response(response)
```

#### Claude Client Implementation
```python
class ClaudeClient(AIClient):
    """Claude API client implementation"""
    
    def __init__(self, api_key: str, model: str = "claude-sonnet-4"):
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        self.model = model
        
    async def check_fact(self, statement: str, context: str) -> FactCheckResult:
        response = await self.client.messages.create(
            model=self.model,
            messages=[
                {"role": "user", "content": f"{FACT_CHECK_PROMPT}\n\nStatement: {statement}\nContext: {context}"}
            ],
            temperature=0.3,
            max_tokens=4000
        )
        return self._parse_fact_check_response(response)
```

### 1.3 Data Models

#### Document Model
```python
from pydantic import BaseModel
from typing import List, Optional

class Document(BaseModel):
    """Represents a markdown document"""
    filepath: str
    filename: str
    content: str
    ast: Optional[dict] = None
    metadata: dict = {}
    warnings: List[Warning] = []
    
class ContentBlock(BaseModel):
    """A block of content within a document"""
    type: str  # paragraph, heading, list, etc.
    content: str
    position: int
    preserved: bool = False
    
class PreservedContent(BaseModel):
    """Content that should not be modified"""
    type: str  # code, link, image
    content: str
    start_pos: int
    end_pos: int
```

#### Result Models
```python
class FactCheckResult(BaseModel):
    """Result of fact-checking operation"""
    statement: str
    confidence: float  # 0.0 to 1.0
    verdict: str  # accurate, inaccurate, uncertain
    explanation: str
    sources: List[str] = []
    
class EditResult(BaseModel):
    """Result of text editing operation"""
    original_text: str
    edited_text: str
    changes: List[TextChange]
    edit_type: str  # clarity, accuracy, consistency, usability
    
class Warning(BaseModel):
    """Warning generated during processing"""
    severity: str  # high, medium, low
    type: str  # fact_check, consistency, structure
    message: str
    location: str
    suggestion: Optional[str] = None
```

## 2. Processing Pipeline

### 2.1 Pipeline Architecture

```python
class ProcessingPipeline:
    """Main processing pipeline orchestrator"""
    
    def __init__(self, config: Config):
        self.parser = MarkdownParser()
        self.analyzer = ContentAnalyzer()
        self.ai_client = self._create_client(config)
        self.fact_checker = FactChecker(self.ai_client)
        self.editor = TextEditor(self.ai_client)
        
    async def process_book(self, input_dir: str) -> ProcessingResult:
        """Process entire book directory"""
        # Step 1: Parse all markdown files
        documents = self.parser.parse_directory(input_dir)
        
        # Step 2: Analyze content
        for doc in documents:
            doc.analysis = self.analyzer.analyze_document(doc)
        
        # Step 3: Fact-checking
        if self.config.fact_checking.enabled:
            await self._fact_check_documents(documents)
        
        # Step 4: Text editing
        await self._edit_documents(documents)
        
        # Step 5: Output generation
        self._write_output(documents)
        
        return ProcessingResult(documents=documents)
```

### 2.2 Error Handling

```python
class ErrorHandler:
    """Centralized error handling"""
    
    @staticmethod
    def handle_api_error(error: Exception, operation: str) -> None:
        """Handle API-related errors"""
        if isinstance(error, RateLimitError):
            logger.warning(f"Rate limit hit during {operation}, backing off...")
            raise RetryableError(error)
        elif isinstance(error, APIConnectionError):
            logger.error(f"API connection failed during {operation}")
            raise
        
    @staticmethod
    def handle_parsing_error(error: Exception, filepath: str) -> Document:
        """Handle markdown parsing errors"""
        logger.error(f"Failed to parse {filepath}: {error}")
        return Document(
            filepath=filepath,
            content="",
            warnings=[Warning(
                severity="high",
                type="parsing",
                message=str(error),
                location=filepath
            )]
        )
```

## 3. Configuration Management

### 3.1 Configuration Schema

```yaml
# config.yaml
api:
  provider: openai  # openai or claude
  model: gpt-5  # gpt-5 or claude-sonnet-4
  api_key_env: OPENAI_API_KEY
  temperature: 0.3
  max_tokens: 4000
  timeout: 30
  max_retries: 3
  
processing:
  batch_size: 5
  parallel_workers: 3
  preserve_structure: true
  backup_originals: true
  
fact_checking:
  enabled: true
  confidence_threshold: 0.7
  verify_statistics: true
  verify_citations: true
  
editing:
  dimensions:
    clarity:
      enabled: true
      aggressiveness: medium  # low, medium, high
    accuracy:
      enabled: true
      use_fact_checks: true
    consistency:
      enabled: true
      style_guide: default
    usability:
      enabled: true
      target_readability: 60  # Flesch score
      
output:
  overwrite_originals: true
  create_backup: true
  backup_suffix: .backup
  warnings_file: warnings.json
  summary_report: true
  
logging:
  level: INFO
  file: editor.log
  console: true
  format: "[{time}] {level} - {message}"
```

### 3.2 Configuration Loader

```python
class ConfigLoader:
    """Loads and validates configuration"""
    
    @staticmethod
    def load_config(config_path: Optional[str] = None) -> Config:
        """Load configuration from file or defaults"""
        if config_path:
            with open(config_path, 'r') as f:
                config_dict = yaml.safe_load(f)
        else:
            config_dict = DEFAULT_CONFIG
        
        # Override with environment variables
        config_dict = ConfigLoader._apply_env_overrides(config_dict)
        
        # Validate and return
        return Config(**config_dict)
    
    @staticmethod
    def _apply_env_overrides(config: dict) -> dict:
        """Apply environment variable overrides"""
        if api_key := os.getenv(config['api']['api_key_env']):
            config['api']['api_key'] = api_key
        return config
```

## 4. CLI Implementation

### 4.1 Command Structure

```python
import click
from pathlib import Path

@click.group()
@click.version_option(version='1.0.0')
def cli():
    """Markdown Technical Editor Agent - AI-powered book editor"""
    pass

@cli.command()
@click.argument('input_dir', type=click.Path(exists=True))
@click.option('--api', type=click.Choice(['openai', 'claude']), default='openai')
@click.option('--config', type=click.Path(exists=True), help='Configuration file')
@click.option('--dry-run', is_flag=True, help='Preview changes without writing')
@click.option('--verbose', is_flag=True, help='Verbose output')
@click.option('--no-backup', is_flag=True, help='Skip creating backups')
def process(input_dir, api, config, dry_run, verbose, no_backup):
    """Process markdown files in INPUT_DIR"""
    # Setup logging
    setup_logging(verbose)
    
    # Load configuration
    cfg = ConfigLoader.load_config(config)
    cfg.api.provider = api
    cfg.output.create_backup = not no_backup
    
    # Initialize pipeline
    pipeline = ProcessingPipeline(cfg)
    
    # Process with progress bar
    with click.progressbar(length=100, label='Processing') as bar:
        result = asyncio.run(pipeline.process_book(input_dir))
        bar.update(100)
    
    # Display results
    display_results(result, verbose)
```

## 5. Testing Implementation

### 5.1 Unit Test Examples

```python
# test_markdown_parser.py
import pytest
from src.processors.markdown_parser import MarkdownParser

class TestMarkdownParser:
    
    @pytest.fixture
    def parser(self):
        return MarkdownParser()
    
    def test_parse_simple_markdown(self, parser, tmp_path):
        # Create test file
        test_file = tmp_path / "test.md"
        test_file.write_text("# Heading\n\nParagraph text.")
        
        # Parse
        doc = parser.parse_file(str(test_file))
        
        # Assertions
        assert doc.filename == "test.md"
        assert "# Heading" in doc.content
        assert len(doc.ast) > 0
    
    def test_preserve_code_blocks(self, parser):
        content = "Text\n```python\ncode here\n```\nMore text"
        doc = parser.parse_content(content)
        preserved = parser.preserve_special_blocks(doc)
        
        assert len(preserved) == 1
        assert preserved[0].type == "code"
        assert "code here" in preserved[0].content
```

### 5.2 Integration Test Examples

```python
# test_end_to_end.py
import pytest
from unittest.mock import Mock, AsyncMock

class TestEndToEnd:
    
    @pytest.mark.asyncio
    async def test_complete_processing(self, tmp_path, mock_ai_client):
        # Setup test data
        input_dir = tmp_path / "input"
        input_dir.mkdir()
        (input_dir / "chapter1.md").write_text("# Chapter 1\n\nThe sun orbits the Earth.")
        
        # Mock AI responses
        mock_ai_client.check_fact.return_value = FactCheckResult(
            statement="The sun orbits the Earth",
            confidence=0.1,
            verdict="inaccurate",
            explanation="The Earth orbits the sun"
        )
        
        # Process
        pipeline = ProcessingPipeline(test_config)
        result = await pipeline.process_book(str(input_dir))
        
        # Verify
        assert len(result.documents) == 1
        assert len(result.documents[0].warnings) > 0
        assert "Earth orbits the sun" in result.documents[0].content
```

## 6. Performance Optimization

### 6.1 Batch Processing

```python
class BatchProcessor:
    """Handles batch processing for efficiency"""
    
    def __init__(self, batch_size: int = 5):
        self.batch_size = batch_size
        
    async def process_batch(self, items: List, processor: Callable) -> List:
        """Process items in batches"""
        results = []
        for i in range(0, len(items), self.batch_size):
            batch = items[i:i + self.batch_size]
            batch_results = await asyncio.gather(
                *[processor(item) for item in batch]
            )
            results.extend(batch_results)
        return results
```

### 6.2 Caching

```python
class ResponseCache:
    """Caches API responses to avoid duplicate calls"""
    
    def __init__(self, ttl: int = 3600):
        self.cache = {}
        self.ttl = ttl
        
    def get(self, key: str) -> Optional[Any]:
        """Get cached response"""
        if key in self.cache:
            entry = self.cache[key]
            if time.time() - entry['timestamp'] < self.ttl:
                return entry['value']
        return None
        
    def set(self, key: str, value: Any) -> None:
        """Cache a response"""
        self.cache[key] = {
            'value': value,
            'timestamp': time.time()
        }
```

## 7. Monitoring and Logging

### 7.1 Logging Configuration

```python
from loguru import logger

def setup_logging(verbose: bool = False):
    """Configure application logging"""
    
    # Remove default handler
    logger.remove()
    
    # Console handler
    level = "DEBUG" if verbose else "INFO"
    logger.add(
        sys.stderr,
        format="<green>{time:HH:mm:ss}</green> | <level>{level: <8}</level> | {message}",
        level=level
    )
    
    # File handler
    logger.add(
        "editor.log",
        rotation="10 MB",
        retention="7 days",
        format="{time} | {level} | {name}:{function}:{line} | {message}",
        level="DEBUG"
    )
```

### 7.2 Performance Metrics

```python
class MetricsCollector:
    """Collects performance metrics"""
    
    def __init__(self):
        self.metrics = {
            'files_processed': 0,
            'api_calls': 0,
            'api_tokens': 0,
            'processing_time': 0,
            'errors': 0
        }
        
    def record_api_call(self, tokens: int, duration: float):
        """Record API call metrics"""
        self.metrics['api_calls'] += 1
        self.metrics['api_tokens'] += tokens
        self.metrics['processing_time'] += duration
        
    def get_summary(self) -> dict:
        """Get metrics summary"""
        return {
            **self.metrics,
            'avg_tokens_per_call': self.metrics['api_tokens'] / max(self.metrics['api_calls'], 1),
            'avg_processing_time': self.metrics['processing_time'] / max(self.metrics['files_processed'], 1)
        }
```

## 8. Security Considerations

### 8.1 API Key Management

```python
class SecureConfig:
    """Secure configuration management"""
    
    @staticmethod
    def get_api_key(provider: str) -> str:
        """Securely retrieve API key"""
        key_name = f"{provider.upper()}_API_KEY"
        api_key = os.getenv(key_name)
        
        if not api_key:
            # Try keyring as fallback
            try:
                import keyring
                api_key = keyring.get_password("mte-agent", key_name)
            except ImportError:
                pass
        
        if not api_key:
            raise ValueError(f"API key for {provider} not found")
        
        return api_key
```

### 8.2 Input Validation

```python
class InputValidator:
    """Validates and sanitizes input"""
    
    @staticmethod
    def validate_markdown_file(filepath: str) -> bool:
        """Validate markdown file"""
        path = Path(filepath)
        
        # Check file exists and is readable
        if not path.exists() or not path.is_file():
            return False
        
        # Check file extension
        if path.suffix.lower() not in ['.md', '.markdown']:
            return False
        
        # Check file size (max 10MB)
        if path.stat().st_size > 10 * 1024 * 1024:
            return False
        
        return True
```

## 9. Deployment

### 9.1 Docker Configuration

```dockerfile
# Dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/
COPY config/ ./config/

# Create non-root user
RUN useradd -m -u 1000 editor && chown -R editor:editor /app
USER editor

ENTRYPOINT ["python", "-m", "src.main"]
```

### 9.2 GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test and Deploy

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
        python-version: [3.9, '3.10', '3.11']
    
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
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Run linting
      run: |
        flake8 src/ tests/
        black --check src/ tests/
        mypy src/
    
    - name: Run tests with coverage
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        pytest tests/ --cov=src --cov-report=xml --cov-report=term
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
    
    - name: Build Docker image
      if: github.event_name == 'push' && github.ref == 'refs/heads/main'
      run: |
        docker build -t mte-agent:latest .
        docker tag mte-agent:latest mte-agent:${{ github.sha }}
```

This technical specification provides detailed implementation guidance for all system components, ensuring consistent development and maintainability.