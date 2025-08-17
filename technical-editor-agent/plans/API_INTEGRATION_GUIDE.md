# API Integration Guide - Markdown Technical Editor Agent

## Overview

This guide provides detailed instructions for integrating both OpenAI and Claude APIs into the Markdown Technical Editor Agent. The system uses an abstraction layer to support both providers seamlessly.

## API Client Architecture

### Abstract Base Client

```python
from abc import ABC, abstractmethod
from typing import List, Optional, Dict, Any
from dataclasses import dataclass

@dataclass
class FactCheckResult:
    statement: str
    confidence: float  # 0.0 to 1.0
    verdict: str  # "accurate", "inaccurate", "uncertain"
    explanation: str
    sources: List[str] = None

@dataclass
class EditInstructions:
    dimension: str  # "clarity", "accuracy", "consistency", "usability"
    aggressiveness: str  # "low", "medium", "high"
    context: Optional[str] = None
    style_guide: Optional[Dict[str, Any]] = None

@dataclass
class EditResult:
    original_text: str
    edited_text: str
    changes_made: List[str]
    confidence: float

class AIClient(ABC):
    """Abstract base class for AI API clients"""
    
    @abstractmethod
    async def initialize(self) -> None:
        """Initialize the client connection"""
        pass
    
    @abstractmethod
    async def check_fact(
        self, 
        statement: str, 
        context: Optional[str] = None
    ) -> FactCheckResult:
        """Verify a factual statement"""
        pass
    
    @abstractmethod
    async def edit_text(
        self, 
        text: str, 
        instructions: EditInstructions
    ) -> EditResult:
        """Edit text according to instructions"""
        pass
    
    @abstractmethod
    async def analyze_content(
        self, 
        content: str
    ) -> Dict[str, Any]:
        """Analyze content for various metrics"""
        pass
    
    @abstractmethod
    def estimate_tokens(self, text: str) -> int:
        """Estimate token count for text"""
        pass
    
    @abstractmethod
    async def close(self) -> None:
        """Clean up client resources"""
        pass
```

## OpenAI Integration

### Installation

```bash
pip install openai>=1.0.0
```

### Client Implementation

