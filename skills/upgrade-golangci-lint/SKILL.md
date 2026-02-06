---
name: upgrade-golangci-lint
description: Upgrade golangci-lint to the latest version, automatically migrate config (v1→v2), run lint and supplement config to make code pass. Core principle: only adjust config, do not modify code.
---

# golangci-lint Version Upgrade

Upgrade golangci-lint to the latest version, handle configuration migration, and adjust configuration to make existing code pass lint checks.

> **Core Principle**: Make existing code pass lint by adjusting configuration, **do not modify existing code**. Only affects new code.

## When to Use

- golangci-lint version is outdated and needs to be upgraded to the latest version
- lint fails in CI due to version issues
- Want to use new linter features from the latest version

## Execution Flow

### 1. Detect Current State

```bash
# Check current version
golangci-lint --version

# Check for configuration file (supports 4 formats)
ls .golangci.* 2>/dev/null || echo "No config"

# Check config format version (v1 or v2) - only valid for YAML format
grep "^version:" .golangci.yml .golangci.yaml 2>/dev/null && echo "v2 config" || echo "v1 config (needs migration)"

# Detect installation method
if command -v brew &> /dev/null && brew list golangci-lint &> /dev/null 2>/dev/null; then
    INSTALL_METHOD="brew"
elif command -v golangci-lint &> /dev/null; then
    INSTALL_METHOD="binary"
else
    echo "golangci-lint not installed, please run setup-golangci-lint first"
    exit 1
fi
```

### 2. Upgrade Binary

Execute upgrade based on detected installation method:

| Installation Method | Upgrade Command |
|---------------------|-----------------|
| **Binary** | `curl -sSfL https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest` |
| **Homebrew** | `brew upgrade golangci-lint` |
| **Alpine (no curl)** | `wget -O- -nv https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest` |

```bash
# Binary upgrade (recommended)
curl -sSfL https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest

# Verify upgrade success
golangci-lint --version
```

### 3. Configuration Migration (v1 → v2)

If v1 configuration is detected (no `version:` field in config file), execute migration:

```bash
# Automatically migrate v1 config to v2 format
golangci-lint migrate --skip-validation
```

**Notes**:
- `--skip-validation`: Skip validation, convert directly (recommended)
- `migrate` command will automatically:
  - Add `version: 2`
  - Convert `enable-all/disable-all` to `linters.default`
  - Change `linters-settings` to `linters.settings`
  - Change `issues.exclude-rules` to `linters.exclusions.rules`
  - **Preserve all existing disable and exclude configurations**

After migration, manually check the configuration to ensure:
- `linters.default` value meets expectations (standard/all/none/fast)
- Disabled linters still have `# TODO fix later by human` comments
- Complexity thresholds are raised and marked with `# TODO reduce this`

```bash
# Verify configuration format
golangci-lint config verify
```

### 4. Run Lint Check

```bash
# Run lint
golangci-lint run --timeout=5m ./...
```

### 4.5. Identify and Handle New Linters

> **🆕 Special Handling for New Linters**:
> After version upgrade, the new version may introduce linters that didn't exist before. Handling strategy for these new linters differs from existing linters.

**Identify New Linters**:

```bash
# Save old version linter list (execute before upgrade)
golangci-lint help linters > /tmp/old_linters.txt

# Compare after upgrade
golangci-lint help linters > /tmp/new_linters.txt
diff /tmp/old_linters.txt /tmp/new_linters.txt
```

Or simply check the error output after upgrade to identify previously unseen linter names.

**New Linter Handling Strategy**:

| Issue Count | Category | Action | Notes |
|-------------|----------|--------|-------|
| **< 5** | Any category | **Provide fix suggestions** | Few issues, worth fixing |
| **5-20** | c. Critical bugs | **Provide fix suggestions** | Security/error issues, recommended to fix |
| **5-20** | a. Configurable | Adjust settings | Prioritize threshold adjustment |
| **5-20** | b/d/f Others | disable + TODO | Handle per original logic |
| **> 20** | Any category | disable + TODO | Too many issues, handle later |

