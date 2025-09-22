# ARF Go 语言设计与代码规范

**适用模块:** `DMS`, `ACR-Runtime`, `API`, `Dev-CLI` 等

## 1. 设计思想 (Guiding Principles)

- **拥抱简洁:** 遵循Go语言的设计哲学，用最简单、最地道的方式解决问题。避免从其他语言（如Java/C++）引入复杂的设计模式。
- **显式错误处理:** **严禁**使用`_`丢弃`error`返回值。每个错误都必须被检查和处理。
- **并发非并行:** 优先使用`goroutine`和`channel`进行并发编程，而不是依赖复杂的锁机制。
- **接口即契约:** 善用`interface`定义行为，实现依赖解耦。

## 2. 代码要求 (Coding Standards)

### 2.1 格式化

- **强制:** 所有代码提交前，**必须**使用`gofmt`和`goimports`进行格式化。

### 2.2 命名规范

- 遵循Go社区的`Effective Go`指南。
- **包名 (Package):** 简短、小写、单个单词。
- **变量/函数/方法:** `camelCase` (私有), `CamelCase` (公有)。
- **接口:** 接口名以`-er`结尾，如`Reader`, `Writer`。

### 2.3 包结构

- 遵循`Standard Go Project Layout`。
- **`cmd/`:** 程序主入口。
- **`internal/`:** 项目内部私有代码。
- **`pkg/`:** 可被外部项目引用的公共库代码。

### 2.4 注释

- 所有公有函数和类型**必须**有标准的Go Doc注释。

### 2.5 错误处理

- 错误信息应包含上下文。推荐使用`fmt.Errorf("module: message: %w", err)`的方式包装错误。
- 错误类型应首字母小写，以`Error`结尾。

### 2.6 测试

- 测试文件与源文件在同一包内，以`_test.go`结尾。
- 测试函数以`Test`开头。
- 善用`testing`包的`t.Parallel()`来加速测试。