```python
import openai
from openai import AsyncOpenAI
import tiktoken
from typing import Optional, Dict, Any
import json

class OpenAIClient(AIClient):
    """OpenAI API client implementation"""
    
    # Prompt templates
    FACT_CHECK_PROMPT = """You are a fact-checking assistant. Analyze the following statement for factual accuracy.

Provide your response in JSON format:
{
    "confidence": 0.0-1.0,
    "verdict": "accurate" | "inaccurate" | "uncertain",
    "explanation": "Brief explanation",
    "sources": ["source1", "source2"] (if applicable)
}

Statement: {statement}
Context: {context}"""

    EDIT_PROMPT_CLARITY = """You are a technical editor focusing on clarity. 
Edit the following text to improve clarity while preserving technical accuracy.
Make the text easier to understand without changing its meaning.

Original text: {text}

Provide your response in JSON format:
{
    "edited_text": "...",
    "changes_made": ["change1", "change2"],
    "confidence": 0.0-1.0
}"""

    def __init__(
        self, 
        api_key: str, 
        model: str = "gpt-5",
        temperature: float = 0.3,
        max_tokens: int = 4000
    ):
        self.api_key = api_key
        self.model = model
        self.temperature = temperature
        self.max_tokens = max_tokens
        self.client: Optional[AsyncOpenAI] = None
        self.encoding = tiktoken.encoding_for_model(model)
        
    async def initialize(self) -> None:
        """Initialize the OpenAI client"""
        self.client = AsyncOpenAI(api_key=self.api_key)
        
    async def check_fact(
        self, 
        statement: str, 
        context: Optional[str] = None
    ) -> FactCheckResult:
        """Verify a factual statement using OpenAI"""
        
        prompt = self.FACT_CHECK_PROMPT.format(
            statement=statement,
            context=context or "No additional context provided"
        )
        
        try:
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "You are a fact-checking assistant."},
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=500,
                response_format={"type": "json_object"}
            )
            
            result_data = json.loads(response.choices[0].message.content)
            
            return FactCheckResult(
                statement=statement,
                confidence=result_data.get("confidence", 0.5),
                verdict=result_data.get("verdict", "uncertain"),
                explanation=result_data.get("explanation", ""),
                sources=result_data.get("sources", [])
            )
            
        except Exception as e:
            logger.error(f"OpenAI fact check failed: {e}")
            raise
            
    async def edit_text(
        self, 
        text: str, 
        instructions: EditInstructions
    ) -> EditResult:
        """Edit text using OpenAI"""
        
        # Select appropriate prompt based on dimension
        prompt_map = {
            "clarity": self.EDIT_PROMPT_CLARITY,
            "accuracy": self.EDIT_PROMPT_ACCURACY,
            "consistency": self.EDIT_PROMPT_CONSISTENCY,
            "usability": self.EDIT_PROMPT_USABILITY
        }
        
        prompt_template = prompt_map.get(
            instructions.dimension, 
            self.EDIT_PROMPT_CLARITY
        )
        
        prompt = prompt_template.format(text=text)
        
        # Add style guide if provided
        if instructions.style_guide:
            prompt += f"\n\nStyle guide: {json.dumps(instructions.style_guide)}"
            
        try:
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"You are a technical editor focusing on {instructions.dimension}."},
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=self.max_tokens,
                response_format={"type": "json_object"}
            )
            
            result_data = json.loads(response.choices[0].message.content)
            
            return EditResult(
                original_text=text,
                edited_text=result_data.get("edited_text", text),
                changes_made=result_data.get("changes_made", []),
                confidence=result_data.get("confidence", 0.8)
            )
            
        except Exception as e:
            logger.error(f"OpenAI text edit failed: {e}")
            raise
            
    def estimate_tokens(self, text: str) -> int:
        """Estimate token count using tiktoken"""
        return len(self.encoding.encode(text))
        
    async def close(self) -> None:
        """Close the OpenAI client"""
        if self.client:
            await self.client.close()
```

### OpenAI Configuration

```python
# OpenAI-specific configuration
OPENAI_CONFIG = {
    "models": {
        "primary": "gpt-5",
        "fallback": "gpt-4-turbo-preview"  # Fallback until GPT-5 is available
    },
    "rate_limits": {
        "gpt-5": {
            "requests_per_minute": "TBD",  # To be determined when GPT-5 is released
            "tokens_per_minute": "TBD"
        },
        "gpt-4-turbo-preview": {
            "requests_per_minute": 500,
            "tokens_per_minute": 10000
        }
    },
    "retry_config": {
        "max_retries": 3,
        "initial_delay": 1.0,
        "max_delay": 60.0,
        "exponential_base": 2.0
    }
}
```

## Claude Integration

### Installation

```bash
pip install anthropic>=0.7.0
```

### Client Implementation