**Fix Suggestion Format**:

```markdown
## 🆕 New Linter Fix Suggestions

The following linters are newly introduced in this upgrade and have few issues, recommended to fix:

### Linter: nilerr
- **Issue Count**: 3
- **Severity**: High (may cause bugs)
- **Fix Suggestions**:
  1. `pkg/api/handler.go:45` - Check error return is correct
  2. `pkg/service/user.go:123` - Ensure non-nil return when error is non-nil
  3. `pkg/db/client.go:67` - Fix error handling logic

### Linter: contextcheck
- **Issue Count**: 2
- **Severity**: Medium (may cause context loss)
- **Fix Suggestions**:
  1. `pkg/api/client.go:89` - Add context parameter
  2. `pkg/service/order.go:156` - Pass context to downstream

---

**Would you like me to help fix these issues?** Reply "fix" to start fixing.
```

**Why Not Auto-Fix New Linter Issues**:
1. **Requires Human Judgment**: Fixes may affect business logic
2. **Incremental Improvement**: Let users choose fix priority
3. **Transparent Decisions**: Users understand impact of each fix

> **✅ AI MUST include "New Linter Analysis" section in final output report**:
> - List all newly added linters
> - Categorize by issue count and severity (recommended to fix vs defer)
> - Provide clear summary table
> - Refer to format in "Output Report Template" below

### 5. Supplement Configuration (Adjust Based on New Issues)

> **📌 AI Operation Requirements**:
> 1. **Core Principle**: **Do not modify existing code**, only make code pass by adjusting configuration (settings/disable/exclusions)
> 2. **Workflow**: Run lint → **Categorize and handle** → Add standardized TODO comments
> 3. **Universal Classification Logic** (based on keywords in linter description):
>
>    **Step 1: Get linter description**
>    ```bash
>    golangci-lint help <linter-name>
>    ```
>
>    **Step 2: Categorize based on keywords in description**
>
>    | Category | Keywords in Description | Action | Example Linters & Descriptions |
>    |----------|------------------------|--------|-------------------------------|
>    | **a. Configurable** | complexity, long, deeply, count, length, size, max, min, limit | Adjust settings | funlen: "Checks for **long** functions"<br>gocyclo: "Checks cyclomatic **complexity**"<br>nestif: "Reports **deeply** nested if"<br>dogsled: "Checks **too many** blank identifiers" |
>    | **b. Code Style-Manual Confirm** | style, format, naming, whitespace, align, order, declaration | disable + TODO | godot: "Check if comments end in period"<br>tagalign: "Check struct tags well **aligned**"<br>misspell: "Finds commonly **misspelled**"<br>varnamelen: "Checks variable name **length**" |
>    | **c. Critical Bugs-Suggest Fix** | bug, security, error, check, nil, unsafe, detect, inspects | disable + TODO | errcheck: "Checking for **unchecked errors**"<br>gosec: "**Inspects** source code for **security**"<br>staticcheck: "set of rules from staticcheck"<br>nilerr: "returns **nil** even if error is not **nil**" |
>    | **d. Cannot Modify** | (check specific error message, usually involves external constraints) | exclusions | canonicalheader: HTTP header spec (3rd-party APIs)<br>asciicheck: Non-ASCII symbols (Chinese function names) |
>    | **e. Can Fix Small** | (determined by actual issue count, < 5) | exclude-rules | Any linter with few issues |
>    | **f. New Features-Defer** | modern, new, latest, replace, simplification, feature | disable + TODO | modernize: "suggest **simplifications** using **modern** language"<br>exptostd: "replaced by **std** functions"<br>usestdlibvars: "use variables from **standard library**" |
>
>    **Step 3: Complete Decision Tree**
>    ```
>    1. Issue count < 5?
>       ├── Yes → Category e (can fix small): exclude-rules
>       └── No → Continue
>    2. Description has complexity/long/deeply/max/min/limit/length?
>       ├── Yes → Category a (configurable): Prioritize adjusting settings
>       └── No → Continue
>    3. Specific error caused by external constraints (3rd-party APIs/generated code/Chinese naming)?
>       ├── Yes → Category d (cannot modify): exclusions path exclusion
>       └── No → Continue
>    4. Description has modern/new/latest/replace/std/simplification?
>       ├── Yes → Category f (new features-defer): disable + TODO
>       └── No → Continue
>    5. Description has bug/security/error/check/nil/unsafe/detect/inspects?
>       ├── Yes → Category c (critical bugs-suggest fix): disable + TODO
>       └── No → Category b (code style-manual confirm): disable + TODO
>    ```
>
> 4. **Configuration Priority**:
>    - **1st Priority**: Adjust `settings` thresholds (e.g., funlen.lines, gocyclo.min-complexity)
>    - **2nd Priority**: Use `linters.exclusions` path exclusion (code that cannot be modified)
>    - **3rd Priority**: Use `issues.exclude-rules` rule-based exclusion (specific few issues)
>    - **Last Resort**: Completely `disable` (many issues and cannot be resolved via config)
> 5. **Avoid Duplicates**: Each linter appears only once, check for duplicates

