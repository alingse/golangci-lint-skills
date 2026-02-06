---
name: upgrade-golangci-lint
description: 升级 golangci-lint 到最新版本，自动迁移配置（v1→v2），运行 lint 并补充配置让代码通过。核心原则：只调整配置，不修改代码。
---

# golangci-lint 版本升级

升级 golangci-lint 到最新版本，处理配置迁移，并调整配置让存量代码通过 lint 检查。

> **核心原则**：通过调整配置让存量代码通过 lint，**不修改现有代码**。只影响新代码。

## 何时使用

- golangci-lint 版本过旧，需要升级到最新版本
- CI 中 lint 失败，发现是版本问题
- 想要使用新版本的 linter 功能

## 执行流程

### 1. 检测当前状态

```bash
# 检查当前版本
golangci-lint --version

# 检查配置文件
ls -la .golangci.yml 2>/dev/null && echo "已有配置" || echo "无配置"

# 检查配置格式版本（v1 还是 v2）
grep "^version:" .golangci.yml 2>/dev/null && echo "v2 配置" || echo "v1 配置（需迁移）"

# 检测安装方式
if command -v brew &> /dev/null && brew list golangci-lint &> /dev/null 2>/dev/null; then
    INSTALL_METHOD="brew"
elif command -v golangci-lint &> /dev/null; then
    INSTALL_METHOD="binary"
else
    echo "golangci-lint 未安装，请先运行 setup-golangci-lint"
    exit 1
fi
```

### 2. 升级二进制

根据检测到的安装方式执行升级：

| 安装方式 | 升级命令 |
|---------|---------|
| **二进制** | `curl -sSfL https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest` |
| **Homebrew** | `brew upgrade golangci-lint` |
| **Alpine (无 curl)** | `wget -O- -nv https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest` |

```bash
# 二进制方式升级（推荐）
curl -sSfL https://golangci-lint.run/install.sh | sh -s -- -b $(go env GOPATH)/bin latest

# 验证升级成功
golangci-lint --version
```

### 3. 配置迁移（v1 → v2）

如果检测到是 v1 配置（配置文件中无 `version:` 字段），执行迁移：

```bash
# 自动迁移 v1 配置到 v2 格式
golangci-lint migrate --skip-validation
```

**说明**：
- `--skip-validation`: 跳过验证，直接转换（推荐）
- `migrate` 命令会自动：
  - 添加 `version: 2`
  - 将 `enable-all/disable-all` 转换为 `linters.default`
  - 将 `linters-settings` 改为 `linters.settings`
  - 将 `issues.exclude-rules` 改为 `linters.exclusions.rules`
  - **保留所有现有的 disable 和 exclude 配置**

迁移后手动检查配置，确保：
- `linters.default` 值符合预期（standard/all/none/fast）
- 禁用的 linters 仍有 `# TODO fix later by human` 注释
- 复杂度阈值已调高并标注 `# TODO reduce this`

```bash
# 验证配置格式
golangci-lint config verify
```

### 4. 运行 lint 检查

```bash
# 运行 lint
golangci-lint run --timeout=5m ./...
```

### 4.5. 识别和处理新增 linter

> **🆕 新增 linter 特殊处理**：
> 升级版本后，新版本可能引入之前不存在的 linter。对于这些新增的 linter，处理策略与存量 linter 有所不同。

**识别新增 linter**：

```bash
# 保存旧版本的 linter 列表（升级前执行）
golangci-lint help linters > /tmp/old_linters.txt

# 升级后对比
golangci-lint help linters > /tmp/new_linters.txt
diff /tmp/old_linters.txt /tmp/new_linters.txt
```

或者在升级后直接查看错误输出，识别之前未见过的 linter 名称。

**新增 linter 的处理策略**：

| 问题数量 | 分类 | 处理方式 | 说明 |
|---------|------|----------|------|
| **< 5 个** | 任何分类 | **提供修复建议** | 少量问题，值得修复 |
| **5-20 个** | c. 严重bug | **提供修复建议** | 安全/错误问题，建议修复 |
| **5-20 个** | a. 可配置调优 | 调整 settings | 优先调整阈值 |
| **5-20 个** | b/d/f 其他 | disable + TODO | 按原有逻辑处理 |
| **> 20 个** | 任何分类 | disable + TODO | 太多问题，暂不处理 |