```python
import anthropic
from anthropic import AsyncAnthropic
import json
from typing import Optional, Dict, Any

class ClaudeClient(AIClient):
    """Claude API client implementation"""
    
    # Claude-specific prompts
    FACT_CHECK_PROMPT = """I need you to fact-check the following statement and provide a detailed analysis.

Statement to verify: {statement}
Additional context: {context}

Please analyze this statement and provide your response in the following JSON format:
{{
    "confidence": <float between 0.0 and 1.0>,
    "verdict": "<one of: accurate, inaccurate, uncertain>",
    "explanation": "<detailed explanation of your verdict>",
    "sources": ["<relevant sources if available>"]
}}

Focus on factual accuracy and provide specific reasoning for your confidence level."""

    EDIT_PROMPT_TEMPLATE = """You are an expert technical editor. Your task is to edit the following text with a focus on {dimension}.

Editing instructions:
- Dimension: {dimension}
- Aggressiveness: {aggressiveness}
- Preserve technical accuracy
- Maintain the original meaning
{additional_instructions}

Original text:
{text}

Provide your edited version in the following JSON format:
{{
    "edited_text": "<your edited version>",
    "changes_made": ["<list of specific changes>"],
    "confidence": <float between 0.0 and 1.0>
}}"""

    def __init__(
        self,
        api_key: str,
        model: str = "claude-sonnet-4",
        temperature: float = 0.3,
        max_tokens: int = 4000
    ):
        self.api_key = api_key
        self.model = model
        self.temperature = temperature
        self.max_tokens = max_tokens
        self.client: Optional[AsyncAnthropic] = None
        
    async def initialize(self) -> None:
        """Initialize the Claude client"""
        self.client = AsyncAnthropic(api_key=self.api_key)
        
    async def check_fact(
        self,
        statement: str,
        context: Optional[str] = None
    ) -> FactCheckResult:
        """Verify a factual statement using Claude"""
        
        prompt = self.FACT_CHECK_PROMPT.format(
            statement=statement,
            context=context or "No additional context provided"
        )
        
        try:
            response = await self.client.messages.create(
                model=self.model,
                messages=[
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=500
            )
            
            # Extract JSON from Claude's response
            content = response.content[0].text
            json_start = content.find('{')
            json_end = content.rfind('}') + 1
            json_str = content[json_start:json_end]
            result_data = json.loads(json_str)
            
            return FactCheckResult(
                statement=statement,
                confidence=result_data.get("confidence", 0.5),
                verdict=result_data.get("verdict", "uncertain"),
                explanation=result_data.get("explanation", ""),
                sources=result_data.get("sources", [])
            )
            
        except Exception as e:
            logger.error(f"Claude fact check failed: {e}")
            raise
            
    async def edit_text(
        self,
        text: str,
        instructions: EditInstructions
    ) -> EditResult:
        """Edit text using Claude"""
        
        # Build additional instructions based on dimension
        additional_instructions = self._build_dimension_instructions(instructions)
        
        prompt = self.EDIT_PROMPT_TEMPLATE.format(
            dimension=instructions.dimension,
            aggressiveness=instructions.aggressiveness,
            additional_instructions=additional_instructions,
            text=text
        )
        
        try:
            response = await self.client.messages.create(
                model=self.model,
                messages=[
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=self.max_tokens
            )
            
            # Extract JSON from response
            content = response.content[0].text
            json_start = content.find('{')
            json_end = content.rfind('}') + 1
            json_str = content[json_start:json_end]
            result_data = json.loads(json_str)
            
            return EditResult(
                original_text=text,
                edited_text=result_data.get("edited_text", text),
                changes_made=result_data.get("changes_made", []),
                confidence=result_data.get("confidence", 0.8)
            )
            
        except Exception as e:
            logger.error(f"Claude text edit failed: {e}")
            raise
            
    def _build_dimension_instructions(self, instructions: EditInstructions) -> str:
        """Build dimension-specific instructions"""
        
        dimension_guides = {
            "clarity": """
- Simplify complex sentences
- Replace jargon with clearer terms
- Improve sentence structure
- Add helpful transitions""",
            "accuracy": """
- Correct factual errors
- Update outdated information
- Ensure technical correctness
- Verify claims against context""",
            "consistency": """
- Maintain consistent terminology
- Ensure uniform style
- Standardize formatting
- Align tone throughout""",
            "usability": """
- Improve readability
- Organize information logically
- Add helpful context
- Enhance navigation"""
        }
        
        guide = dimension_guides.get(instructions.dimension, "")
        
        if instructions.style_guide:
            guide += f"\n\nStyle guide requirements:\n{json.dumps(instructions.style_guide, indent=2)}"
            
        return guide
        
    def estimate_tokens(self, text: str) -> int:
        """Estimate token count for Claude (approximate)"""
        # Claude uses a different tokenization than OpenAI
        # This is a rough approximation
        words = len(text.split())
        return int(words * 1.3)  # Rough estimate
        
    async def close(self) -> None:
        """Close the Claude client"""
        # Claude client doesn't require explicit closing
        pass
```

