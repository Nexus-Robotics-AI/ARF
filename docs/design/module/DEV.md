# 🔥 ARF 模块开发参考文档：开发者生态 (Dev)

> 🎯 **角色定位:** ARF的"大门"与"工具箱" - 赋能所有开发者。
>
> 📦 **模块代号:** `arf-dev-tools`
>
> ⚡ **所属:** ARF 边缘平台 (Edge Plane)

## 📋 1. 核心职责与设计理念

### 🎯 核心使命 (Core Mission)

作为ARF生态的“大门”和“工具箱”，Dev模块的核心使命是**为所有开发者（包括我们自己）提供一流的、无摩擦的开发、调试和测试体验**。此模块的优劣直接决定了我们生态系统的吸引力、开发者的生产力以及社区的活跃度。

主要应用场景包括：

- **🚀 项目快速启动:** 开发者可以通过一个命令，快速创建符合ARF规范的标准项目模板。
- **💻 本地开发与调试:** 提供工具链，让开发者可以在本地轻松运行、测试和调试他们的模块。
- **📦 统一的SDK:** 提供简洁、易用的多语言SDK，封装底层gRPC通信的复杂性。
- **📖 知识传递:** 通过文档中心，为开发者提供所有必要的教程、API参考和最佳实践。

### 🏗️ 核心架构：CLI为中心 + SDK为核心 (CLI-Centric + SDK-Core)

Dev模块的架构以两个核心产出物为中心：

- **`arf-cli` (命令行工具):** 这是开发者与ARF生态系统交互的**统一入口 (Unified Gateway)**。无论是管理本地容器、查询云端状态还是发布插件，所有操作都应通过`arf-cli`完成。它是一个“瑞士军刀”，为开发者提供一切所需。
- **`Python SDK` (软件开发包):** 这是算法开发者编写逻辑的**核心依赖 (Core Dependency)**。SDK的设计必须追求极致的简洁和Pythonic，让开发者可以专注于算法本身，而不是平台的API细节。

### ⚖️ 设计原则 (Design Principles)

- **🔧 体验至上 (DX First):** 所有工具和SDK的设计，都必须将开发者体验（Developer Experience）放在第一位。命令要直观，API要简洁，错误提示要清晰。
- **🔌 一致性:** CLI的命令风格、SDK的API设计模式，在整个生态系统中必须保持高度一致。
- **🤖 自动化:** 尽可能将重复性的工作（如项目创建、代码生成、环境配置）自动化，解放开发者的生产力。
- **📖 文档即代码:** 文档是Dev模块最重要的产出物之一，必须与代码同步更新，保持准确和清晰。

## 📝 2. 核心需求 (Core Requirements)

| **ID** | **需求描述**                             | **验收标准**                                                 | **优先级** |
| ------ | ---------------------------------------- | ------------------------------------------------------------ | ---------- |
| **V1** | **项目生命周期管理 (Project Lifecycle)** | `arf-cli`必须提供`init`, `build`, `test`, `run`等命令，覆盖从项目创建到运行的全过程。 | **最高**   |
| **V2** | **平台服务交互 (Platform Interaction)**  | `arf-cli`必须能作为客户端，与`ACR`, `DMS`, `Fleet Management`等核心服务进行gRPC通信，执行管理任务。 | **高**     |
| **V3** | **Python SDK核心功能**                   | Python SDK必须提供简洁的`Node`, `Publisher`, `Subscriber`抽象，封装与`DMS`的通信细节。 | **最高**   |
| **V4** | **多语言SDK支持 (Multi-lang SDK)**       | (未来规划) 架构应支持扩展到其他语言的SDK，如C++和Rust。      | **中**     |
| **V5** | **文档中心 (Doc Center)**                | 必须建立一个集中的、基于Web的文档中心，提供教程、API参考和贡献指南。 | **高**     |

## ⚙️ 3. 关键功能与子模块架构

### CLI 3.1 `arf-cli` (命令行工具)

**瑞士军刀**：一个用`Go`实现的、跨平台的单个二进制文件。

- **命令解析器 (Command Parser):** 基于`Cobra`库构建，实现丰富的子命令和参数解析。
  - `arf container [start|stop|ls|logs]` - 管理ACR容器。
  - `arf data [record|info]` - 控制DMS数据记录。
  - `arf market [publish|search]` - 与云端插件市场交互。
- **gRPC客户端管理器 (gRPC Client Manager):** 内部维护一个gRPC客户端池，用于连接到`ACR运行时`、`DMS服务`等后端。
- **项目模板引擎 (Template Engine):** `arf init`命令会内置一系列标准项目模板（如`acr-python-template`, `hal-rust-template`），用于快速创建新项目。

### 🐍 3.2 Python SDK

**翻译官**：为Python开发者提供一个优雅的接口，将复杂的gRPC调用和异步逻辑“翻译”成简单的Python对象和方法。

- **客户端封装:** SDK内部会封装好与`DMS`, `HAL`等核心服务通信的gRPC客户端，并处理好连接、重试等细节，对开发者透明。
- **核心基类 (`Node`, `Publisher`, `Subscriber`):** 这是SDK的核心抽象，让开发者可以用类似ROS的模式进行开发。
- **初始化与日志 (`arf_sdk.init()`, `arf_sdk.get_logger()`):** 提供全局的初始化和日志获取功能，简化开发者的入口代码。

## 🔗 4. 接口设计与数据流

Dev模块是系统中多个gRPC服务的“**主要客户**”。

### 📥 输入数据流

