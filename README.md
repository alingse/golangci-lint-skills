# golangci-lint Skills

Claude Code skills for setting up and upgrading golangci-lint in Go projects.

## Overview

This repository contains two specialized skills for managing golangci-lint:

- **setup-golangci-lint**: Quickly set up golangci-lint environment for new Go projects
- **upgrade-golangci-lint**: Upgrade golangci-lint to the latest version and handle configuration migration

## Core Principle

**Do not modify existing code** - Only adjust configuration to make existing code pass lint checks. This ensures:

- Existing code passes lint safely
- New code follows strict rules
- Zero friction when introducing lint to projects

## Skills

### setup-golangci-lint

Initialize golangci-lint for Go projects with smart configuration.

**Features:**
- Auto-detect Go version and select appropriate golangci-lint version
- Create minimal configuration (v2 format)
- Integrate with CI (GitHub Actions, GitLab CI)
- Generate smart ignore rules based on 6-category classification
- Ensure existing code passes without modification

**Use when:**
- Starting a new Go project
- Adding lint to existing projects
- Integrating lint into CI/CD workflows

### upgrade-golangci-lint

Upgrade golangci-lint to the latest version with automatic migration.

**Features:**
- Auto-detect installation method (binary, Homebrew)
- Migrate v1 config to v2 format automatically
- Identify and analyze newly introduced linters
- Provide fix suggestions for new critical issues
- Preserve all existing disable/exclude settings

**Use when:**
- golangci-lint version is outdated
- CI lint fails due to version issues
- Want to use new linter features

## Linter Classification System

Both skills use a universal 6-category classification system:

| Category | Keywords | Action |
|----------|----------|--------|
| **a. Configurable** | complexity, long, deeply | Adjust settings thresholds |
| **b. Code Style** | style, format, naming | disable + TODO |
| **c. Critical Bugs** | bug, security, error | disable + TODO (suggest fix) |
| **d. Cannot Modify** | external constraints | exclusions |
| **e. Can Fix Small** | < 5 issues | exclude-rules |
| **f. New Features** | modern, new, latest | disable + TODO (defer) |

## Configuration Priority

1. **1️⃣ Highest**: Adjust `settings` thresholds
2. **2️⃣ Second**: Use `linters.exclusions` path exclusion
3. **3️⃣ Then**: Use `issues.exclude-rules` rule exclusion
4. **4️⃣ Last**: Completely `disable`

## Installation

Add these skills to your Claude Code skills directory.

## Example Usage

```bash
# For new projects
/setup-golangci-lint

# For existing projects with outdated lint
/upgrade-golangci-lint
```

## Supported Configuration Formats

- `.golangci.yml`
- `.golangci.yaml`
- `.golangci.toml`
- `.golangci.json`

## Requirements

- Go 1.20+ for golangci-lint v2
- Go 1.19- for golangci-lint v1

## Related Resources

- [Official golangci-lint Documentation](https://golangci-lint.run/)
- [Configuration Reference](https://golangci-lint.run/docs/configuration/)
- [Supported Linters](https://golangci-lint.run/usage/linters/)

## License

MIT