### Claude Configuration

```python
# Claude-specific configuration
CLAUDE_CONFIG = {
    "models": {
        "primary": "claude-sonnet-4",
        "fallback": "claude-3-sonnet-20240229"  # Fallback until Claude Sonnet 4 is available
    },
    "rate_limits": {
        "claude-sonnet-4": {
            "requests_per_minute": "TBD",  # To be determined when Claude Sonnet 4 is released
            "tokens_per_minute": "TBD"
        },
        "claude-3-sonnet-20240229": {
            "requests_per_minute": 50,
            "tokens_per_minute": 40000
        }
    },
    "context_window": {
        "claude-sonnet-4": "TBD",  # To be determined
        "claude-3-sonnet-20240229": 200000
    },
    "retry_config": {
        "max_retries": 3,
        "initial_delay": 1.0,
        "max_delay": 60.0
    }
}
```

## Client Factory

### Implementation

```python
from typing import Union
from enum import Enum

class APIProvider(Enum):
    OPENAI = "openai"
    CLAUDE = "claude"

class AIClientFactory:
    """Factory for creating AI client instances"""
    
    @staticmethod
    def create_client(
        provider: Union[str, APIProvider],
        api_key: str,
        model: Optional[str] = None,
        **kwargs
    ) -> AIClient:
        """Create an AI client based on provider"""
        
        if isinstance(provider, str):
            provider = APIProvider(provider.lower())
            
        if provider == APIProvider.OPENAI:
            model = model or "gpt-5"
            return OpenAIClient(
                api_key=api_key,
                model=model,
                **kwargs
            )
        elif provider == APIProvider.CLAUDE:
            model = model or "claude-sonnet-4"
            return ClaudeClient(
                api_key=api_key,
                model=model,
                **kwargs
            )
        else:
            raise ValueError(f"Unsupported provider: {provider}")
            
    @staticmethod
    async def create_and_initialize(
        provider: Union[str, APIProvider],
        api_key: str,
        **kwargs
    ) -> AIClient:
        """Create and initialize a client"""
        client = AIClientFactory.create_client(provider, api_key, **kwargs)
        await client.initialize()
        return client
```

## Rate Limiting

### Rate Limiter Implementation

```python
import asyncio
from datetime import datetime, timedelta
from collections import deque
from typing import Optional

class RateLimiter:
    """Token bucket rate limiter for API calls"""
    
    def __init__(
        self,
        requests_per_minute: int,
        tokens_per_minute: int
    ):
        self.requests_per_minute = requests_per_minute
        self.tokens_per_minute = tokens_per_minute
        
        # Request tracking
        self.request_times = deque()
        self.token_usage = deque()
        
        # Lock for thread safety
        self.lock = asyncio.Lock()
        
    async def acquire(self, estimated_tokens: int) -> None:
        """Wait if necessary to respect rate limits"""
        async with self.lock:
            now = datetime.now()
            
            # Clean old entries
            self._clean_old_entries(now)
            
            # Check request limit
            while len(self.request_times) >= self.requests_per_minute:
                wait_time = self._calculate_wait_time(
                    self.request_times[0], 
                    now
                )
                if wait_time > 0:
                    await asyncio.sleep(wait_time)
                    now = datetime.now()
                    self._clean_old_entries(now)
                else:
                    break
                    
            # Check token limit
            current_tokens = sum(t[1] for t in self.token_usage)
            while current_tokens + estimated_tokens > self.tokens_per_minute:
                if self.token_usage:
                    wait_time = self._calculate_wait_time(
                        self.token_usage[0][0],
                        now
                    )
                    if wait_time > 0:
                        await asyncio.sleep(wait_time)
                        now = datetime.now()
                        self._clean_old_entries(now)
                        current_tokens = sum(t[1] for t in self.token_usage)
                    else:
                        break
                else:
                    break
                    
            # Record usage
            self.request_times.append(now)
            self.token_usage.append((now, estimated_tokens))
            
    def _clean_old_entries(self, now: datetime) -> None:
        """Remove entries older than 1 minute"""
        cutoff = now - timedelta(minutes=1)
        
        while self.request_times and self.request_times[0] < cutoff:
            self.request_times.popleft()
            
        while self.token_usage and self.token_usage[0][0] < cutoff:
            self.token_usage.popleft()
            
    def _calculate_wait_time(
        self,
        oldest_time: datetime,
        now: datetime
    ) -> float:
        """Calculate wait time in seconds"""
        one_minute_later = oldest_time + timedelta(minutes=1)
        if one_minute_later > now:
            return (one_minute_later - now).total_seconds()
        return 0
```

