# 🔥 ARF 模块开发参考文档：数据采集与管理层 (DMS)

> 🎯 **角色定位:** ARF的"血液循环系统"与"黑匣子" - 采集、路由、记录和同步所有数据。
>
> 📦 **模块代号:** `arf-edge-dms`
>
> ⚡ **所属:** ARF 边缘平台 (Edge Plane)

## 📋 1. 核心职责与设计理念

### 🎯 核心使命 (Core Mission)

作为ARF边缘平面的“血液循环系统”和“黑匣子”，DMS的核心使命是**采集、记录、路由和同步**所有模块间的数据，为实时决策提供数据源，并为云端的离线训练提供高质量的“食粮”。

主要应用场景包括：

- **📨 实时数据路由:** 为所有模块提供高性能的发布/订阅消息总线。
- **📹 数据采集与记录:** 提供类似`rosbag`的机制，将关键数据流持久化到本地。
- **🔄 云端数据同步:** 负责将本地记录的数据安全、可靠地上传到云端`数据中心`。
- **⏪ 数据回放与调试:** (未来) 支持将记录的数据按原有时序重新发布，用于算法调试和回归测试。

### 🏗️ 核心架构：中心化路由 + 分布式代理 (Centralized Routing + Distributed Agent)

DMS的核心是一个**中心化的gRPC服务 (`dms-service`)**，但其功能通过轻量级的客户端库（代理）分布在每个需要收发数据的模块中。

- **`dms-service` (Go):** 这是“交通指挥中心”。它负责维护主题路由表、实现数据记录的控制逻辑，并管理云同步的后台任务。
- **`dms-client` (Go/Python):** 这是“邮递员”。每个模块（如HAL驱动、ACR容器）内部都会集成这个客户端库，它负责处理消息的序列化/反序列化，并与`dms-service`建立持久的gRPC连接。

### ⚖️ 设计原则 (Design Principles)

- **⚡ 高性能:** 数据总线的路由延迟应尽可能低，吞吐量应尽可能高。
- **🛡️ 可靠性:** 数据记录和云同步过程必须可靠，支持断点续传，确保数据不丢失。
- **decoupling:** 数据记录功能与数据总线的核心路由功能完全解耦。
- **🔧 易用性:** 为开发者提供的客户端API（`Publisher`/`Subscriber`）必须极其简洁易用。

## 📝 2. 核心需求 (Core Requirements)

| **ID** | **需求描述**                             | **验收标准**                                                 | **优先级** |
| ------ | ---------------------------------------- | ------------------------------------------------------------ | ---------- |
| **D1** | **高性能数据总线 (High-Perf Bus)**       | 必须提供一个低延迟、高吞吐量的发布/订阅系统，用于模块间的实时数据交换。 | **最高**   |
| **D2** | **选择性数据记录 (Selective Recording)** | 必须提供一个类似`rosbag`的机制，允许用户或系统根据配置，选择性地将一个或多个主题的数据流记录到本地文件中。**这是数据采集的核心功能。** | **最高**   |
| **D3** | **标准化数据格式 (Standard Format)**     | 所有被记录的数据，都应以高效的、标准化的格式（如Apache Parquet）进行存储，便于后续的离线处理和分析。 | **高**     |
| **D4** | **远程控制接口 (Remote Control)**        | 必须提供gRPC接口，允许外部模块（如`遥操作模块`或`DIL`）远程启动、停止和配置数据记录任务。 | **高**     |
| **D5** | **云同步代理 (Cloud Sync Agent)**        | 必须包含一个后台服务，负责在网络条件允许时，将本地录制好的数据文件，安全、可靠地上传到云端的`数据中心 (Data Hub)`。 | **中**     |
| **D6** | **数据回放 (Data Replay)**               | (未来规划) 能够读取本地的数据记录文件，并将其内容按原始的时间戳和速率，重新发布到数据总线上，用于调试和测试。 | **低**     |

## ⚙️ 3. 关键功能与子模块架构

### 🧠 3.1 实时数据总线 (Real-time Pub/Sub Bus)

**交通指挥中心**：这是DMS的核心，负责所有消息的实时路由。