**提供修复建议的格式**：

```markdown
## 🆕 新增 Linter 修复建议

以下 linter 是本次升级新增的，且问题数量较少，建议修复：

### Linter: nilerr
- **问题数量**: 3 个
- **严重性**: 高（可能导致 bug）
- **修复建议**:
  1. `pkg/api/handler.go:45` - 检查错误返回是否正确
  2. `pkg/service/user.go:123` - 确保错误不为 nil 时返回非 nil
  3. `pkg/db/client.go:67` - 修复错误处理逻辑

### Linter: contextcheck
- **问题数量**: 2 个
- **严重性**: 中（可能导致上下文丢失）
- **修复建议**:
  1. `pkg/api/client.go:89` - 添加 context 参数
  2. `pkg/service/order.go:156` - 传递 context 到下游

---

**是否需要我帮你修复这些问题？** 回复 "fix" 开始修复。
```

**为什么不自动修复新增 linter 的问题**：
1. **需要人工判断**：修复可能影响业务逻辑
2. **渐进式改进**：让用户选择修复优先级
3. **透明决策**：用户了解每个修复的影响

### 5. 补充配置（根据新问题调整）

> **📌 AI 操作要求**：
> 1. **核心原则**：**不修改存量代码**，只通过配置调整（settings/disable/exclusions）让代码通过
> 2. **工作流程**：运行 lint → **分类处理** → 添加规范的 TODO 注释
> 3. **通用分类判断逻辑**（基于 linter 描述的关键词）：
>
>    **第一步：获取 linter 描述**
>    ```bash
>    golangci-lint help <linter-name>
>    ```
>
>    **第二步：根据描述中的关键词分类**
>
>    | 分类 | 描述中的关键词 | 处理方式 | 示例 linter 及其描述 |
>    |------|---------------|----------|---------------------|
>    | **a. 可配置调优** | complexity, long, deeply, count, length, size, max, min, limit | 调整 settings | funlen: "Checks for **long** functions"<br>gocyclo: "Checks cyclomatic **complexity**"<br>nestif: "Reports **deeply** nested if"<br>dogsled: "Checks **too many** blank identifiers" |
>    | **b. 代码风格-人工确认** | style, format, naming, whitespace, align, order, declaration | disable + TODO | godot: "Check if comments end in period"<br>tagalign: "Check struct tags well **aligned**"<br>misspell: "Finds commonly **misspelled**"<br>varnamelen: "Checks variable name **length**" |
>    | **c. 严重bug-建议修复** | bug, security, error, check, nil, unsafe, detect, inspects | disable + TODO | errcheck: "Checking for **unchecked errors**"<br>gosec: "**Inspects** source code for **security**"<br>staticcheck: "set of rules from staticcheck"<br>nilerr: "returns **nil** even if error is not **nil**" |
>    | **d. 无法修改** | (需看具体错误消息，通常涉及外部约束) | exclusions | canonicalheader: HTTP header 规范（第三方接口）<br>asciicheck: 非 ASCII 符号（中文函数名） |
>    | **e. 可小修** | (由实际问题数量决定，< 5 个) | exclude-rules | 任何 linter，如果只有少量问题 |
>    | **f. 新特性-后续考虑** | modern, new, latest, replace, simplification, feature | disable + TODO | modernize: "suggest **simplifications** using **modern** language"<br>exptostd: "replaced by **std** functions"<br>usestdlibvars: "use variables from **standard library**" |
>
>    **第三步：完整决策树**
>    ```
>    1. 问题数量 < 5？
>       ├── 是 → 分类 e (可小修): exclude-rules
>       └── 否 → 继续
>    2. 描述中有 complexity/long/deeply/max/min/limit/length？
>       ├── 是 → 分类 a (可配置调优): 优先调整 settings
>       └── 否 → 继续
>    3. 具体错误是外部约束导致（第三方接口/生成代码/中文命名）？
>       ├── 是 → 分类 d (无法修改): exclusions 路径排除
>       └── 否 → 继续
>    4. 描述中有 modern/new/latest/replace/std/simplification？
>       ├── 是 → 分类 f (新特性-后续考虑): disable + TODO
>       └── 否 → 继续
>    5. 描述中有 bug/security/error/check/nil/unsafe/detect/inspects？
>       ├── 是 → 分类 c (严重bug-建议修复): disable + TODO
>       └── 否 → 分类 b (代码风格-人工确认): disable + TODO
>    ```
>
> 4. **配置优先级**：
>    - **第一优先**：调整 `settings` 阈值（如 funlen.lines, gocyclo.min-complexity）
>    - **第二优先**：使用 `linters.exclusions` 路径排除（无法修改的代码）
>    - **第三优先**：使用 `issues.exclude-rules` 按特定规则排除（少量问题）
>    - **最后选择**：完全 `disable`（大量问题且无法通过配置解决）
> 5. **避免重复**：每个 linter 只出现一次，检查是否有重复项

