# 🔥 ARF 模块开发参考文档：插件市场 (Marketplace)



> 🎯 **角色定位:** ARF生态的"中央银行"与"应用商店" - 存储、版本化和分发所有数字资产。
>
> 📦 **模块代号:** `arf-cloud-marketplace`
>
> ⚡ **所属:** ARF 云端平面 (Cloud Plane)

------



## 📋 1. 核心职责与设计理念





### 🎯 核心使命 (Core Mission)



作为ARF生态所有“数字资产”的“中央银行”和“应用商店”，插件市场的核心使命是为所有驱动、算法、模型、应用套件等插件提供一个统一的**存储、版本管理、发现和分发**中心。 它是连接云端“生产”（训练、评测）和“消费”（部署）的枢纽，确保了整个生态系统软件资产的一致性、可追溯性和安全性。

主要应用场景包括：

- **📦 资产存储与版本化:** 为所有数字资产提供类似Docker Hub或PyPI的版本管理能力。
- **🔍 搜索与发现:** 提供强大的搜索功能，方便开发者和`舰队管理`等自动化系统找到所需的插件。
- **🛡️ 访问控制:** 支持插件的公有和私有模式，并提供精细化的权限管理。
- **🚚 可靠分发:** 为`舰队管理`模块提供稳定、可靠的渠道来拉取插件，并部署到边缘机器人。



### 🏗️ 核心架构：元数据服务 + 存储后端 (Metadata Service + Storage Backend)



插件市场的架构将“资产信息”和“资产实体”分离，以实现最大的灵活性和可伸缩性。

- **核心元数据服务 (Go Service):** 这是一个用`Go`实现的核心gRPC服务。它负责管理所有插件的元数据，如名称、版本、描述、依赖关系、兼容性标签（如CUDA版本）以及访问权限。所有搜索、查询和权限校验都在这一层完成。
- **存储后端 (Storage Backend):** 这是实际存储插件文件（如容器镜像、模型权重）的地方。我们采用业界成熟的开源仓库作为后端（如**Harbor**或**Nexus**），以处理大规模二进制文件的存储和分发。



### ⚖️ 设计原则 (Design Principles)



- **🔄 版本化一切 (Version Everything):** 每个上传到市场的插件都必须有唯一的、不可变的语义化版本号，确保部署和实验的可复现性。
- **🏷️ 元数据驱动 (Metadata-Driven):** 插件的可发现性和可用性由其丰富的元数据决定。特别是兼容性标签，是实现异构硬件适配的关键。
- **🔒 安全第一 (Security First):** 必须提供严格的认证和授权机制，确保只有授权的用户或服务才能发布或访问特定的私有插件。
- **🧩 原子性与自包含 (Atomic & Self-Contained):** 每个插件都应被视为一个独立的、自包含的单元。其所有依赖都应在容器内部解决。

------



## 📝 2. 核心需求 (Core Requirements)



| ID     | 需求描述               | 验收标准                                                     | 优先级   |
| ------ | ---------------------- | ------------------------------------------------------------ | -------- |
| **M1** | **插件版本管理**       | 系统必须支持插件的发布、查询和下载。每个插件支持多个版本，并遵循语义化版本规范。 | **最高** |
| **M2** | **与舰队管理集成**     | `舰队管理`模块必须能够查询插件信息，并获取下载链接以部署到边缘设备。 | **最高** |
| **M3** | **精细化访问控制**     | 必须支持将插件设为“公开”或“私有”，私有插件只有授权用户/团队可以访问。 | **高**   |
| **M4** | **丰富的元数据与搜索** | 必须支持按名称、类型、关键词和兼容性标签（如`cuda-11.8`）进行搜索。 | **高**   |
| **M5** | **自动化发布流程**     | `训练模块`和`评测模块`必须能将通过验证的模型和算法包自动发布到市场。 | **高**   |
| **M6** | **资产类型支持**       | 必须支持所有核心数字资产类型，包括`HAL_DRIVER`, `ACR_ALGORITHM`, `TRAINING_SCRIPT`, 和 `APPLICATION_SUITE`。 | **最高** |

------