## Error Handling

### Retry Logic

```python
import asyncio
from typing import TypeVar, Callable, Optional
from functools import wraps

T = TypeVar('T')

class RetryConfig:
    """Configuration for retry logic"""
    
    def __init__(
        self,
        max_retries: int = 3,
        initial_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: bool = True
    ):
        self.max_retries = max_retries
        self.initial_delay = initial_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter

def with_retry(config: Optional[RetryConfig] = None):
    """Decorator for adding retry logic to async functions"""
    
    if config is None:
        config = RetryConfig()
        
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(config.max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except (
                    openai.RateLimitError,
                    anthropic.RateLimitError,
                    asyncio.TimeoutError
                ) as e:
                    last_exception = e
                    
                    if attempt < config.max_retries:
                        delay = min(
                            config.initial_delay * (config.exponential_base ** attempt),
                            config.max_delay
                        )
                        
                        if config.jitter:
                            import random
                            delay *= (0.5 + random.random())
                            
                        logger.warning(
                            f"Attempt {attempt + 1} failed: {e}. "
                            f"Retrying in {delay:.2f} seconds..."
                        )
                        await asyncio.sleep(delay)
                    else:
                        logger.error(f"All retry attempts failed: {e}")
                        
            raise last_exception
            
        return wrapper
    return decorator
```

### Error Handling Example

```python
class APIErrorHandler:
    """Centralized error handling for API calls"""
    
    @staticmethod
    async def handle_api_call(
        client: AIClient,
        method: str,
        *args,
        **kwargs
    ):
        """Safely execute an API call with error handling"""
        
        try:
            # Get the method
            api_method = getattr(client, method)
            
            # Execute with retry
            @with_retry(RetryConfig(max_retries=3))
            async def execute():
                return await api_method(*args, **kwargs)
                
            return await execute()
            
        except openai.AuthenticationError as e:
            logger.error(f"OpenAI authentication failed: {e}")
            raise ValueError("Invalid OpenAI API key")
            
        except anthropic.AuthenticationError as e:
            logger.error(f"Claude authentication failed: {e}")
            raise ValueError("Invalid Claude API key")
            
        except (openai.BadRequestError, anthropic.BadRequestError) as e:
            logger.error(f"Bad request: {e}")
            raise ValueError(f"Invalid request parameters: {e}")
            
        except Exception as e:
            logger.error(f"Unexpected API error: {e}")
            raise
```

## Usage Examples

### Basic Usage

```python
import asyncio
from typing import List

async def process_document_with_fact_checking(
    document: str,
    provider: str = "openai"
):
    """Example of processing a document with fact-checking"""
    
    # Create client
    api_key = os.getenv(f"{provider.upper()}_API_KEY")
    client = await AIClientFactory.create_and_initialize(
        provider=provider,
        api_key=api_key,
        temperature=0.3
    )
    
    try:
        # Extract statements (simplified)
        statements = document.split('. ')
        
        # Fact-check each statement
        results = []
        for statement in statements:
            if len(statement.strip()) > 10:  # Skip short fragments
                result = await client.check_fact(
                    statement=statement,
                    context=document[:500]  # Provide context
                )
                results.append(result)
                
                # Log warnings for low-confidence facts
                if result.confidence < 0.7:
                    logger.warning(
                        f"Low confidence fact: {statement[:50]}... "
                        f"(confidence: {result.confidence})"
                    )
                    
        return results
        
    finally:
        await client.close()
```

