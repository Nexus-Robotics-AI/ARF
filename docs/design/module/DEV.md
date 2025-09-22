# ARF 模块开发参考文档：开发者生态 (Dev)

所属平面: ARF 边缘平面 (Edge Plane)

模块代号: arf-dev-tools

### 1. 核心职责 (Core Mission)

作为ARF生态的“大门”和“工具箱”，为所有开发者（包括我们自己）提供一流的、无摩擦的开发、调试和测试体验。此模块的优劣直接决定了我们生态系统的吸引力和开发者的生产力。

### 2. 关键功能与子模块

- **`arf-cli` (命令行工具):**
  - **项目生命周期管理:** 提供`init`, `build`, `test`, `run`等命令，覆盖从项目创建到运行的全过程。
  - **平台交互:** 作为与ARF边缘平面和云端平面服务交互的统一入口（如部署、监控、日志查看）。
- **Python SDK (软件开发包):**
  - **简化交互:** 封装所有底层的gRPC通信细节，为算法开发者提供简洁、面向对象的Python接口。
  - **核心抽象:** 提供`Node`, `Publisher`, `Subscriber`等核心基类，让开发者可以专注于算法逻辑本身。
- **文档中心:**
  - **API参考:** 自动生成所有`.proto`接口的文档。
  - **教程与最佳实践:** 提供从快速入门到高级主题的系列教程。
  - **贡献者指南:** 明确社区贡献代码和文档的标准与流程。

### 3. 主要接口与数据流

- **输入:**
  - 开发者的命令和代码。
- **输出:**
  - 标准化的项目结构、可执行的程序、清晰的文档。
- **交互协议:**
  - `arf-cli`作为gRPC客户端，调用`ACR`、`DMS`、`Fleet Management`等多个服务。
  - `Python SDK`内部也封装了gRPC客户端，主要与`DMS`服务进行交互。

### 4. 主选技术栈与开发参考

- **CLI:** Go
  - **参考:** [Cobra CLI Framework for Go](https://github.com/spf13/cobra), [Viper](https://github.com/spf13/viper) for[ configuration](https://github.com/spf13/viper)
- **SDK:** Python
  - **参考:** [Python Packaging User Guide](https://www.bing.com/ck/a?!&&p=3365db603b4e59babcad4b5f50980585b49fdbf257a92c4e09a1b54a4fcc2ca5JmltdHM9MTc1NzYzNTIwMA&ptn=3&ver=2&hsh=4&fclid=0f7432fe-449d-608b-3d6a-270045d761a5&psq=Python+Packaging+User+Guide&u=a1aHR0cHM6Ly9wYWNrYWdpbmcucHl0aG9uLm9yZy8)