- **实现方案:** 基于`Go`语言实现，底层可插拔地支持`Redis Pub/Sub`（用于本地开发）或`ZeroMQ`（用于更高性能的场景）。
- **路由逻辑:** 维护一个`map[topic][]subscribers`的内存路由表，实现高效的消息分发。

### 📹 3.2 数据记录器 (Data Recorder)

**黑匣子**：数据记录功能被设计为一个**DMS内部的特殊订阅者**。

- **工作流程:**
  1. 当`dms-service`收到一个`StartRecordingRequest`请求时，它会在内部创建一个**“记录器订阅者 (Recorder Subscriber)”**。
  2. 这个订阅者会订阅请求中指定的所有主题（如`hal.sensor.*`, `acr.perception.*`）。
  3. 在收到任何消息时，将消息高效地写入一个本地的**Apache Parquet**文件中。
  4. 当收到`StopRecordingRequest`时，取消订阅并关闭文件句柄。

### ☁️ 3.3 云同步代理 (Cloud Sync Agent)

**搬运工**：云同步被设计为一个**低优先级的后台任务**，运行在`dms-service`内部。

- **工作流程:**
  1. 维护一个待上传文件的队列。
  2. 独立的、低优先级的goroutine会定期检查队列。
  3. 在上传前，检查网络连接状态和系统资源占用。
  4. 上传过程支持断点续传。
  5. 上传完成后，根据配置策略清理本地旧文件。

## 🔗 4. 接口设计与数据流

DMS模块的对外接口，由`dms.proto`文件严格定义。

### 📥 输入数据流

| **数据源**     | **数据类型**            | **优先级** | **延迟要求** | **示例场景**            |
| -------------- | ----------------------- | ---------- | ------------ | ----------------------- |
| **HAL传感器**  | `ImageFrame`, `IMUData` | `HIGH`     | `< 5ms`      | 摄像头图像发布          |
| **ACR算法**    | `DetectionResult`       | `NORMAL`   | `< 20ms`     | 目标检测结果发布        |
| **遥操作/API** | `StartRecordingRequest` | `NORMAL`   | `< 100ms`    | 用户通过CLI启动数据采集 |

### 📤 输出数据流

| **目标模块**     | **数据类型**   | **保证**   | **性能指标**          |
| ---------------- | -------------- | ---------- | --------------------- |
| **ACR/DIL/其他** | `BusMessage`   | 低延迟路由 | 消息端到端延迟 < 10ms |
| **本地文件系统** | `.parquet`文件 | 可靠写入   | 写入带宽 > 100MB/s    |
| **云端数据中心** | 数据文件包     | 断点续传   | 上传成功率 > 99.9%    |

### 🧬 核心API草案 (`dms.proto`)

```
// protos/arf/edge/v1/dms.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/v1/common.proto";
import "google/protobuf/any.pb";

// 数据总线消息
message BusMessage {
    arf.v1.Header header = 1;
    string topic = 2;
    google.protobuf.Any payload = 3;
}

// 数据记录(采集)服务的状态
message RecorderState {
    bool is_recording = 1;
    string current_file_path = 2;
    uint64 duration_seconds = 3;
    uint64 size_bytes = 4;
}

// 数据采集与管理服务
service DMSService {
  // 发布一条消息到总线
  rpc Publish(BusMessage) returns (PublishResponse);
  // 订阅一个主题
  rpc Subscribe(SubscribeRequest) returns (stream BusMessage);
  // 开始记录指定主题的数据
  rpc StartRecording(StartRecordingRequest) returns (RecorderState);
  // 停止记录
  rpc StopRecording(StopRecordingRequest) returns (RecorderState);
  // 获取当前记录器状态
  rpc GetRecorderState(GetRecorderStateRequest) returns (RecorderState);
}

// ... 详细消息定义请参考核心接口文档 ...
```

## 🛠️ 5. 技术栈与开发环境

### 💻 核心技术栈

