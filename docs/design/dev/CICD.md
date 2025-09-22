### **文档一：`.github` 文件夹结构与说明**



这是我们仓库中`.github`文件夹的建议结构。您只需要在您的仓库根目录下创建这些文件和文件夹即可。

```
.github/
├── ISSUE_TEMPLATE/          # (可选, 推荐) Issue模板
│   ├── bug_report.md
│   └── feature_request.md
├── pull_request_template.md # (可选, 推荐) Pull Request模板
└── workflows/               # 核心：GitHub Actions工作流
    └── ci.yml
```

------



### **文档二：Pull Request 模板 (`pull_request_template.md`)**



这份文件是推荐的最佳实践。当团队成员创建Pull Request时，描述会自动填充这个模板，引导他们提供清晰、完整的信息。

Code snippet

```markdown
# .github/pull_request_template.md

## 关联的Issue (Related Issue)

## PR类型 (PR Type)

- [ ] Bug修复 (Bug fix)
- [ ] 新功能 (New feature)
- [ ] 文档更新 (Documentation update)
- [ ] 代码重构 (Code refactoring)
- [ ] CI/CD配置 (CI/CD configuration)
- [ ] 其他 (Other)

## PR做了什么 (What does this PR do?)

## 如何手动测试 (How to manually test it?)

## 审查者注意事项 (Notes for reviewers)
```

------



### **文档三：核心CI工作流 (`ci.yml`)**



这是我们整个CI/CD的核心。这份文件定义了当一个Pull Request被创建或更新时，需要自动执行的所有检查。它被设计为高度并行和智能的，只会检查发生变化的部分。

**请注意：** YAML文件对缩进极其敏感，请确保复制时格式正确。

Code snippet

```yaml
# .github/workflows/ci.yml

# 工作流名称
name: ARF Continuous Integration

# 触发条件：当有代码推送到main分支，或者创建/更新一个指向main分支的Pull Request时
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  # 第一个任务：确定哪些文件发生了变化
  changes:
    runs-on: ubuntu-latest
    outputs:
      # 输出将用于后续任务的判断
      go: ${{ steps.filter.outputs.go }}
      python: ${{ steps.filter.outputs.python }}
      rust: ${{ steps.filter.outputs.rust }}
      cpp: ${{ steps.filter.outputs.cpp }}
      proto: ${{ steps.filter.outputs.proto }}
      docs: ${{ steps.filter.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            go:
              - '**/*.go'
              - 'go.mod'
              - 'go.sum'
            python:
              - '**/*.py'
              - '**/requirements.txt'
            rust:
              - '**/*.rs'
              - '**/Cargo.toml'
              - '**/Cargo.lock'
            cpp:
              - '**/*.cpp'
              - '**/*.h'
              - '**/CMakeLists.txt'
            proto:
              - 'api/**/*.proto'
            docs:
              - 'docs/**/*.md'
              - '*.md'

  # Go语言相关任务
  lint-test-go:
    needs: changes
    if: needs.changes.outputs.go == 'true'
    name: Lint & Test Go
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      - name: Go Lint
        run: |
          go install honnef.co/go/tools/cmd/staticcheck@latest
          staticcheck ./...
          go vet ./...
      - name: Go Test
        run: go test -v -race ./...

  # Python语言相关任务
  lint-test-python:
    needs: changes
    if: needs.changes.outputs.python == 'true'
    name: Lint & Test Python
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: pip install black isort flake8 pytest
      - name: Python Lint
        run: |
          black --check .
          isort --check-only .
          flake8 .
      - name: Python Test
        run: pytest

  # Rust语言相关任务
  lint-test-rust:
    needs: changes
    if: needs.changes.outputs.rust == 'true'
    name: Lint & Test Rust
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Rust Lint
        run: cargo clippy -- -D warnings
      - name: Rust Test
        run: cargo test

  # C++语言相关任务
  build-test-cpp:
    needs: changes
    if: needs.changes.outputs.cpp == 'true'
    name: Build & Test C++
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: sudo apt-get update && sudo apt-get install -y build-essential cmake
      - name: C++ Build
        run: |
          cmake -B build
          cmake --build build
      - name: C++ Test
        run: |
          cd build
          ctest

  # Protobuf接口相关任务
  lint-proto:
    needs: changes
    if: needs.changes.outputs.proto == 'true'
    name: Lint Protobuf
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bufbuild/buf-setup-action@v1
      - name: Buf Lint
        run: buf lint api/
      - name: Buf Breaking Change Detection
        # 检查与main分支的最新版本相比，是否有不兼容的API变更
        run: |
          git config --global --add safe.directory /github/workspace
          buf breaking --against ".git#branch=main,ref=HEAD~1" api/

  # 最终状态检查任务
  ci-status:
    # 只有当所有前置任务都成功后，这个任务才会运行
    needs: [lint-test-go, lint-test-python, lint-test-rust, build-test-cpp, lint-proto]
    # "if: always()" 确保即使有任务被跳过（因为文件未更改），这个任务也能运行
    if: always()
    name: CI Status
    runs-on: ubuntu-latest
    steps:
      - name: Check status of all jobs
        # 如果任何一个依赖任务失败了，这个步骤就会失败，从而让整个PR检查失败
        run: |
          if [[ "${{ needs.lint-test-go.result }}" == "failure" || \
                "${{ needs.lint-test-python.result }}" == "failure" || \
                "${{ needs.lint-test-rust.result }}" == "failure" || \
                "${{ needs.build-test-cpp.result }}" == "failure" || \
                "${{ needs.lint-proto.result }}" == "failure" ]]; then
            echo "One or more CI jobs failed."
            exit 1
          else
            echo "All CI jobs passed or were skipped."
            exit 0
          fi
```



### **如何使用这套CI/CD**



1. **创建文件：** 在您的`arf-workspace`仓库的根目录下，创建`.github/workflows`文件夹，并将上述`ci.yml`文件的内容保存进去。
2. **配置仓库：** 在GitHub仓库的`Settings -> Actions -> General`中，确保Actions是启用的。在`Settings -> Branches`中，为`main`分支添加保护规则，要求在合并前必须通过`CI Status`这个状态检查。
3. **开始开发：** 之后，当任何开发者创建一个指向`main`分支的Pull Request时，这套工作流就会自动运行。他们会在PR页面上看到所有检查的实时状态。只有当所有相关的检查都通过后，代码才被允许合入`main`分支。

这套CI/CD流程，将成为我们项目质量的自动化“守门员”，强制执行我们之前定义的所有开发规范，让您和您的团队可以更专注于代码逻辑本身。