**Adjust Configuration Based on Actual Errors**:

**Configuration Priority** (try in order):

| Priority | Action | Use Case |
|----------|--------|----------|
| **1️⃣ Highest** | Adjust `settings` thresholds | Linters with config options for complexity/length |
| **2️⃣ Second** | `linters.exclusions` path exclusion | Code that cannot be modified (generated/3rd-party) |
| **3️⃣ Then** | `issues.exclude-rules` rule exclusion | Specific few issues |
| **4️⃣ Last** | `disable` completely | Many issues and no config options |

**Category a: Configurable (Prioritize adjusting settings)**

```yaml
linters:
  settings:
    # Complexity linters: Prioritize adjusting thresholds rather than completely disabling
    funlen:
      lines: 100        # Default 60, raised to accommodate existing code
      statements: 60    # Default 40
    gocyclo:
      min-complexity: 25  # Default 15
    gocognit:
      min-complexity: 30  # Default 15
    nestif:
      min-complexity: 8   # Default 5
```

**Category b: Code Style-Manual Confirm**

```yaml
linters:
  disable:
    # TODO code style-confirm whether to disable: Does not affect functionality, requires manual confirmation
    - mnd              # magic numbers, style issues
    - wsl              # whitespace, code style
    - lll              # line length limit, style issues
    - godot            # comment period
    - tagalign         # struct tag alignment
    - goconst          # constant extraction suggestion
```

**Category c: Critical Bugs-Suggest Fix**

```yaml
linters:
  disable:
    # TODO critical bugs-suggest fix: Security and error handling issues, recommend gradual fixes
    - errcheck         # unchecked errors
    - gosec            # security checks
    - staticcheck      # static analysis
    - wrapcheck       # error wrapping
    - nilerr           # nil error checks
```

**Category d: Cannot Modify (Path Exclusion)**

```yaml
linters:
  exclusions:
    rules:
      # Generated code (cannot modify)
      - path: \.pb\.go|\.gen\.go|\.gen-\w+\.go|\.mock\.go
        linters: [all]

      # Third-party dependencies (cannot modify)
      - path: vendor/|third_party/
        linters: [all]

issues:
  exclude-rules:
    # Third-party APIs (cannot modify)
    - text: "non-canonical header"
      linters: [canonicalheader]

    # Chinese function names (business requirement, cannot modify)
    - text: "ID.*must match"
      linters: [asciicheck]
```

**Category e: Can Fix Small (Few Specific Issues)**

```yaml
issues:
  exclude-rules:
    # Specific business scenarios (cannot modify)
    - text: "G101: potential hardcoded credential"
      path: config/.*\.go
      linters: [gosec]

    # Relax for test files
    - path: _test\.go
      linters: [errcheck, gosec, contextcheck]
```

**Category f: New Features-Defer**

```yaml
linters:
  disable:
    # TODO new features-defer: Does not affect current code, can be selectively enabled later
    - modernize        # modern Go syntax suggestions
    - revive           # revive ruleset
    - gocritic         # code style suggestions
    - exhaustruct      # struct field completeness
```

