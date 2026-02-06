# golangci-lint Skills

Claude Code skills for setting up and upgrading golangci-lint in Go projects.

## Skills

### setup-golangci-lint

Quickly set up golangci-lint environment for Go projects.

**Features:**
- Auto-detect Go version and select appropriate golangci-lint version
- Create minimal configuration (v2 format)
- Integrate with CI (GitHub Actions, GitLab CI)
- Generate smart ignore rules to ensure existing code passes safely

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

**Use when:**
- golangci-lint version is outdated
- CI lint fails due to version issues
- Want to use new linter features

## Installation

```bash
npx skills add alingse/golangci-lint-skills
```

## Usage

```bash
# For new projects
/setup-golangci-lint

# For existing projects with outdated lint
/upgrade-golangci-lint
```

## Related Resources

- [Official golangci-lint Documentation](https://golangci-lint.run/)
- [Configuration Reference](https://golangci-lint.run/docs/configuration/)
