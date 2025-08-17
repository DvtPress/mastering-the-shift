# GitHub Workflow Specification - Markdown Technical Editor Agent

## Overview

This document specifies the GitHub Actions workflows for continuous integration, testing, and deployment of the Markdown Technical Editor Agent.

## Workflow Structure

```
.github/
└── workflows/
    ├── ci.yml              # Main CI/CD workflow
    ├── test.yml            # Comprehensive testing
    ├── integration.yml     # Integration tests with real APIs
    ├── release.yml         # Release automation
    └── codeql.yml          # Security analysis
```

## Main CI/CD Workflow

### File: `.github/workflows/ci.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: 
      - main
      - develop
      - 'feature/**'
  pull_request:
    branches: 
      - main
      - develop
  schedule:
    # Run daily at 2 AM UTC
    - cron: '0 2 * * *'

env:
  PYTHON_VERSION: '3.9'
  NODE_VERSION: '18'

jobs:
  # Job 1: Code Quality Checks
  quality-checks:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history for better analysis
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Cache pip packages
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
        restore-keys: |
          ${{ runner.os }}-pip-
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements-dev.txt
    
    - name: Run linting with flake8
      run: |
        flake8 src/ tests/ \
          --max-line-length=100 \
          --exclude=__pycache__,venv,env \
          --statistics \
          --count
    
    - name: Check code formatting with black
      run: |
        black --check --diff src/ tests/
    
    - name: Run type checking with mypy
      run: |
        mypy src/ --ignore-missing-imports --strict
    
    - name: Check import sorting with isort
      run: |
        isort --check-only --diff src/ tests/
    
    - name: Security scan with bandit
      run: |
        bandit -r src/ -f json -o bandit-report.json
    
    - name: Upload security report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: bandit-security-report
        path: bandit-report.json

  # Job 2: Unit Tests
  unit-tests:
    name: Unit Tests (Python ${{ matrix.python-version }})
    runs-on: ${{ matrix.os }}
    needs: quality-checks
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11']
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Run unit tests with coverage
      run: |
        pytest tests/unit \
          --cov=src \
          --cov-report=xml \
          --cov-report=term-missing \
          --junit-xml=junit-${{ matrix.os }}-${{ matrix.python-version }}.xml \
          -v
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: coverage-${{ matrix.os }}-${{ matrix.python-version }}
        fail_ci_if_error: false
    
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: test-results-${{ matrix.os }}-${{ matrix.python-version }}
        path: junit-*.xml
    
    - name: Check coverage threshold
      run: |
        coverage report --fail-under=80

  # Job 3: Integration Tests
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event_name == 'push' || github.event_name == 'schedule'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Prepare test data
      run: |
        mkdir -p tests/fixtures/test_book
        cp -r tests/fixtures/sample_book/* tests/fixtures/test_book/
    
    - name: Run integration tests with mocked APIs
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
        TEST_MODE: "mock"
      run: |
        pytest tests/integration \
          -v \
          --tb=short \
          --junit-xml=integration-results.xml
    
    - name: Upload integration test results
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: integration-test-results
        path: integration-results.xml

  # Job 4: End-to-End Tests
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    
    - name: Run E2E tests
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        python -m src.main process \
          tests/fixtures/sample_book \
          --api openai \
          --dry-run \
          --verbose
    
    - name: Test API switching
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        python -m src.main process \
          tests/fixtures/sample_book \
          --api claude \
          --dry-run \
          --verbose

  # Job 5: Build and Package
  build:
    name: Build Package
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install build dependencies
      run: |
        python -m pip install --upgrade pip
        pip install build wheel setuptools
    
    - name: Build distribution packages
      run: |
        python -m build
    
    - name: Check distribution
      run: |
        pip install twine
        twine check dist/*
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: dist-packages
        path: dist/

  # Job 6: Docker Build
  docker:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Docker Hub
      if: github.ref == 'refs/heads/main'
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: |
          ${{ secrets.DOCKER_USERNAME }}/markdown-editor-agent
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: ${{ github.ref == 'refs/heads/main' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # Job 7: Documentation
  documentation:
    name: Generate Documentation
    runs-on: ubuntu-latest
    needs: quality-checks
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install documentation dependencies
      run: |
        pip install sphinx sphinx-rtd-theme sphinx-autodoc-typehints
    
    - name: Generate API documentation
      run: |
        sphinx-apidoc -o docs/api src/
        cd docs && make html
    
    - name: Upload documentation
      uses: actions/upload-artifact@v3
      with:
        name: documentation
        path: docs/_build/html/
```

## Integration Test Workflow

### File: `.github/workflows/integration.yml`

```yaml
name: Integration Tests with Live APIs

on:
  schedule:
    # Run weekly on Sunday at 3 AM UTC
    - cron: '0 3 * * 0'
  workflow_dispatch:
    inputs:
      api_provider:
        description: 'API Provider to test'
        required: true
        default: 'both'
        type: choice
        options:
          - openai
          - claude
          - both

jobs:
  live-api-tests:
    name: Live API Integration Tests
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Test with OpenAI API
      if: github.event.inputs.api_provider == 'openai' || github.event.inputs.api_provider == 'both' || github.event_name == 'schedule'
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        TEST_MODE: "live"
      run: |
        pytest tests/integration/test_openai_live.py \
          -v \
          --tb=short \
          --maxfail=3
    
    - name: Test with Claude API
      if: github.event.inputs.api_provider == 'claude' || github.event.inputs.api_provider == 'both' || github.event_name == 'schedule'
      env:
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
        TEST_MODE: "live"
      run: |
        pytest tests/integration/test_claude_live.py \
          -v \
          --tb=short \
          --maxfail=3
    
    - name: Process test book with live APIs
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
      run: |
        # Test with small sample to avoid excessive API costs
        python -m src.main process \
          tests/fixtures/small_sample \
          --api ${{ github.event.inputs.api_provider || 'openai' }} \
          --verbose
    
    - name: Generate cost report
      if: always()
      run: |
        python scripts/calculate_api_costs.py \
          --output cost-report.json
    
    - name: Upload cost report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: api-cost-report
        path: cost-report.json
```

## Release Workflow

### File: `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g., v1.0.0)'
        required: true

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        pip install build wheel twine
    
    - name: Build package
      run: |
        python -m build
    
    - name: Create changelog
      id: changelog
      run: |
        python scripts/generate_changelog.py \
          --from-tag $(git describe --tags --abbrev=0 HEAD^) \
          --to-tag ${{ github.ref_name }} \
          --output CHANGELOG_RELEASE.md
    
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v1
      with:
        tag_name: ${{ github.ref_name }}
        name: Release ${{ github.ref_name }}
        body_path: CHANGELOG_RELEASE.md
        draft: false
        prerelease: ${{ contains(github.ref_name, 'rc') || contains(github.ref_name, 'beta') }}
        files: |
          dist/*.whl
          dist/*.tar.gz
    
    - name: Publish to PyPI
      if: "!contains(github.ref_name, 'rc') && !contains(github.ref_name, 'beta')"
      env:
        TWINE_USERNAME: __token__
        TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
      run: |
        twine upload dist/*
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKER_USERNAME }}/markdown-editor-agent:${{ github.ref_name }}
          ${{ secrets.DOCKER_USERNAME }}/markdown-editor-agent:latest
```

## Security Analysis Workflow

### File: `.github/workflows/codeql.yml`

```yaml
name: CodeQL Security Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1'  # Weekly on Monday

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    
    strategy:
      fail-fast: false
      matrix:
        language: ['python']
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v2
      with:
        languages: ${{ matrix.language }}
        queries: security-and-quality
    
    - name: Autobuild
      uses: github/codeql-action/autobuild@v2
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2
      with:
        category: "/language:${{ matrix.language }}"
    
    - name: Run Trivy security scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Check for dependency vulnerabilities
      run: |
        pip install safety
        safety check --json --output safety-report.json
    
    - name: Upload safety report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: safety-vulnerability-report
        path: safety-report.json
```

## Dockerfile for CI/CD

### File: `Dockerfile`

```dockerfile
# Multi-stage build for production
FROM python:3.9-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.9-slim

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Create non-root user
RUN useradd -m -u 1000 editor && \
    chown -R editor:editor /app

USER editor

# Update PATH
ENV PATH=/root/.local/bin:$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import src; print('healthy')" || exit 1

ENTRYPOINT ["python", "-m", "src.main"]
CMD ["--help"]
```

## Test Data Preparation Script

### File: `scripts/prepare_test_data.sh`

```bash
#!/bin/bash

# Script to prepare test data for integration tests

set -e

echo "Preparing test data for integration tests..."

# Create test book structure
mkdir -p tests/fixtures/test_book/{chapters,appendices,images}

# Generate sample chapters
cat > tests/fixtures/test_book/chapters/chapter1.md << 'EOF'
# Chapter 1: Introduction to Testing

This chapter contains various markdown elements for testing.

## Facts to Check
- The Earth orbits the Sun in approximately 365.25 days.
- Water boils at 100°C at sea level.
- Python was first released in 1991.

## Code Block (Should Not Be Edited)
```python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

## Links and Images
- [OpenAI Documentation](https://platform.openai.com/docs)
- ![Test Diagram](../images/diagram.png)

## Text for Clarity Editing
The implementation of sophisticated algorithmic paradigms necessitates a comprehensive understanding of underlying theoretical foundations, which subsequently facilitates the development of optimized solutions.
EOF

cat > tests/fixtures/test_book/chapters/chapter2.md << 'EOF'
# Chapter 2: Advanced Topics

## Technical Content
The TCP/IP protocol stack consists of four layers: Application, Transport, Internet, and Network Access. Each layer provides specific services to the layers above it.

## Consistency Test
In this chapter, we use the word "color" for visual elements.
We configure the color settings in our application.
The color scheme should be consistent throughout.

## Factual Errors to Detect
- The sun rises in the west and sets in the east. (Wrong)
- HTML was invented in 1990 by Tim Berners-Lee. (Correct)
- JavaScript and Java are the same language. (Wrong)
EOF

# Create metadata file
cat > tests/fixtures/test_book/metadata.yaml << 'EOF'
title: "Test Book for Integration Testing"
author: "Test Author"
version: "1.0.0"
chapters:
  - chapters/chapter1.md
  - chapters/chapter2.md
processing_config:
  fact_checking: true
  editing_dimensions:
    - clarity
    - accuracy
    - consistency
EOF

echo "Test data prepared successfully!"
```

## GitHub Secrets Configuration

### Required Secrets

1. **API Keys**
   - `OPENAI_API_KEY`: OpenAI API key for testing
   - `CLAUDE_API_KEY`: Anthropic Claude API key for testing

2. **Docker Hub**
   - `DOCKER_USERNAME`: Docker Hub username
   - `DOCKER_PASSWORD`: Docker Hub access token

3. **PyPI**
   - `PYPI_API_TOKEN`: PyPI API token for package publishing

4. **Code Coverage**
   - `CODECOV_TOKEN`: Codecov.io token for coverage reports

### Setting Up Secrets

```bash
# Via GitHub CLI
gh secret set OPENAI_API_KEY --body "$OPENAI_API_KEY"
gh secret set CLAUDE_API_KEY --body "$CLAUDE_API_KEY"
gh secret set DOCKER_USERNAME --body "your-docker-username"
gh secret set DOCKER_PASSWORD --body "your-docker-token"
gh secret set PYPI_API_TOKEN --body "pypi-token"
gh secret set CODECOV_TOKEN --body "codecov-token"
```

## Branch Protection Rules

### Main Branch Protection

```yaml
# Settings for main branch protection
protection_rules:
  main:
    required_status_checks:
      strict: true
      contexts:
        - "Code Quality"
        - "Unit Tests (ubuntu-latest, 3.9)"
        - "Unit Tests (ubuntu-latest, 3.10)"
        - "Unit Tests (ubuntu-latest, 3.11)"
        - "Integration Tests"
    enforce_admins: false
    required_pull_request_reviews:
      required_approving_review_count: 1
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
    restrictions: null
    allow_force_pushes: false
    allow_deletions: false
```

## Monitoring and Notifications

### Slack Integration

```yaml
# Add to any workflow job
- name: Notify Slack on Failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'CI/CD Pipeline Failed!'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
    channel: '#ci-notifications'
```

### Email Notifications

```yaml
# Add to workflow
- name: Send email notification
  if: failure()
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    server_port: 465
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: Build Failed - ${{ github.repository }}
    body: |
      Build failed in workflow ${{ github.workflow }}.
      Repository: ${{ github.repository }}
      Branch: ${{ github.ref }}
      Commit: ${{ github.sha }}
    to: dev-team@example.com
```

## Performance Metrics Collection

### Workflow Metrics Script

```python
# scripts/collect_workflow_metrics.py
import json
import requests
from datetime import datetime, timedelta

def collect_metrics(repo, token, days=30):
    """Collect GitHub Actions metrics for the past N days"""
    
    headers = {"Authorization": f"token {token}"}
    base_url = f"https://api.github.com/repos/{repo}"
    
    # Calculate date range
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    
    # Fetch workflow runs
    url = f"{base_url}/actions/runs"
    params = {
        "created": f">{start_date.isoformat()}",
        "per_page": 100
    }
    
    response = requests.get(url, headers=headers, params=params)
    runs = response.json()["workflow_runs"]
    
    # Calculate metrics
    metrics = {
        "total_runs": len(runs),
        "successful_runs": sum(1 for r in runs if r["conclusion"] == "success"),
        "failed_runs": sum(1 for r in runs if r["conclusion"] == "failure"),
        "average_duration": sum(
            (datetime.fromisoformat(r["updated_at"].replace("Z", "+00:00")) - 
             datetime.fromisoformat(r["created_at"].replace("Z", "+00:00"))).seconds
            for r in runs if r["conclusion"]
        ) / max(len([r for r in runs if r["conclusion"]]), 1),
        "success_rate": sum(1 for r in runs if r["conclusion"] == "success") / max(len(runs), 1) * 100
    }
    
    return metrics

if __name__ == "__main__":
    import os
    
    metrics = collect_metrics(
        repo=os.getenv("GITHUB_REPOSITORY"),
        token=os.getenv("GITHUB_TOKEN"),
        days=30
    )
    
    print(json.dumps(metrics, indent=2))
    
    # Save to file
    with open("workflow-metrics.json", "w") as f:
        json.dump(metrics, f, indent=2)
```

## Best Practices

### 1. Workflow Optimization
- Use matrix builds for testing multiple Python versions
- Cache dependencies to speed up builds
- Run jobs in parallel when possible
- Use conditional steps to avoid unnecessary work

### 2. Security
- Never hardcode secrets in workflows
- Use environment variables for sensitive data
- Regularly update GitHub Actions versions
- Enable Dependabot for automatic updates

### 3. Cost Management
- Limit integration tests with live APIs
- Use scheduled runs for expensive operations
- Monitor API usage and costs
- Use mocked tests for most scenarios

### 4. Reliability
- Set appropriate timeouts for jobs
- Use retry logic for flaky tests
- Implement proper error handling
- Monitor workflow success rates

This comprehensive GitHub workflow specification ensures robust CI/CD practices, thorough testing, and reliable deployment processes for the Markdown Technical Editor Agent.