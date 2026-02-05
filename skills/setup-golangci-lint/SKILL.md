---
name: setup-golangci-lint
description: 为 Go 项目快速设置 golangci-lint 环境。自动检测版本、创建配置、集成 CI、生成智能忽略规则，确保存量代码安全通过。
---

# golangci-lint 环境配置

为 Go 项目快速搭建完整的 golangci-lint 环境。

> **核心原则**：通过调整配置让存量代码通过 lint，**不修改现有代码**。只影响新代码。

## 何时使用

- 初始化 Go 项目需要添加 linter
- 为现有项目添加代码质量检查
- 集成 lint 到 CI/CD 流程

## 执行流程

### 1. 检测环境

```bash
# 检查 go.mod 中的 Go 版本（优先）
cat go.mod | grep "^go "

# 检查系统 Go 版本（参考）
go version

# 检查现有配置
ls .golangci.yml 2>/dev/null && echo "已有配置" || echo "无配置"

# 检查 CI 配置
ls .gitlab-ci.yml .github/workflows/*.yml .circleci/config.yml 2>/dev/null

# 检查 Makefile
ls Makefile 2>/dev/null && echo "有 Makefile" || echo "无 Makefile"
```

### 2. 选择版本

根据 `go.mod` 中的 Go 版本选择：

| go.mod 版本 | golangci-lint 版本 |
|-------------|-------------------|
| < 1.20 | v1.x |
| >= 1.20 | v2.x (推荐) |

### 3. 安装

```bash
# v2 (推荐)
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin latest

# v1
# curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.59.1

# 验证
$(go env GOPATH)/bin/golangci-lint --version
```

### 3.5. 配置迁移（已有 v1 配置时）

如果项目已有 `.golangci.yml` 且是 v1 格式，可以使用自动迁移工具：

```bash
# 自动迁移 v1 配置到 v2 格式
$(go env GOPATH)/bin/golangci-lint migrate --skip-validation

# 说明：
# --skip-validation: 跳过验证，直接转换（推荐）
# migrate 命令会自动：
#   - 添加 version: 2
#   - 将 enable-all/disable-all 转换为 linters.default
#   - 将 linters-settings 改为 linters.settings
#   - 将 issues.exclude-rules 改为 linters.exclusions.rules
```

迁移后手动检查配置，确保：
- `linters.default` 值符合预期（standard/all/none/fast）
- 禁用的 linters 仍有 `# TODO fix later by human` 注释
- 复杂度阈值已调高并标注 `# TODO reduce this`

### 4. 创建配置

如果已有 `.golangci.yml`，跳过此步骤。

> **📌 AI 操作要求**：
> 1. **必须**先访问官方文档确认最新格式：https://golangci-lint.run/docs/configuration/file/
> 2. **核心原则**：**不修改存量代码**，只通过配置调整（settings/disable/exclusions）让代码通过
> 3. **Formatters 处理**：
>    - 先运行 `gofmt -l .` 和 `goimports -l .` 检查代码格式
>    - 如果都无输出 → 启用 formatters（取消注释）
>    - 如果有输出 → 保持注释，在对应项上方添加 `# TODO 格式化代码并开启 - 当前有 X 个文件不符合`
> 4. **工作流程**：创建最小配置 → 运行 lint → **分类处理** → 添加规范的 TODO 注释
> 5. **通用分类判断逻辑**（基于 linter 描述的关键词）：
>
>    **第一步：获取 linter 描述**
>    ```bash
>    $(go env GOPATH)/bin/golangci-lint help <linter-name>
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
>    **示例演示**：
>
>    | Linter | 描述 | 关键词 | 分类 |
>    |--------|------|--------|------|
>    | cyclop | "Checks function **complexity**" | complexity | a |
>    | gocritic | "checks for **bugs**, performance and **style**" | bugs (优先) | c |
>    | revive | "replacement of golint" | (无明确关键词) | b |
>    | bodyclose | "Checks whether response body is **closed successfully**" | error/check | c |
>    | goconst | "Finds repeated strings that could be replaced by a **constant**" | (style) | b |
>    | fatcontext | "Detects **nested contexts**" | nested → complexity | a |
>
> 6. **配置优先级**：
>    - **第一优先**：调整 `settings` 阈值（如 funlen.lines, gocyclo.min-complexity）
>    - **第二优先**：使用 `linters.exclusions` 路径排除（无法修改的代码）
>    - **第三优先**：使用 `issues.exclude-rules` 按特定规则排除（少量问题）
>    - **最后选择**：完全 `disable`（大量问题且无法通过配置解决）
> 7. **避免重复**：每个 linter 只出现一次，检查是否有重复项

**v2 最小配置模板 (.golangci.yml)**

```yaml
version: "2"

run:
  concurrency: 4
  timeout: 5m
  skip-dirs: [vendor, third_party, testdata, examples, gen-go]
  tests: true

output:
  format: colored-line-number
  print-linter-name: true

linters:
  default: all

formatters:
  enable:
    # TODO 格式化代码并开启 - 当前有 17 个文件不符合 goimports
    # - gofmt
    # - goimports