## ⚙️ 3. 关键功能与子模块架构





### 🧠 3.1 市场API服务 (Marketplace API Service)



**信息中心角色**：这是一个用`Go`实现的、高并发的gRPC服务，是所有与插件市场交互的统一入口。

- **API端点:** 对外提供`marketplace.proto`中定义的gRPC服务，处理插件的注册、查询、下载请求等。
- **认证与授权:** 对每个API请求进行身份验证，并根据插件的访问控制列表（ACL）进行授权检查。
- **元数据管理器:** 负责插件元数据的增删改查，并将其持久化到数据库中。
- **存储后端适配器:** 将上层的下载请求，翻译成对底层Harbor/Nexus API的调用或生成相应的下载URL。



### 📦 3.2 存储后端 (Storage Backend)



**仓库角色**：这是插件二进制文件的实际存储位置。

- **容器镜像仓库 (Container Registry):** 使用**Harbor**或类似方案，专门用于存储`HAL`驱动、`ACR`算法等容器化插件。
- **通用制品仓库 (Artifact Repository):** 使用**Nexus**或S3/MinIO，用于存储非容器化的资产，如模型文件（`.pth`, `.onnx`）、数据集描述文件等。

------



## 🔗 4. 接口设计与数据流





### 📥 输入数据流



| **数据源**        | **数据类型**            | **优先级** | **延迟要求** | **示例场景**                             |
| ----------------- | ----------------------- | ---------- | ------------ | ---------------------------------------- |
| **训练/评测模块** | `RegisterPluginRequest` | `NORMAL`   | N/A (异步)   | 训练任务成功后，自动发布新版模型插件     |
| **开发者(CLI)**   | `RegisterPluginRequest` | `NORMAL`   | `< 10s`      | 开发者手动发布一个新版本的硬件驱动       |
| **CI/CD流水线**   | `RegisterPluginRequest` | `NORMAL`   | N/A (自动化) | CI/CD在代码合并后，自动构建并发布ACR容器 |



### 📤 输出数据流



| **目标模块**    | **数据类型**                              | **保证** | **性能指标**         |
| --------------- | ----------------------------------------- | -------- | -------------------- |
| **舰队管理**    | `DownloadPluginResponse`                  | 可靠下载 | 提供高带宽下载链接   |
| **ACR运行时**   | `PluginDetails`, `DownloadPluginResponse` | 可靠查询 | 查询延迟 < 500ms     |
| **开发者(CLI)** | `PluginDetails`                           | 实时响应 | 搜索响应时间 < 1s    |
| **训练模块**    | `DownloadPluginResponse`                  | 可靠下载 | 下载基础模型用于微调 |



### 🧬 核心API草案 (`marketplace.proto`)



```protobuf
// protos/arf/cloud/v1/marketplace.proto
syntax = "proto3";

package arf.cloud.v1;

import "arf/v1/common.proto";

// 插件市场服务
service MarketplaceService {
  // 发布/注册一个新插件
  rpc RegisterPlugin(RegisterPluginRequest) returns (arf.v1.PluginIdentifier);
  // 获取插件的详细信息
  rpc GetPlugin(arf.v1.PluginIdentifier) returns (PluginDetails);
  // 下载插件（获取下载链接或直接流式传输）
  rpc DownloadPlugin(arf.v1.PluginIdentifier) returns (DownloadPluginResponse);
}

enum PluginType {
  TYPE_UNSPECIFIED = 0;
  HAL_DRIVER = 1;
  ACR_ALGORITHM = 2;
  TRAINING_SCRIPT = 3;
  APPLICATION_SUITE = 4;
}

message RegisterPluginRequest {
  string name = 1;
  PluginType type = 2;
  // ... 其他元数据，如描述、作者等
}
message PluginDetails {
  arf.v1.PluginIdentifier id = 1;
  PluginType type = 2;
  string description = 3;
  repeated string versions = 4;
}
message DownloadPluginResponse {
  string download_url = 1;
}
```

------



## 🛠️ 5. 技术栈与开发环境





### 💻 核心技术栈