### 6. Final Verification

```bash
# Run again to ensure pass
golangci-lint run --timeout=5m ./...
```

## Output Report Template

```markdown
# golangci-lint Upgrade Complete

## Version Information
- Old Version: v1.59.1
- New Version: v2.8.0
- Config Migration: v1 → v2

## 🆕 New Linter Analysis

This upgrade introduced **5 new linters**, here are handling recommendations:

### Recommended New Linters to Fix (issue count < 5 or critical issues)

| Linter | Issues | Category | Severity | Recommended Action |
|--------|--------|----------|----------|-------------------|
| **nilerr** | 3 | c. Critical Bugs-Suggest Fix | 🔴 High | **Recommended Fix** - May cause bugs |
| **contextcheck** | 2 | c. Critical Bugs-Suggest Fix | 🟡 Medium | **Recommended Fix** - May cause context loss |
| **sqlclosecheck** | 1 | c. Critical Bugs-Suggest Fix | 🟡 Medium | **Recommended Fix** - Resource leak risk |

**Fix Suggestion Example**:
```go
// ❌ Current code (issue detected by nilerr)
if err != nil {
    return nil, nil  // bug: returning nil error with nil value
}

// ✅ After fix
if err != nil {
    return nil, err
}
```

### New Linters Not to Fix (too many issues or style issues)

| Linter | Issues | Category | Action | Notes |
|--------|--------|----------|--------|-------|
| exhaustruct | 49 | f. New Features-Defer | disable + TODO | Struct field completeness, non-critical |
| testifylint | 8 | b. Code Style-Manual Confirm | disable + TODO | Test code style, can optimize later |

---

**Would you like me to help fix the above "recommended fix" issues?** Reply "fix" to start auto-fix.

## Configuration Changes

### Newly Disabled Linters
| Linter | Category | Reason |
|--------|----------|--------|
| errcheck | c. Critical Bugs-Suggest Fix | 13 issues: unchecked errors |
| gosec | c. Critical Bugs-Suggest Fix | 10 issues: security checks |
| mnd | b. Code Style-Manual Confirm | 47 issues: magic numbers |
| exhaustruct | f. New Features-Defer | 49 issues: new linter, too many issues |
| testifylint | b. Code Style-Manual Confirm | 8 issues: new linter, test style |

### Newly Adjusted Settings
| Linter | Config | Old Value | New Value |
|--------|--------|-----------|-----------|
| funlen | lines | 60 | 100 |
| gocyclo | min-complexity | 15 | 25 |

### Newly Added Exclusions
- Generated code: `.pb.go`, `.gen.go`, `.mock.go`

## Next Steps
1. ✅ **Small issues from new linters**: Recommend prioritizing fixes (see "recommended fix" list above)
2. New code must pass lint
3. Prioritize fixing issues in "c. Critical Bugs-Suggest Fix" category
4. After code improvements, gradually reduce complexity thresholds
```

## Notes

1. **Core Principle**: **Do not modify existing code**, only adjust configuration to make code pass lint
2. **Special Handling for New Linters**:
   - For new linters with few issues (< 5) or critical issues (security/error handling), **provide fix suggestions**
   - Users can choose whether to fix, AI will not automatically modify code
   - This does not conflict with "do not modify existing code" principle, as these are "new rule" issues
3. **v2 Configuration Format**: Must add `version: 2`, use `linters.default` instead of `enable-all`
4. **Preserve Existing Configuration**: Keep all disable/exclude settings during migration
5. **Backup Before Upgrade**: Recommended to backup `.golangci.yml` configuration file
6. **Gradual Tightening**: Can gradually reduce settings thresholds later to improve code quality

## Related Resources

- [Official Installation Docs](https://golangci-lint.run/docs/welcome/install/local/)
- [Configuration Migration Guide](https://golangci-lint.run/docs/configuration/migrate/)
- [v2 Changes](https://golangci-lint.run/usage/v2/)