### Advanced Usage with Both APIs

```python
class DualAPIProcessor:
    """Process using both APIs for comparison"""
    
    def __init__(self, openai_key: str, claude_key: str):
        self.openai_key = openai_key
        self.claude_key = claude_key
        
    async def compare_edits(
        self,
        text: str,
        dimension: str = "clarity"
    ):
        """Get edits from both APIs and compare"""
        
        # Create both clients
        openai_client = await AIClientFactory.create_and_initialize(
            provider="openai",
            api_key=self.openai_key
        )
        
        claude_client = await AIClientFactory.create_and_initialize(
            provider="claude",
            api_key=self.claude_key
        )
        
        try:
            # Get edits from both
            instructions = EditInstructions(
                dimension=dimension,
                aggressiveness="medium"
            )
            
            openai_result, claude_result = await asyncio.gather(
                openai_client.edit_text(text, instructions),
                claude_client.edit_text(text, instructions)
            )
            
            # Compare results
            comparison = {
                "original": text,
                "openai": {
                    "edited": openai_result.edited_text,
                    "confidence": openai_result.confidence,
                    "changes": openai_result.changes_made
                },
                "claude": {
                    "edited": claude_result.edited_text,
                    "confidence": claude_result.confidence,
                    "changes": claude_result.changes_made
                },
                "agreement": self._calculate_agreement(
                    openai_result.edited_text,
                    claude_result.edited_text
                )
            }
            
            return comparison
            
        finally:
            await openai_client.close()
            await claude_client.close()
            
    def _calculate_agreement(self, text1: str, text2: str) -> float:
        """Calculate similarity between two texts"""
        # Simplified similarity calculation
        if text1 == text2:
            return 1.0
            
        # Use Levenshtein distance or similar metric
        from difflib import SequenceMatcher
        return SequenceMatcher(None, text1, text2).ratio()
```

## Testing API Integrations

### Mock Clients for Testing

```python
class MockAIClient(AIClient):
    """Mock AI client for testing"""
    
    def __init__(self, responses: Dict[str, Any] = None):
        self.responses = responses or {}
        self.call_count = {
            "check_fact": 0,
            "edit_text": 0,
            "analyze_content": 0
        }
        
    async def initialize(self) -> None:
        pass
        
    async def check_fact(
        self,
        statement: str,
        context: Optional[str] = None
    ) -> FactCheckResult:
        self.call_count["check_fact"] += 1
        
        # Return predefined response or default
        if "fact_check" in self.responses:
            return self.responses["fact_check"]
            
        return FactCheckResult(
            statement=statement,
            confidence=0.9,
            verdict="accurate",
            explanation="Mock verification",
            sources=["mock source"]
        )
        
    async def edit_text(
        self,
        text: str,
        instructions: EditInstructions
    ) -> EditResult:
        self.call_count["edit_text"] += 1
        
        if "edit" in self.responses:
            return self.responses["edit"]
            
        return EditResult(
            original_text=text,
            edited_text=text + " (edited)",
            changes_made=["Mock edit applied"],
            confidence=0.95
        )
        
    def estimate_tokens(self, text: str) -> int:
        return len(text.split())
        
    async def close(self) -> None:
        pass
```

### Integration Tests

```python
import pytest
from unittest.mock import patch, AsyncMock

@pytest.mark.asyncio
async def test_openai_fact_checking():
    """Test OpenAI fact-checking integration"""
    
    with patch('openai.AsyncOpenAI') as mock_openai:
        # Setup mock response
        mock_response = AsyncMock()
        mock_response.choices = [
            AsyncMock(
                message=AsyncMock(
                    content='{"confidence": 0.95, "verdict": "accurate", '
                           '"explanation": "This is correct", "sources": []}'
                )
            )
        ]
        mock_openai.return_value.chat.completions.create.return_value = mock_response
        
        # Test fact checking
        client = OpenAIClient(api_key="test_key")
        await client.initialize()
        
        result = await client.check_fact(
            statement="The Earth orbits the Sun",
            context="Solar system facts"
        )
        
        assert result.confidence == 0.95
        assert result.verdict == "accurate"
        assert "correct" in result.explanation
```

