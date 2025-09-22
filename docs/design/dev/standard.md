# ARF 统一开发规范

**文档目的:** 本文档定义了所有ARF项目开发者都必须遵守的通用工程标准。它是保证我们多语言、多模块项目能够高效、高质量协同工作的基石。

## 1. 设计思想总则 (Guiding Principles)

- **简洁至上 (Keep It Simple, Stupid - KISS):** 除非有充分的理由，否则永远选择最简单、最直接的实现方式。避免过度工程化。
- **高内聚，低耦合 (High Cohesion, Low Coupling):** 模块内部的功能应该高度相关，模块之间的依赖应尽可能少，并且只能通过明确定义的接口（gRPC）进行交互。
- **接口驱动开发 (Interface-Driven Development):** 先定义并评审`.proto`接口，再进行具体实现。接口是模块间的唯一契约。
- **你构建，你运维 (You Build It, You Run It):** 每个模块的开发者都有责任确保其代码的可测试性、可观测性和可维护性。

## 2. 代码管理 (Source Code Management)

### 2.1 版本控制系统

- **系统:** 统一使用 **Git**。
- **托管:** 统一使用 **GitHub**。

### 2.2 分支模型

- **模型:** 严格遵守 **GitHub Flow**。
  1. `main` 分支是唯一的主分支，永远处于可发布状态。目前基本上用不上这个主分支！
  2. 所有新功能或修复，都必须从`main`创建新的特性分支。(后话)
  3. 分支命名：`<type>/<scope>/<short-description>`
     - **`<type>`:** `feat`, `fix`, `refactor`, `docs`, `test`。
     - **`<scope>`:** 模块名，如`dms`, `hal`, `sdk`。
     - **示例:** `feat/dms/add-publish-api`, `fix/hal/camera-timeout-issue`。
  4. 开发完成后，向`main`分支提交Pull Request (PR)。
  5.feat, fix, refactor, docs, testsu所有类型的分支都有一个main(例如:feat/main,fix/main...),和唯一那个主main不一样。

### 2.3 提交规范 (Commit Convention)

- **规范:** **必须**遵循 **Conventional Commits** 规范。
- **格式:** `<type>(<scope>): <subject>`
  - **示例:** `feat(dms): add message publishing service`
  - **好处:** 提交历史清晰，便于自动化生成更新日志(Changelog)。

## 3. 代码审查 (Code Review)

- **强制性:** **任何代码都合入`main`分支前，必须至少有1名其他核心成员的审查批准 (Approve)。**
- **作者职责:**
  - PR必须是小而专注的，解决一个明确的问题。
  - PR描述中必须清晰说明“做了什么”、“为什么这么做”以及“如何测试”。
  - 提交前必须确保所有CI检查（Lint, Test）通过。
- **审查者职责:**
  - **必须**在24个工作小时内响应。
  - 审查重点：逻辑正确性、是否遵循所有规范、可读性、可维护性。
  - 保持建设性、尊重的沟通，多使用`Suggestion`功能。

## 4. 接口设计 (API Design)

- **协议:** 统一使用 **gRPC + Protocol Buffers (Protobuf 3)**。
- **版本控制:** 所有gRPC服务**必须**在其包名中包含版本号，例如 `package arf.dms.v1;`。接口发生不兼容变更时，必须发布新版本（`v2`）。
- **命名:** 遵循Protobuf官方风格指南（`CamelCase` for Services/RPCs/Messages, `snake_case` for fields）。

## 5. 文档与注释 (Documentation & Comments)

- **模块文档:** 每个独立的模块（服务/库）**必须**在其根目录包含一个`README.md`。
- **接口注释:** 所有的公开函数、方法、类和模块，都**必须**有标准的文档字符串（Docstring），解释其功能、参数和返回值。
- **行内注释:** 只对复杂的、非显而易见的逻辑进行注释，解释“为什么”这么做，而不是“做了什么”。

## 6. 命名规范 (Naming Convention)

- **通用原则:** 清晰、简洁、可预测。避免使用缩写，除非是广泛接受的（如`http`, `id`）。
- **变量:** 使用能清晰表达其业务含义的名称。
- **具体语言规范:** 遵循各语言的惯例（见特定语言规范文档）。