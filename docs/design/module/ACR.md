# 🔥 ARF 模块开发参考文档：算法容器层 (ACR)  V1.1

> 🎯 **角色定位:** ARF的"心脏"与"工具箱" - 承载、管理和执行所有AI算法。
>
> 📦 **模块代号:** `arf-edge-acr`
>
> ⚡ **所属:** ARF 边缘平台 (Edge Plane)

## 📋 1. 核心职责与设计理念

### 🎯 核心使命 (Core Mission)

作为ARF的“心脏”和“工具箱”，ACR的核心使命是**为算法科研团队和工程团队提供一个标准化的、隔离的、可插拔的算法运行环境**。它负责将千差万别的AI算法（视觉感知、SLAM、语音处理等）封装成统一的、可被`DIL`决策层调度的“能力单元”。

主要应用场景包括：

- **📦 算法封装:** 将Python/C++等语言实现的算法，打包成标准的、自包含的容器。
- **📦 核心智能算法承载:** 运行感知、SLAM、语音处理等由`DIL`主导调度的核心机器人AI算法。
- **⚙️ 生命周期管理:** 负责算法容器的启动、停止、健康检查和资源监控。
- **🔌 动态加载:** 支持在系统运行时，动态地加载、更新或替换算法，实现热切换。
- **🔗 数据流对接:** 作为算法与`DMS`数据总线之间的桥梁，负责数据的输入和输出。
- **🧠 在线学习容器支持:** 承载特殊的在线学习算法，如“在线残差学习”容器，它订阅`DIL`的动作和`HAL`的力反馈，实时计算补偿动作，是实现人机协同和持续学习的关键一环。
- **⚡ 算法图编排:** 将多个独立的算法容器，按需编排成一个高性能的算法流水线（DAG），以完成更复杂的任务。
- **⚡ 边缘数据预处理:** 作为`DMS`数据采集流水线的一部分，对原始数据进行实时的预处理、标注或质量检查。
- **🧩 应用技能包执行:** 执行由`应用层`定义的、由多个算法容器组合而成的“技能包”。

### 🏗️ 核心架构：运行时与容器分离 (Runtime-Container Decoupling)

ACR的设计将“管理器”和“被管理者”彻底分离：

- **ACR运行时 (ACR Runtime):** 这是一个常驻的、核心的Go服务 (`acr-runtime-service`)。它像一个“应用商店管理器”，负责接收指令，下载、安装、启动和监控所有的算法容器。V1.1架构中，它还新增了**资源感知**和**算法图编排**的能力。
- **算法容器 (Algorithm Container):** 这是被管理的“App”。每一个算法都被打包成一个独立的容器（初期为`venv`隔离的进程，后期为`Docker`容器）。每个容器内部都包含一个轻量级的gRPC服务，用于接收`ACR运行时`的管理指令和数据。

### ⚖️ 设计原则 (Design Principles)

- **📦 隔离性优先:** 算法容器之间、算法与核心服务之间必须严格隔离。单个算法的崩溃或资源泄漏，**绝不能**影响系统的其他部分。
- **🔌 标准化接口:** 所有算法容器必须对外暴露统一的gRPC管理接口和DMS数据接口。
- **⚡ 高性能:** ACR运行时与算法容器之间的数据交换应尽可能高效，避免不必要的拷贝和序列化开销。
- **🔧 易用性:** 为算法开发者提供的封装工具和SDK必须极其简单，让他们可以专注于算法逻辑本身。
- **🧠 自适应性:** 运行时应能感知机器人整体状态，并自适应地调整资源分配，保障关键任务的执行。
- **🛠️ 服务化 (Service-Oriented):** ACR容器不仅是主流程的一部分，也可以作为后台服务被其他边缘模块（如`HAL`, `DMS`）按需调用。
- **🐛 可调试性优先 (Debuggability First):** `ACR运行时`的设计必须支持与`Dev`工具的深度集成，允许开发者轻松地将容器连接到‘时间旅行’调试器的数据流上，实现对复杂错误的确定性复现。

## 📝 2. 核心需求 (Core Requirements)