**根据实际错误调整配置**：

**配置优先级**（按顺序尝试）：

| 优先级 | 处理方式 | 适用场景 |
|--------|----------|----------|
| **1️⃣ 最高** | 调整 `settings` 阈值 | 复杂度、长度类有配置项的 linter |
| **2️⃣ 其次** | `linters.exclusions` 路径排除 | 无法修改的代码（生成/第三方） |
| **3️⃣ 然后** | `issues.exclude-rules` 按规则排除 | 少量特定问题 |
| **4️⃣ 最后** | `disable` 完全禁用 | 大量问题且无配置选项 |

**分类 a: 可配置调优（优先调整 settings）**

```yaml
linters:
  settings:
    # 复杂度类 linter：优先调整阈值，而非完全禁用
    funlen:
      lines: 100        # 默认 60，调高以适应存量代码
      statements: 60    # 默认 40
    gocyclo:
      min-complexity: 25  # 默认 15
    gocognit:
      min-complexity: 30  # 默认 15
    nestif:
      min-complexity: 8   # 默认 5
```

**分类 b: 代码风格-人工确认**

```yaml
linters:
  disable:
    # TODO 代码风格-确认是否禁用: 不影响功能，需人工确认处理方式
    - mnd              # magic numbers，风格问题
    - wsl              # whitespace，代码风格
    - lll              # 行长度限制，风格问题
    - godot            # 注释句点
    - tagalign         # struct tag 对齐
    - goconst          # 常量提取建议
```

**分类 c: 严重bug-建议修复**

```yaml
linters:
  disable:
    # TODO 严重bug-建议修复: 安全和错误处理问题，建议逐步修复
    - errcheck         # 未检查错误
    - gosec            # 安全检查
    - staticcheck      # 静态分析
    - wrapcheck       # 错误包装
    - nilerr           # nil 错误检查
```

**分类 d: 无法修改（路径排除）**

```yaml
linters:
  exclusions:
    rules:
      # 生成代码（无法修改）
      - path: \.pb\.go|\.gen\.go|\.gen-\w+\.go|\.mock\.go
        linters: [all]

      # 第三方依赖（无法修改）
      - path: vendor/|third_party/
        linters: [all]

issues:
  exclude-rules:
    # 第三方接口（不可修改）
    - text: "non-canonical header"
      linters: [canonicalheader]

    # 中文函数名（业务需求，不可修改）
    - text: "ID.*must match"
      linters: [asciicheck]
```

**分类 e: 可小修（少量特定问题）**

```yaml
issues:
  exclude-rules:
    # 特定业务场景（无法修改）
    - text: "G101: potential hardcoded credential"
      path: config/.*\.go
      linters: [gosec]

    # 测试文件放宽
    - path: _test\.go
      linters: [errcheck, gosec, contextcheck]
```

**分类 f: 新特性-后续考虑**

```yaml
linters:
  disable:
    # TODO 新特性-后续考虑: 不影响当前代码，后续可选择性启用
    - modernize        # 现代 Go 语法建议
    - revive           # revive 规则集
    - gocritic         # 代码风格建议
    - exhaustruct      # 结构体字段完整性
```