```

> **Formatters 处理**：先运行 `gofmt -l .` 和 `goimports -l .` 检查，如果都无输出则取消注释启用。

> **说明**：
> - 从最小配置开始，不做任何预设的 disable
> - 运行后根据错误逐步添加 `disable` 和 `exclusions`
> - 常见需要调整的 linters（按需添加）：
>   - `mnd` (magic numbers) → style issues, 可永久忽略
>   - `wsl` (whitespace) → style issues, 可永久忽略
>   - `lll` (line length) → style issues, 可永久忽略
>   - `err113` → error handling, 临时忽略并标注 TODO

**v1 配置模板 (.golangci.yml)**

> **注意**：v1 不需要 `version` 字段。推荐使用 v2。

```yaml
run:
  concurrency: 4
  timeout: 5m
  skip-dirs: [vendor, testdata]

linters:
  enable-all: true
```

> 运行后根据错误再添加 `disable` 和 `exclude-rules`。

### 5. 集成 CI

**GitLab CI**

```yaml
lint:
  stage: test
  image: golang:1.23-alpine
  script:
    - curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin latest
    - $(go env GOPATH)/bin/golangci-lint run --timeout=5m ./...
```

**GitHub Actions**

```yaml
- name: golangci-lint
  uses: golangci/golangci-lint-action@v6
  with:
    version: latest
```

### 6. 更新 Makefile（可选）

```makefile
lint:  ## 运行 lint 检查
	golangci-lint run --timeout=5m ./...

lint-fix:  ## 自动修复问题
	golangci-lint run --fix --timeout=5m ./...
```

### 7. 运行并按需调整

```bash
# 运行 lint
$(go env GOPATH)/bin/golangci-lint run --timeout=5m ./...
```

> **重要**：通过调整配置解决问题，**不修改存量代码**。

**根据错误输出分析**：

1. **Linter 名称错误**（如 `unknown linters`）：
   ```bash
   # 查看支持的 linters
   $(go env GOPATH)/bin/golangci-lint help linters
   ```
   - `gomnd` → `mnd`
   - `goerr113` → `err113`
   - `execinquery` → 已移除

2. **配置格式错误**：
   - `output.formats` 需要 map 格式，不是 list
   - 使用 `output.format` 单数形式

3. **根据实际错误调整配置**：

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
       # 如果调整阈值后仍有问题，再考虑 disable
   ```

   **分类 b: 代码风格-人工确认**

   ```yaml
   linters:
     disable:
       # TODO 代码风格-确认是否禁用: 不影响功能，需人工确认处理方式
       - mnd              # 47 issues: magic numbers，风格问题
       - wsl              # 39 issues: whitespace，已弃用，被 wsl_v5 替代
       - wsl_v5           # 50 issues: whitespace v5，代码风格
       - lll              # 46 issues: 行长度限制，风格问题
       - godot            # 3 issues: 注释句点
       - tagalign         # 12 issues: struct tag 对齐
       - tagliatelle      # 50 issues: struct tag 命名规范
       - whitespace       # 3 issues: 空格问题
       - goconst          # 7 issues: 常量提取建议
       - prealloc         # 2 issues: 预分配切片建议
       - nakedret         # 1 issue: naked return
       - nlreturn         # 9 issues: return 前空行
       - inamedparam      # 1 issue: 命名参数
       - varnamelen       # 23 issues: 变量名长度
       - nonamedreturns   # 2 issues: 命名返回值
       - paralleltest     # 47 issues: 并行测试建议
       - testpackage      # 3 issues: 测试包命名
       - testifylint      # 3 issues: testify 使用规范
       - ireturn          # 4 issues: 接口返回
       - intrange         # 3 issues: int range 循环
       - nilnil           # 3 issues: nil 接口
       - nilnesserr       # 3 issues: nil 错误检查
       - noinlineerr      # 3 issues: 内联错误
       - gosmopolitan     # 3 issues: 国际化
       - usestdlibvars    # 1 issue: 标准库常量
       - unparam          # 1 issue: 未使用参数
       - perfsprint       # 9 issues: 性能打印建议
   ```

   **分类 c: 严重bug-建议修复**

   ```yaml
   linters:
     disable:
       # TODO 严重bug-建议修复: 安全和错误处理问题，建议逐步修复
       - errcheck         # 13 issues: 未检查错误
       - gosec            # 10 issues: 安全检查
       - staticcheck      # 8 issues: 静态分析
       - rowserrcheck     # 3 issues: database rows.Err 检查
       - errchkjson       # 4 issues: JSON 错误检查
       - errorlint        # 1 issue: 错误处理规范
       - errname          # 2 issues: 错误变量命名
       - wrapcheck       # 44 issues: 错误包装
       - noctx            # 3 issues: context 参数检查
       - forcetypeassert  # 2 issues: 强制类型断言
       - contextcheck     # 4 issues: context 传递检查
       - nilerr           # nil 错误检查
       - govet            # 1 issue: vet 检查
       - unused           # 16 issues: 未使用变量/包
       - err113           # 21 issues: 动态错误定义
       - ineffassign      # 1 issue: 无效赋值
       - sqlclosecheck    # 3 issues: SQL 未关闭检查
       - wastedassign     # 2 issues: 浪费赋值
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

         # 示例代码（可选择性修改）
         - path: examples/
           linters: [all]

   issues:
     exclude-rules:
       # 第三方接口（不可修改）
       - text: "non-canonical header"
         linters: [canonicalheader]

       # 中文函数名（业务需求，不可修改）
       - text: "ID.*must match"
         linters: [asciicheck]

       # 内部项目使用内部包（业务需求）
       - text: "import of package"
         linters: [depguard]
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

       # main 函数放宽
       - path: cmd/
         linters: [gocyclo, funlen, errcheck]
   ```

   **分类 f: 新特性-后续考虑**

   ```yaml
   linters:
     disable:
       # TODO 新特性-后续考虑: 不影响当前代码，后续可选择性启用
       - modernize        # 12 issues: 现代 Go 语法建议
       - revive           # 48 issues: revive 规则集
       - gocritic         # 10 issues: 代码风格建议
       - godoclint        # 3 issues: godoc 格式
       - gomoddirectives  # module 指令检查
       - dogsled          # 空白标识符数量
       - embeddedstructfieldcheck # 2 issues: 嵌入字段空行
       - exhaustruct      # 49 issues: 结构体字段完整性
       - forbidigo        # 6 issues: 禁止特定函数
       - gochecknoglobals # 15 issues: 全局变量检查
       - gochecknoinits   # 2 issues: init 函数检查
       - godox            # 2 issues: TODO/FIXME 标记
       - nolintlint       # 1 issue: nolint 注释检查
   ```