| **技术领域**     | **选型**                     | **版本要求** | **用途说明**                 |
| ---------------- | ---------------------------- | ------------ | ---------------------------- |
| **核心服务语言** | **Go**                       | `1.21+`      | 构建高性能、高并发的API服务  |
| **容器镜像仓库** | **Harbor**                   | 最新稳定版   | 存储和分发Docker镜像         |
| **通用制品仓库** | **Nexus Repository / MinIO** | 最新稳定版   | 存储模型文件等非容器资产     |
| **数据库**       | **PostgreSQL**               | 最新稳定版   | 存储插件元数据和访问控制列表 |
| **gRPC框架**     | **grpc-go**                  | 最新稳定版   | 实现`.proto`定义的接口       |

------



## 🔧 6. 开发实施细节





### 🏗️ 6.1 项目结构 V1



```
services/cloud-plane/marketplace/
├── internal/
│   ├── store/               # 数据库交互逻辑
│   ├── auth/                # 认证与授权逻辑
│   └── backend/             # 与Harbor/Nexus交互的适配器
├── server/                  # gRPC服务器实现
├── api/                     # .proto文件的软链接
├── deployments/
│   └── helm/                # 部署服务的Helm Chart
├── Dockerfile
├── go.mod
└── README.md
```



### 🧪 6.2 测试与验证策略



- **单元测试:** 对元数据管理、权限校验等核心业务逻辑进行单元测试。
- **集成测试:**
  - **发布-下载闭环测试:** 编写脚本，通过gRPC `RegisterPlugin`发布一个测试插件，然后通过`DownloadPlugin`下载，验证文件完整性。
  - **权限控制测试:** 创建不同角色的用户，验证私有插件的访问控制是否符合预期。
  - **搜索功能测试:** 发布一系列带有不同标签的插件，然后执行搜索查询，验证返回结果的准确性。

------



## 🚀 7. 开发任务 (Getting Started)





#### **第一阶段：基础存储与手动发布 (Basic Storage & Manual Publishing)**



- **任务1：部署存储后端**
  - **交付物:** 一个在Kubernetes上运行的Harbor实例，用于存储容器镜像。
- **任务2：开发API服务骨架**
  - **交付物:** 一个Go gRPC服务，实现`RegisterPlugin`和`DownloadPlugin`的空方法。元数据可以暂时存在内存或一个简单的JSON文件中。
- **任务3：实现`arf-cli`集成**
  - **交付物:** 实现`arf-cli market publish`和`arf-cli market download`命令，允许开发者手动发布和下载插件。



#### **第二阶段：元数据管理与集成 (Metadata & Integration)**



- **任务1：集成数据库**
  - **交付物:** 将插件元数据管理逻辑迁移到PostgreSQL数据库。
- **任务2：与`舰队管理`集成**
  - **交付物:** `舰队管理`模块可以调用`MarketplaceService`的`GetPlugin`和`DownloadPlugin`接口，成功拉取插件用于部署。
- **任务3：实现`ACR运行时`的兼容性检查**
  - **交付物:** `ACR运行时`能够查询插件的兼容性标签，并与自身环境进行比对。



#### **第三阶段：高级功能与Web UI (Advanced Features & UI)**



- **任务1：实现搜索功能**
  - **交付物:** API支持基于名称、类型和标签的复杂搜索查询。
- **任务2：实现访问控制**
  - **交付物:** `RegisterPlugin`时可以指定插件为私有，只有插件所有者或团队成员才能访问。
- **任务3：开发Web UI原型**
  - **交付物:** 一个简单的React/Vue前端页面，可以浏览、搜索和查看插件市场的公开插件。



#### **第四阶段：自动化与生态闭环 (Automation & Ecosystem Loop)**



- **任务1：与`训练模块`集成**
  - **交付物:** `训练模块`在训练成功后，可以自动调用`RegisterPlugin`将新训练的模型发布到市场。
- **任务2：与`评测模块`集成**
  - **交付物:** `评测模块`可以将被评测的插件（如算法）的版本与评测报告关联，并将结果更新到插件市场的元数据中。
- **任务3：性能与可靠性测试**
  - **交付物:** 提交一份测试报告，展示在高并发下载请求下的系统性能和稳定性。