| **技术领域**   | **选型**                           | **版本要求**               | **用途说明**           |
| -------------- | ---------------------------------- | -------------------------- | ---------------------- |
| **编程语言**   | **Go (核心服务), Python (客户端)** | `Go 1.21+`, `Python 3.10+` | 高并发服务与易用客户端 |
| **gRPC框架**   | **grpc-go, grpcio (Python)**       | 最新稳定版                 | 实现`.proto`定义的接口 |
| **消息中间件** | **Redis / ZeroMQ**                 | `Redis 6.0+`               | v0.1的本地数据总线后端 |
| **数据序列化** | **Apache Arrow / Parquet**         | 最新稳定版                 | 高效的数据记录格式     |
| **测试框架**   | **Go Native Test, Pytest**         | 最新稳定版                 | 单元测试和接口测试     |

### ⚙️ 系统配置要求

```
# 确保Redis服务已安装并正在运行
$ redis-cli ping
PONG
```

## 🔧 6. 开发实施细节

### 🏗️ 6.1 项目结构 V1

```
services/edge-plane/dms-service/
├── internal/
│   ├── bus/                     # 消息总线核心逻辑
│   │   ├── bus.go
│   │   └── redis_bus.go
│   ├── recorder/                # 数据记录器实现
│   │   └── parquet_recorder.go
│   ├── sync/                    # 云同步代理实现
│   │   └── s3_uploader.go
│   └── server/                  # gRPC服务器实现
│       └── server.go
├── pkg/
│   └── client/                  # 可供其他Go项目引用的客户端库
│       └── client.go
├── api/                         # .proto文件的软链接
├── Dockerfile
├── go.mod
└── README.md
```

### 🧪 6.2 测试与验证策略



- **单元测试:** 对`recorder`和`bus`的核心逻辑进行单元测试。
- **集成测试:**
  - **Pub/Sub正确性测试:** 编写测试验证一个发布者和多个订阅者之间的消息传递。
  - **记录完整性测试:** 发布1000条消息，启动记录，停止记录，然后读取生成的Parquet文件，验证消息数量和内容是否完全一致。
  - **云同步测试:** 模拟上传一个1GB的文件到MinIO，验证上传的完整性和断点续传能力。

------



## 🚀 7. 开发任务 (Getting Started)





#### **第一阶段：核心数据总线 (Core Bus)**



- **任务1：实现DMS核心服务**
  - **交付物:** 一个可独立运行的`dms-service` gRPC服务，严格实现`Publish`和`Subscribe`方法，后端使用Redis Pub/Sub。
- **任务2：开发客户端库**
  - **交付物:** `dms-client-go`和`dms-client-py`两个库，提供简洁的`publish`和`subscribe`接口。
- **任务3：编写Pub/Sub集成测试**
  - **交付物:** 一个测试脚本，验证一个Go发布者和多个Python订阅者之间的通信。



#### **第二阶段：本地数据采集 (Local Recording)**



- **任务1：实现数据记录控制逻辑**
  - **交付物:** 完整实现`Start/Stop/GetRecorderState`的gRPC方法。
- **任务2：实现Parquet格式写入**
  - **交付物:** `Recorder Subscriber`能够将接收到的消息（包括嵌套的Protobuf `Any`类型）正确地写入Parquet文件。
- **任务3：编写记录完整性测试**
  - **交付物:** 提交一份测试报告，证明记录的数据100%无损。



#### **第三阶段：云端同步与可靠性 (Cloud Sync & Reliability)**



- **任务1：实现云同步代理**
  - **交付物:** DMS服务中的后台任务，能将录制完成的文件上传到S3/MinIO。
- **任务2：实现断点续传**
  - **交付物:** 上传功能在网络中断后能够恢复并继续上传。
- **任务3：压力测试**
  - **交付物:** 提交一份测试报告，展示DMS服务在高消息吞吐量（如1000 msg/s）下的CPU、内存占用和消息延迟。



#### **第四阶段：高级功能与优化 (Advanced Features & Optimization)**



- **任务1：实现数据回放功能**
  - **交付物:** 新增`StartReplay` gRPC接口，能够读取Parquet文件并按原速回放数据。
- **任务2：性能优化**
  - **交付物:** 探索使用共享内存等零拷贝技术优化本地模块间的数据传输。
- **任务3：文档与示例完善**
  - **交付物:** 撰写详细的开发者文档，提供丰富的数据采集、回放和调试示例。