**目标**：存量代码能通过 lint，**不修改存量代码**，新代码必须遵守规则。

### 8. 最终验证

```bash
# 再次运行确保通过
$(go env GOPATH)/bin/golangci-lint run --timeout=5m ./...

# 或使用 make
make lint
```

## 输出报告模板

```markdown
# golangci-lint 配置完成

## 环境信息
- Go 版本 (go.mod): go 1.23
- golangci-lint 版本: v2.8.0

## 配置
- 创建: .golangci.yml (version: 2)
- 初始配置: linters.default: all (最小配置)

## 问题处理

### 处理优先级

| 优先级 | 方式 | Linters |
|--------|------|---------|
| 1️⃣ 调整 settings | 阈值调优 | funlen, gocyclo, gocognit, nestif |
| 2️⃣ exclusions | 路径排除 | 生成代码、第三方依赖、特定接口 |
| 3️⃣ exclude-rules | 规则排除 | 测试文件、特定业务场景 |
| 4️⃣ disable | 完全禁用 | 大量问题的 linter |

### Linter 分类统计

| 分类 | 数量 | 说明 |
|------|------|------|
| a. 可配置调优 | 4 | 优先调整 settings 阈值 |
| b. 代码风格-人工确认 | 26 | 风格问题，需确认是否禁用 |
| c. 严重bug-建议修复 | 18 | 安全/错误处理，建议修复 |
| d. 无法修改 | - | 路径排除（外部约束） |
| e. 可小修 | - | issues 排除（少量问题） |
| f. 新特性-后续考虑 | 13 | 新特性，后续考虑 |

### Settings 阈值调整
```yaml
linters:
  settings:
    funlen:
      lines: 100        # 默认 60
      statements: 60    # 默认 40
    gocyclo:
      min-complexity: 25  # 默认 15
    gocognit:
      min-complexity: 30  # 默认 15
    nestif:
      min-complexity: 8   # 默认 5
```

### 路径排除（无法修改）
- 生成代码: `.pb.go`, `.gen.go`, `.mock.go`
- 第三方依赖: `vendor/`, `third_party/`
- 示例代码: `examples/`

### Issues 排除（特定场景）
- 第三方接口: `non-canonical header` (canonicalheader)
- 中文函数名: `ID.*must match` (asciicheck)
- 测试文件: 放宽 errcheck, gosec, contextcheck

## 下一步
1. 新代码必须通过 lint
2. 优先修复 "c. 严重bug-建议修复" 分类的问题
3. 改进代码后，可逐步降低复杂度阈值
4. 定期运行 `make lint` 检查
```

## 注意事项

1. **核心原则**：**不修改存量代码**，只调整配置让代码通过 lint
2. **v2 配置格式**：必须添加 `version: 2`，使用 `linters.default` 而非 `enable-all`
3. **已有配置不修改**
4. **风格问题可永久忽略**：函数长度、圈复杂度等不影响功能
5. **严重问题需提醒修复**：errcheck、gosec、staticcheck 等（通过配置排除，不直接修复）
6. **新代码不得绕过检查**
7. **CI 需足够资源**：建议至少 2GB 内存

## 相关资源

- [官方文档](https://golangci-lint.run/)
- [支持的 Linters](https://golangci-lint.run/usage/linters/)