## Performance Optimization

### Batch Processing

```python
class BatchAPIProcessor:
    """Batch processing for API efficiency"""
    
    def __init__(self, client: AIClient, batch_size: int = 5):
        self.client = client
        self.batch_size = batch_size
        self.rate_limiter = RateLimiter(
            requests_per_minute=50,
            tokens_per_minute=10000
        )
        
    async def batch_process(
        self,
        items: List[str],
        operation: str,
        **kwargs
    ) -> List[Any]:
        """Process items in batches"""
        
        results = []
        
        for i in range(0, len(items), self.batch_size):
            batch = items[i:i + self.batch_size]
            
            # Estimate tokens for rate limiting
            total_tokens = sum(
                self.client.estimate_tokens(item) 
                for item in batch
            )
            
            # Wait for rate limit
            await self.rate_limiter.acquire(total_tokens)
            
            # Process batch concurrently
            batch_tasks = []
            for item in batch:
                if operation == "fact_check":
                    task = self.client.check_fact(item, **kwargs)
                elif operation == "edit":
                    task = self.client.edit_text(item, **kwargs)
                else:
                    raise ValueError(f"Unknown operation: {operation}")
                    
                batch_tasks.append(task)
                
            batch_results = await asyncio.gather(*batch_tasks)
            results.extend(batch_results)
            
        return results
```

## Monitoring and Metrics

### API Usage Tracker

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Dict, List

@dataclass
class APIMetrics:
    """Track API usage metrics"""
    
    provider: str
    total_requests: int = 0
    total_tokens: int = 0
    total_cost: float = 0.0
    errors: int = 0
    average_latency: float = 0.0
    requests_by_type: Dict[str, int] = field(default_factory=dict)
    
class APIUsageTracker:
    """Track and report API usage"""
    
    def __init__(self):
        self.metrics: Dict[str, APIMetrics] = {}
        self.request_log: List[Dict] = []
        
    def record_request(
        self,
        provider: str,
        operation: str,
        tokens: int,
        latency: float,
        success: bool = True
    ):
        """Record an API request"""
        
        if provider not in self.metrics:
            self.metrics[provider] = APIMetrics(provider=provider)
            
        metrics = self.metrics[provider]
        metrics.total_requests += 1
        metrics.total_tokens += tokens
        
        if operation not in metrics.requests_by_type:
            metrics.requests_by_type[operation] = 0
        metrics.requests_by_type[operation] += 1
        
        if not success:
            metrics.errors += 1
            
        # Update average latency
        metrics.average_latency = (
            (metrics.average_latency * (metrics.total_requests - 1) + latency) /
            metrics.total_requests
        )
        
        # Estimate cost (simplified)
        if provider == "openai":
            # GPT-4 pricing (example)
            metrics.total_cost += (tokens / 1000) * 0.03
        elif provider == "claude":
            # Claude pricing (example)
            metrics.total_cost += (tokens / 1000) * 0.024
            
        # Log request
        self.request_log.append({
            "timestamp": datetime.now().isoformat(),
            "provider": provider,
            "operation": operation,
            "tokens": tokens,
            "latency": latency,
            "success": success
        })
        
    def get_summary(self) -> Dict:
        """Get usage summary"""
        return {
            provider: {
                "total_requests": metrics.total_requests,
                "total_tokens": metrics.total_tokens,
                "total_cost": f"${metrics.total_cost:.2f}",
                "error_rate": f"{(metrics.errors / max(metrics.total_requests, 1)) * 100:.1f}%",
                "average_latency": f"{metrics.average_latency:.2f}s",
                "requests_by_type": metrics.requests_by_type
            }
            for provider, metrics in self.metrics.items()
        }
```

This comprehensive API integration guide provides everything needed to successfully integrate both OpenAI and Claude APIs into the Markdown Technical Editor Agent, with proper error handling, rate limiting, and monitoring capabilities.