| **ID** | **需求描述**                           | **验收标准**                                                 | **优先级**     |
| ------ | -------------------------------------- | ------------------------------------------------------------ | -------------- |
| **A1** | **算法环境隔离 (Env. Isolation)**      | 每个算法必须运行在独立的环境中（`venv`/`Docker`），拥有自己独立的依赖库版本。 | **最高**       |
| **A2** | **生命周期管理 (Lifecycle Mgt.)**      | `ACR运行时`必须提供gRPC接口，用于远程启动、停止、查询算法容器的状态。 | **最高**       |
| **A3** | **数据流对接 (Data Flow Integration)** | 算法容器必须能通过标准SDK，轻松地从`DMS`订阅输入数据，并将处理结果发布回`DMS`。 | **最高**       |
| **A4** | **资源监控与限制 (Resource Mgt.)**     | `ACR运行时`必须能监控每个算法容器的CPU、内存使用情况，并能在未来支持资源限制。 | **高**         |
| **A5** | **配置注入 (Config Injection)**        | `ACR运行时`必须支持在启动算法容器时，通过环境变量或配置文件，向其注入自定义的参数。 | **高**         |
| **A6** | **热更新与热切换 (Hot Swap)**          | (未来规划) 支持在不中断系统服务的情况下，将一个正在运行的算法容器平滑地升级到新版本。 | **中**         |
| **A7** | **资源自适应调度**                     | **[V1.1+ 新增]** `ACR运行时`必须能根据机器人的全局状态（如电量、任务模式）动态调整容器的资源配额。 | **高 (V1.1+)** |
| **A8** | **零拷贝通信**                         | **[V1.1+ 新增]** 必须为指定的高性能算法流水线提供基于共享内存的零拷贝通信机制。 | **中 (V1.1+)** |
| **A9** | **算法图执行**                         | **[V1.1+ 新增]** 必须支持通过配置文件定义一个由多个算法容器组成的有向无环图（DAG），并由运行时负责编排执行。 | **中 (V1.1+)** |

## V1.1

### **`ACR` (算法容器层)**

- **需要做的工作:**
  1. **新增“在线残差学习”容器:**
     - 这是一个全新的`ACR`算法容器。
     - **职责:** 订阅`DIL`输出的VLA宏观动作、`HAL`输出的实时力反馈数据，然后加载一个预训练的残差网络（如MLP），计算出补偿动作`Δa`，并将其发布到一个新的DMS主题上。
     - 这个容器需要遵循我们所有的ACR开发规范。

## ⚙️ 3. 关键功能与子模块架构

### 🧠 3.1 ACR运行时服务 (ACR Runtime Service)

**应用商店管理器**：这是ACR的核心，一个用`Go`实现的常驻gRPC服务。

- **容器管理器:** 内部维护一个`map[container_id]ContainerProcess`的状态表，追踪所有算法容器的进程ID、状态、资源使用等。
- **进程启动器:**
  - **v0.1 (venv模式):** 负责在指定目录自动创建Python `venv`，通过`pip install`安装依赖，然后以**子进程**的方式启动算法的`main.py`。
  - **v0.2 (Docker模式):** 调用Docker Engine API来拉取镜像、创建和管理容器。
- **日志聚合器:** 收集所有算法子进程的标准输出/错误流，并通过`StreamLogs` gRPC接口向上层提供。

### V1.1 新增高级功能子模块



- **资源调节器 (Resource Governor):**
  - **职责:** 这是`ACR运行时`内部的一个新组件。它会订阅`DMS`上的机器人状态主题，获取电量、系统负载、当前DIL任务模式等信息。基于预设的规则（可由`Fleet Management`下发），它会动态调整正在运行的容器的CPU/GPU资源限制，以确保系统整体的稳定性和续航。
- **算法图执行引擎 (Graph Execution Engine):**
  - **职责:** 负责解析用户提交的算法图（DAG）配置文件。**该引擎是实现`应用层`技能商店的核心技术支撑**。每一个可安装的‘技能’，都可以被定义为一个算法图，由本引擎负责解析和执行。它会根据图的拓扑结构，按顺序启动各个算法容器，并为需要零拷贝通信的节点建立共享内存通道，然后监控整个图的执行状态。
- **共享内存管理器 (Shared Memory Manager):**
  - **职责:** 负责在需要零拷贝通信的容器之间，创建、管理和清理共享内存段，并将内存段的地址信息注入到相应的容器中。



### 📦 3.2 算法容器 (Algorithm Container)

**标准化的App**：每一个算法都被封装成一个符合ARF规范的项目。

- **`arf.yaml`:** 每个算法项目的根目录下都有一个元数据文件，描述了该算法的名称、版本、输入/输出主题、资源需求等。V1.1中可扩展，用于声明资源需求和零拷贝通信端口。
- **`main.py`:** 算法的启动入口。它会使用`ARF Python SDK`来初始化一个`Node`，创建`Publisher`和`Subscriber`，然后进入`spin()`循环。
- **`Dockerfile` (Docker模式):** 定义了如何将算法代码和依赖打包成一个可运行的镜像。

