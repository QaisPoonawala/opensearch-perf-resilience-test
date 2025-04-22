# Contributing to OpenSearch Benchmark

Thank you for your interest in contributing to OpenSearch Benchmark! This document provides guidelines and instructions for contributing to this project.

## Code of Conduct

This project adheres to the [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code.

## How to Contribute

### Reporting Issues

- Before creating an issue, please check existing issues to avoid duplicates
- When creating an issue, provide:
  - Clear description of the problem
  - Steps to reproduce
  - Expected vs actual behavior
  - Version information (OpenSearch, Python, OS)
  - Relevant logs or error messages

### Pull Requests

1. Fork the repository and create your branch from `main`
2. If you've added code that should be tested, add tests
3. Ensure your code follows the project's coding style
4. Update documentation as needed
5. Write a clear commit message
6. Submit a pull request

### Development Setup

1. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/opensearch-benchmark.git
   cd opensearch-benchmark
   ```

2. Create a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### Coding Standards

- Follow PEP 8 style guide for Python code
- Use meaningful variable and function names
- Write docstrings for functions and classes
- Keep functions focused and modular
- Add comments for complex logic
- Use type hints where appropriate

### Testing

- Write unit tests for new functionality
- Ensure all tests pass before submitting PR
- Run tests using:
  ```bash
  python -m pytest
  ```

### Documentation

- Update documentation for new features or changes
- Include docstrings for public functions and classes
- Update README.md if needed
- Add examples for new functionality

## Project Structure

```
opensearch-benchmark/
├── builder/           # Cluster building and management
├── resources/         # Configuration and templates
├── utils/            # Utility functions
├── workload/         # Workload management
└── worker_coordinator/ # Test coordination
```

## Release Process

1. Version numbers follow [Semantic Versioning](https://semver.org/)
2. Changes are documented in CHANGELOG.md
3. Releases are tagged in git
4. Release notes detail new features and breaking changes

## Getting Help

- Check the documentation
- Search existing issues
- Join the OpenSearch community channels
- Ask questions on GitHub discussions

## License

By contributing to OpenSearch Benchmark, you agree that your contributions will be licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