### 6. 最终验证

```bash
# 再次运行确保通过
golangci-lint run --timeout=5m ./...
```

## 输出报告模板

```markdown
# golangci-lint 升级完成

## 版本信息
- 旧版本: v1.59.1
- 新版本: v2.8.0
- 配置迁移: v1 → v2

## 🆕 新增 Linter 分析

本次升级引入了 **5 个新 linter**，以下是处理建议：

### 建议修复的新增 linter（问题数量 < 5 或属于严重问题）

| Linter | 问题数 | 分类 | 严重性 | 建议操作 |
|--------|--------|------|--------|----------|
| **nilerr** | 3 | c. 严重bug-建议修复 | 🔴 高 | **建议修复** - 可能导致 bug |
| **contextcheck** | 2 | c. 严重bug-建议修复 | 🟡 中 | **建议修复** - 可能导致上下文丢失 |
| **sqlclosecheck** | 1 | c. 严重bug-建议修复 | 🟡 中 | **建议修复** - 资源泄漏风险 |

**修复建议示例**：
```go
// ❌ 当前代码（nilerr 检测到的问题）
if err != nil {
    return nil, nil  // bug: 返回了 nil error 但值也是 nil
}

// ✅ 修复后
if err != nil {
    return nil, err
}
```

### 暂不修复的新增 linter（问题数量过多或为风格问题）

| Linter | 问题数 | 分类 | 处理方式 | 说明 |
|--------|--------|------|----------|------|
| exhaustruct | 49 | f. 新特性-后续考虑 | disable + TODO | 结构体字段完整性，非关键问题 |
| testifylint | 8 | b. 代码风格-人工确认 | disable + TODO | 测试代码风格，可后续优化 |

---

**是否需要我帮你修复上述"建议修复"的问题？** 回复 "fix" 开始自动修复。

## 配置变更

### 新增 disable 的 linters
| Linter | 分类 | 原因 |
|--------|------|------|
| errcheck | c. 严重bug-建议修复 | 13 issues: 未检查错误 |
| gosec | c. 严重bug-建议修复 | 10 issues: 安全检查 |
| mnd | b. 代码风格-人工确认 | 47 issues: magic numbers |
| exhaustruct | f. 新特性-后续考虑 | 49 issues: 新增 linter，问题过多 |
| testifylint | b. 代码风格-人工确认 | 8 issues: 新增 linter，测试风格 |

### 新增 settings 调整
| Linter | 配置项 | 旧值 | 新值 |
|--------|--------|------|------|
| funlen | lines | 60 | 100 |
| gocyclo | min-complexity | 15 | 25 |

### 新增 exclusions
- 生成代码: `.pb.go`, `.gen.go`, `.mock.go`

## 下一步
1. ✅ **新增 linter 的小量问题**：建议优先修复（上述"建议修复"列表）
2. 新代码必须通过 lint
3. 优先修复 "c. 严重bug-建议修复" 分类的问题
4. 改进代码后，可逐步降低复杂度阈值
```

## 注意事项

1. **核心原则**：**不修改存量代码**，只调整配置让代码通过 lint
2. **新增 linter 特殊处理**：
   - 对于新增 linter 的小量问题（< 5 个）或严重问题（安全/错误处理），**会提供修复建议**
   - 用户可以选择是否修复，AI 不会自动修改代码
   - 这与"存量代码不修改"原则不冲突，因为这是"新规则"的问题
3. **v2 配置格式**：必须添加 `version: 2`，使用 `linters.default` 而非 `enable-all`
4. **保留现有配置**：迁移时保留所有 disable/exclude 设置
5. **升级前备份**：建议备份 `.golangci.yml` 配置文件
6. **逐步收紧**：后续可以逐步降低 settings 阈值，提高代码质量

## 相关资源

- [官方安装文档](https://golangci-lint.run/docs/welcome/install/local/)
- [配置迁移指南](https://golangci-lint.run/docs/configuration/migrate/)
- [v2 变更说明](https://golangci-lint.run/usage/v2/)