## 🔗 4. 接口设计与数据流

ACR模块的对外接口，由`acr.proto`文件严格定义。

### 📥 输入数据流

| **数据源**        | **数据类型**            | **优先级** | **延迟要求** | **示例场景**                            |
| ----------------- | ----------------------- | ---------- | ------------ | --------------------------------------- |
| **DMS数据总线**   | `BusMessage`            | `HIGH`     | `< 10ms`     | 算法容器订阅摄像头原始数据              |
| **Dev (arf-cli)** | `StartContainerRequest` | `NORMAL`   | `< 500ms`    | 开发者通过命令行启动一个算法            |
| **DIL决策层**     | `StartContainerRequest` | `HIGH`     | `< 100ms`    | DIL根据任务需求动态加载一个运动规划算法 |

### 📤 输出数据流

| **目标模块**      | **数据类型**             | **保证**   | **性能指标**             |
| ----------------- | ------------------------ | ---------- | ------------------------ |
| **DMS数据总线**   | `BusMessage`             | 高吞吐量   | 算法处理结果的发布       |
| **Dev (arf-cli)** | `LogEntry`               | 实时日志流 | 方便开发者调试           |
| **DIL决策层**     | `StartContainerResponse` | 异步响应   | 容器启动成功或失败的确认 |

### 🧬 核心API草案 (`acr.proto`)

```protobuf
// protos/arf/edge/v1/acr.proto
syntax = "proto3";

package arf.acr.v1;

// 容器的运行状态
enum ContainerStatus {
  STATUS_UNSPECIFIED = 0;
  RUNNING = 1;
  STOPPED = 2;
  ERROR = 3;
}

//ContainerInfo, LogEntry 等消息定义 ...

// 算法容器运行时服务
service ACRRuntimeService {
  // 启动并管理一个算法容器
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
  // 停止一个正在运行的算法容器
  rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
  // 获取当前运行时管理的所有容器的状态
  rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
  // 流式传输指定容器的日志
  rpc StreamLogs(StreamLogsRequest) returns (stream LogEntry);

  // [V1.1 新增] 启动并管理一个算法图
  rpc StartGraph(StartGraphRequest) returns (StartGraphResponse);
  // [V1.1 新增] 停止一个正在运行的算法图
  rpc StopGraph(StopGraphRequest) returns (StopGraphResponse);
}

message StartContainerRequest {
  string container_id = 1;
  string project_path = 2;
  // ... 其他字段 ...
  
  // [V1.1 新增] 资源限制
  optional ResourceLimits resource_limits = 4;
}

message ResourceLimits {
    optional float cpu_cores = 1;
    optional int64 memory_mb = 2;
}

message StartGraphRequest {
    string graph_id = 1;
    // Base64编码的算法图YAML定义文件内容
    string graph_yaml_base64 = 2;
}
// ... 详细消息定义请参考核心接口文档 ...
```

## 🛠️ 5. 技术栈与开发环境

### 💻 核心技术栈

| **技术领域**       | **选型**                   | **版本要求**    | **用途说明**                      |
| ------------------ | -------------------------- | --------------- | --------------------------------- |
| **运行时编程语言** | **Go**                     | `1.21+`         | 构建高性能、高并发的ACR运行时服务 |
| **算法封装语言**   | **Python / C++**           | `Python 3.10+`  | 算法实现和SDK调用                 |
| **gRPC框架**       | **grpc-go, grpcio**        | 最新稳定版      | 实现`.proto`定义的接口            |
| **容器化技术**     | **Python `venv`, Docker**  | `Docker 20.10+` | 实现算法的环境隔离                |
| **测试框架**       | **Go Native Test, Pytest** | 最新稳定版      | 单元测试和接口测试                |

## 🔧 6. 开发实施细节

### 🏗️ 6.1 项目结构 V1

```
services/edge-plane/acr-runtime/       # ACR运行时服务 (Go)
├── internal/
│   ├── manager/                   # 容器生命周期管理核心逻辑
│   │   ├── manager.go
│   │   └── process_manager.go     # venv进程管理
│   └── server/                    # gRPC服务器实现
│       └── server.go
├── api/                             # .proto文件的软链接
├── Dockerfile
├── go.mod
└── README.md

algorithms/perception/object-detector/ # 一个算法容器项目 (Python)
├── models/
│   └── yolov8n.pt
├── src/
│   ├── main.py                    # 算法入口，使用ARF SDK
│   └── infer.py                   # 核心推理脚本
├── arf.yaml                         # 算法元数据描述
├── requirements.txt
└── README.md
```