| **数据源**      | **数据类型**          | **优先级** | **延迟要求** | **示例场景**                       |
| --------------- | --------------------- | ---------- | ------------ | ---------------------------------- |
| **人类开发者**  | 命令行输入            | `CRITICAL` | `< 1s`       | 开发者在终端执行`arf container ls` |
| **ACR运行时**   | `LogEntry` (gRPC流)   | `NORMAL`   | `< 1s`       | `arf-cli`实时显示算法容器的日志    |
| **DMS数据总线** | `BusMessage` (gRPC流) | `NORMAL`   | `< 50ms`     | Python SDK的`Subscriber`接收数据   |

### 📤 输出数据流

| **目标模块**    | **数据类型**                   | **保证** | **性能指标**            |
| --------------- | ------------------------------ | -------- | ----------------------- |
| **ACR运行时**   | `StartContainerRequest` (gRPC) | 可靠调用 | CLI命令响应时间 < 500ms |
| **DMS数据总线** | `BusMessage` (gRPC)            | 可靠调用 | SDK发布消息延迟 < 10ms  |
| **标准输出**    | 格式化的文本/表格              | 用户友好 | 清晰、易读的命令行输出  |

## 🛠️ 5. 技术栈与开发环境

### 💻 核心技术栈

| **技术领域**    | **选型**                | **版本要求** | **用途说明**                   |
| --------------- | ----------------------- | ------------ | ------------------------------ |
| **CLI编程语言** | **Go**                  | `1.21+`      | 跨平台、无依赖的单个二进制文件 |
| **SDK编程语言** | **Python**              | `3.10+`      | AI生态的核心，快速迭代         |
| **CLI框架**     | **Cobra, Viper (Go)**   | 最新稳定版   | 构建强大、可配置的命令行应用   |
| **SDK打包**     | **Poetry / Hatch**      | 最新稳定版   | 现代化的Python包管理和发布     |
| **文档生成**    | **Docusaurus / MkDocs** | 最新稳定版   | 从Markdown生成静态文档网站     |

## 🔧 6. 开发实施细节

### 🏗️ 6.1 项目结构 V1

```
tools/
└── arf-cli/                  # arf-cli (Go)
    ├── cmd/                  # Cobra命令定义
    │   ├── root.go
    │   ├── container.go      # 'container'子命令族
    │   └── data.go           # 'data'子命令族
    ├── internal/
    │   ├── clients/          # 调用ARF后台服务的gRPC客户端
    │   └── templates/        # 'init'命令使用的项目模板
    └── go.mod

sdk/
└── python/                   # python-sdk
    ├── arf_sdk/
    │   ├── __init__.py
    │   ├── node.py
    │   └── clients/
    ├── examples/
    │   └── talker_listener.py
    └── pyproject.toml
```

### 🧪 6.2 测试与验证策略



- **单元测试:** 对CLI的命令解析逻辑、SDK的核心类进行单元测试。
- **集成测试:**
  - **CLI-Service测试:** 编写集成测试，启动一个`mock`的gRPC服务，然后用`arf-cli`去调用它，验证端到端的命令执行是否正确。
  - **SDK-DMS测试:** 编写集成测试，启动一个真实的`DMS`服务，然后用Python SDK进行发布和订阅，验证数据收发的正确性。

------



## 🚀 7. 开发任务 (Getting Started)





#### **第一阶段：核心工具链骨架 (Core Toolchain Skeleton)**



- **任务1：实现`arf-cli`骨架和`init`命令**
  - **交付物:** 一个用Go实现的`arf-cli`程序，`init`子命令可以生成一个包含`arf.yaml`, `main.py`, `requirements.txt`的标准算法项目模板。
- **任务2：实现Python SDK骨架**
  - **交付物:** 一个`arf_sdk` Python包，包含空的`Node`, `Publisher`, `Subscriber`类定义和清晰的Docstrings。
- **任务3：搭建文档中心框架**
  - **交付物:** 一个基于Docusaurus/MkDocs的、可以本地运行的空文档网站。



#### **第二阶段：打通与核心服务的交互 (Interaction with Core Services)**



- **任务1：实现`arf-cli run`和日志查看**
  - **交付物:** `arf-cli run`命令能成功调用`ACR运行时`的gRPC服务。`arf-cli container logs`能流式打印日志。
- **任务2：实现SDK的Pub/Sub功能**
  - **交付物:** `Python SDK`的`Publisher`和`Subscriber`能通过gRPC成功连接到`DMS服务`并收发数据。
- **任务3：编写第一个“Hello, World”教程**
  - **交付物:** 一份Markdown教程，指导开发者如何使用`arf-cli`和`SDK`编写一个简单的“发布者-订阅者”应用。



#### **第三阶段：完善用户体验与生态功能 (DX & Ecosystem Features)**



- **任务1：完善CLI的用户体验**
  - **交付物:** 为所有CLI命令增加详细的帮助信息、参数自动补全、以及用户友好的错误提示。
- **任务-2：与插件市场集成**
  - **交付物:** 实现`arf market search/publish`等命令，打通与云端`Marketplace`的交互。
- **任务3：完成“30分钟上手”的“杀手级”教程**
  - **交付物:** 一份图文并茂的`Quickstart.md`文档，并邀请新用户进行测试，确保能在30分钟内完成。



#### **第四阶段：多语言支持与社区建设 (Multi-language & Community)**



- **任务1：设计并实现C++ SDK骨架**
  - **交付物:** 一个C++版本的SDK，提供与Python SDK对等的核心功能。
- **任务2：建立贡献者指南**
  - **交付物:** 一份`CONTRIBUTING.md`文档，清晰地说明社区开发者如何贡献代码、提交Issue和PR。
- **任务3：发布v1.0**
  - **交付物:** 将`arf-cli`的可执行文件和`Python SDK`的wheel包，通过CI/CD自动发布到GitHub Releases和PyPI。