### 🧪 6.2 测试与验证策略



- **单元测试:** 对`ACR运行时`的`manager`模块进行单元测试。对算法容器的`infer.py`进行单元测试。
- **集成测试:**
  - **启动/停止测试:** 编写测试脚本，通过gRPC调用`StartContainer`和`StopContainer`，并验证`ListContainers`返回的状态是否正确。
  - **数据流测试:** 启动一个“回声”算法容器，该容器订阅`topic_A`，并将收到的数据原样发布到`topic_B`。编写测试验证数据的完整性和正确性。
  - **依赖安装测试:** 创建一个有复杂依赖的`requirements.txt`的算法，验证ACR运行时能否正确安装所有依赖。

------



## 🚀 7. 开发任务 (Getting Started)





#### **第一阶段：本地进程管理 (Local Process Management)**



- **任务1：实现`ACR运行时`的gRPC服务骨架**
  - **交付物:** 一个用Go实现的`ACRRuntimeService`，包含所有RPC方法的空实现。
- **任务2：实现`venv`模式的`StartContainer`**
  - **交付物:** `StartContainer`方法能够接收一个本地项目路径，在其中自动创建Python `venv`，安装`requirements.txt`中的依赖，并以子进程方式启动`main.py`。
- **任务3：实现日志流式传输**
  - **交付物:** `StreamLogs`方法能够捕获算法子进程的标准输出/错误，并将其流式传输给gRPC客户端。



#### **第二阶段：与DMS集成 (Integration with DMS)**



- **任务1：实现`arf.yaml`解析**
  - **交付物:** `ACR运行时`能够解析算法项目中的`arf.yaml`，并获取其`subscribes`和`publishes`的主题列表。
- **任务2：数据注入**
  - **交付物:** 与`DMS`模块协同，设计并实现一种机制（如共享内存、本地socket或管道），将`DMS`订阅到的数据高效地传递给算法子进程。
- **任务3：实现第一个算法容器**
  - **交付物:** 一个简单的“回声”算法容器，使用`Python SDK`，能完整地跑通“从DMS订阅->处理->发布回DMS”的流程。



#### **第三阶段：Docker集成与资源监控 (Dockerization & Monitoring)**



- **任务1：实现Docker模式的`StartContainer`**
  - **交付物:** 为`StartContainer`增加一个`runtime`参数。当值为`docker`时，`ACR运行时`将通过Docker Engine API来启动算法容器。
- **任务2：实现资源监控**
  - **交付物:** `ListContainers`的返回结果中，包含每个容器的实时CPU和内存使用率。
- **任务3：提供`arf-cli`集成**
  - **交付物:** 与`Dev`模块负责人协作，确保`arf-cli container start/stop/list/logs`等命令能够正常工作。



#### **第四阶段：鲁棒性与高级功能 (Robustness & Advanced Features)**



- **任务1：完善健康检查与自动重启**
  - **交付物:** `ACR运行时`能够监控算法容器的健康状况，并在其意外退出时，根据`arf.yaml`中的重启策略进行自动重启。
- **任务2：实现配置注入**
  - **交付物:** `StartContainer`请求中可以传入环境变量，并被成功注入到算法容器的运行环境中。
- **任务3：压力测试**
  - **交付物:** 提交一份测试报告，展示`ACR运行时`在同时管理多个（如20个）算法容器时的性能表现和资源开销。



#### **第五阶段及以后：迈向智能与可组合运行时 (Towards Intelligent & Composable Runtime)**



*此部分为V1.1及后续版本规划，旨在提升性能和灵活性。*

- **任务5.1: 实现资源自适应调度**
  - **交付物:** 实现`资源调节器`。在一个测试场景中，当模拟的电池电量低于阈值时，`ACR运行时`能够自动降低一个或多个非关键算法容器的CPU占用率。
- **任务5.2: 开发零拷贝通信原型**
  - **交付物:** `ACR运行时`能够为两个指定的容器建立共享内存通道。在一个高帧率图像传输的测试中，验证使用共享内存的延迟和CPU占用率显著低于通过`DMS`传输。
- **任务5.3: 实现算法图执行引擎**
  - **交付物:** `ACR运行时`能够解析一个简单的三节点（A->B->C）算法图YAML，并成功按顺序启动、连接和运行所有容器。提供`StartGraph`和`StopGraph` API。
- **任务5.4: 提供算法图开发套件**
  - **交付物:** 完善`arf-cli`和SDK，提供工具和示例，指导开发者如何创建和调试算法图，并利用零拷贝通信等高级功能。