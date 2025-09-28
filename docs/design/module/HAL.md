# 🔥 ARF 模块开发参考文档：硬件抽象层 (HAL)

> 🎯 **角色定位:** ARF的"感官"与"神经末梢" - 连接数字世界与物理世界。
>
> 📦 **模块代号:** `arf-edge-hal`
>
> ⚡ **所属:** ARF 边缘平台 (Edge Plane)

## 📋 1. 核心职责与设计理念

### 🎯 核心使命 (Core Mission)

作为ARF的"感官"和"神经末梢"，HAL的核心使命是**屏蔽物理世界所有硬件的复杂性和差异性**，为上层数字世界提供一套**统一、标准、可靠、并具备自我诊断与校准能力**的硬件访问接口。无论底层连接的是ZED相机、Basler工业相机，还是一个普通的USB摄像头，对于系统的其他部分来说，它们都应该“看起来”和“用起来”一样。其进化目标是超越简单的驱动抽象，成为一个能够**主动进行在线自校准**和**预测性维护**的智能硬件平面。

主要应用场景包括：

- **👁️ 传感器数据接入:** 为摄像头、激光雷达、IMU等提供统一的数据出口。
- **🦾 执行器指令下发:** 为电机、机械臂、夹爪等提供统一的控制入口。
- **⚙️ 硬件参数配置:** 提供标准化的方式查询和配置硬件参数（如曝光、增益）。
- **⚙️ 在线自校准:** 提供自动化的校准服务，如机器人手眼标定、内外参校准，极大降低部署和维护成本。
- **🩺 预测性维护:** 持续监控硬件底层遥测数据，通过边缘模型预测潜在故障，实现主动运维。
- **🔌 设备生命周期管理:** 动态发现、初始化和监控硬件的连接与健康状态。

### 🏗️ 核心架构：“驱动即服务” + “健康与校准” (Driver-as-a-Service + Health & Calibration)

我们的HAL设计基于一个核心理念：每一个硬件驱动，都不是一个简单的“库”，而是一个**独立的、长期运行的微服务**（通常打包在Docker容器中），并为其增加了主动的健康管理和校准能力。

- **驱动即插件:** 每个驱动容器在启动时，会向一个中心化的`HALManagerService`进行**“注册”**，报告自己能处理的设备类型。
- **管理器负责枚举:** `HALManagerService`负责与底层操作系统交互（`udev`/`DirectShow`）来发现和管理设备。
- **动态分配:** 当`HALManagerService`发现新设备时，会通知对应的驱动去“认领”并初始化。
- **健康与校准层 (Health & Calibration Layer):** 这是V1.1架构的核心扩展。它是一组新的服务和内置于驱动中的逻辑，负责执行自动校准流程，并持续运行健康诊断与预测模型。

### ⚖️ 设计原则 (Design Principles)

- **🛡️ 安全第一:** 驱动层的稳定是系统物理安全的第一道防线，必须追求极致的稳定和内存安全（首选Rust）。
- **🔌 统一接口:** 对外接口（gRPC）必须严格遵守`.proto`定义，屏蔽内部实现差异。
- **📦 隔离性:** 单个驱动的崩溃或错误，**绝不能**导致ARF核心服务或其他驱动的崩溃。
- **⚙️ 自描述性:** 每个驱动都应能向上层报告自己支持的参数和能力。
- **🤖 自我完善 (Self-Improving):** 硬件平面应具备自我校准和自我诊断的能力，以应对物理世界的磨损和变化。
- **⏱️ 时间感知 (Time-Aware):** `HAL`驱动在向`RTS`提交实时任务时，有责任根据任务的重要性为其指定正确的**关键性等级**。同时，驱动应持续监控`RTS`为其提供的**运行时性能统计**，并将任何延迟或抖动异常作为自身健康状态的一部分进行上报。

## 📝 2. 核心需求 (Core Requirements)



| **ID** | **需求描述**                        | **验收标准**                                                 | **优先级**     |
| ------ | ----------------------------------- | ------------------------------------------------------------ | -------------- |
| **H1** | **统一接口 (Unified Interface)**    | 每一种硬件**类型**（如“相机”）都必须通过一个统一的、标准化的gRPC接口对外提供服务，无论其品牌和型号。 | **最高**       |
| **H2** | **资源隔离 (Resource Isolation)**   | 每个硬件驱动都必须作为一个独立的进程或容器运行。单个驱动的崩溃或错误，**绝不能**导致ARF核心服务或其他驱动的崩溃。 | **最高**       |
| **H3** | **高性能数据流 (High-Perf Stream)** | 对于高数据吞吐量的传感器（如相机），驱动必须能以低延迟、高帧率的方式，将数据发布到DMS数据总线。 | **最高**       |
| **H4** | **动态设备发现与热插拔**            | 系统必须能够自动发现可用设备（如枚举DirectShow/V4L2设备），并在设备连接/断开时，能够处理相应事件并更新设备列表。 | **高**         |
| **H5** | **复杂SDK集成**                     | 架构必须支持深度集成功能复杂、依赖特殊的厂商SDK（如Basler Pylon API）。 | **高**         |
| **H6** | **高级参数控制**                    | 必须提供一个标准化的机制，用于查询和控制特定设备的高级参数（如自动曝光/增益、硬件触发模式）。 | **高**         |
| **H7** | **实时状态监控**                    | 所有硬件驱动必须能以一定的频率，持续地将其关键状态（如温度、连接状态、负载）发布到DMS数据总线，用于实时监控和故障诊断。 | **最高**       |
| **H8** | **在线自校准**                      | **[V1.1+ 新增]** 必须提供标准化的服务接口，用于触发和管理机器人的在线自校准流程（如手眼标定）。 | **高 (V1.1+)** |
| **H9** | **预测性维护**                      | **[V1.1+ 新增]** 驱动必须能上报详细的底层遥测数据，并支持运行健康预测模型，提前预警潜在故障。 | **中 (V1.1+)** |

## ⚙️ 3. 关键功能与子模块架构

### 🧠 3.1 HAL管理器 (HAL Manager)

**门户角色**：`HALManagerService`是所有硬件交互的“门户”，负责设备的发现与生命周期管理。

- **OS适配:**
  - 在**Linux**上，通过监听`udev`事件来发现新设备和处理热插拔。
  - 在**Windows**上，通过轮询**DirectShow**或**Windows Portable Devices** API来枚举设备。
- **事件广播:** 通过`WatchDeviceEvents`这个流式RPC，向整个ARF系统广播设备连接/断开的事件。

### 📸 3.2 驱动服务 (Driver Service) - 以Basler Pylon为例

**适配器角色**：每个驱动服务都是一个“适配器”，将厂商私有的SDK调用，翻译成ARF的标准gRPC接口。

- **数据流:** `硬件 -> Pylon SDK -> Rust gRPC服务 -> [转换为ImageFrame] -> 调用DMS发布 -> DMS数据总线`
- **控制流:** `DIL/遥操作 -> [gRPC调用SetParameters] -> Rust gRPC服务 -> [转换为Pylon API调用] -> 硬件`
- **容器化封装:** `Dockerfile`负责在一个干净的Ubuntu镜像中，完整地安装官方的Pylon SDK及其所有依赖，创建一个完美的、隔离的“生态园”。

### 🎛️ 3.3 通用参数控制器 (Generic Parameter Controller)

**自描述机制**：为了满足需求**H6**，我们引入了一套通用的参数Get/Set机制。

- **参数自描述:** 每个驱动在初始化设备后，都有责任通过`ListParameters`这个RPC，向上层报告它支持的**所有高级参数**的名称、类型、范围和可选值。
- **动态UI生成:** 上层的应用程序（如一个调试工具）可以调用`ListParameters`，然后**动态地为每个相机生成一个独有的、完整的参数设置界面**，无需为任何特定相机硬编码。

### 🛠️ 3.4 自动校准管理器 (Automated Calibration Manager)

**校准工程师角色**：这是一个新的HAL服务，负责编排复杂的自动校准任务。

- **职责:**
  1. 对外提供`CalibrationService` gRPC接口。
  2. 接收到校准请求后（如`StartHandEyeCalibration`），它会作为客户端，调用`MotorActuatorService`和`CameraSensorService`等多个驱动服务。
  3. 它会协调硬件执行一系列精心设计的动作（如移动相机到不同位置拍照），并收集数据。
  4. 调用一个专门的`ACR`校准算法容器来处理收集到的数据，并计算出最终的校准结果（如变换矩阵）。
  5. 将校准结果持久化到机器人的配置文件中。



### 🩺 3.5 健康监控与预测代理 (Health Monitor & Prediction Agent)

**设备医生角色**：这部分能力通常内置于各个驱动服务内部。

- **职责:**
  1. 除了发布标准状态信息外，驱动会以更高频率向一个专门的DMS主题发布详细的底层遥测数据（如电机电流的原始波形、IMU的原始振动数据）。
  2. 驱动内部可以集成一个轻量级的边缘推理引擎（如ONNX Runtime），运行一个由云端`训练模块`训练好的健康预测模型。
  3. 模型根据遥测数据，输出一个硬件健康分值或故障概率，并将其作为标准状态信息的一部分发布出去。

## 🔗 4. 接口设计与数据流

`HAL`模块的对外接口，由`hal.proto`文件严格定义。

### 📥 输入数据流

| **数据源**     | **数据类型**     | **优先级** | **延迟要求** | **示例场景**              |
| -------------- | ---------------- | ---------- | ------------ | ------------------------- |
| **DIL决策层**  | `MotorCommand`   | `HIGH`     | `< 10ms`     | 自主导航                  |
| **遥操作模块** | `TeleopCommand`  | `CRITICAL` | `< 5ms`      | 远程控制                  |
| **API应用层**  | `ParameterValue` | `NORMAL`   | `< 100ms`    | 用户通过WebUI调整相机曝光 |

### 📤 输出数据流

| **目标模块**    | **数据类型**                                 | **保证** | **性能指标**     |
| --------------- | -------------------------------------------- | -------- | ---------------- |
| **DMS数据总线** | `ImageFrame`, `PointCloud`, `HardwareStatus` | 高吞吐量 | 相机帧率 > 30fps |
| **HAL管理器**   | `DeviceEvent`                                | 事件驱动 | 实时响应热插拔   |

### 🧬 核心API草案 (`hal.proto`)

```protobuf
// protos/arf/edge/v1/hal.proto

// 1. HAL 管理器服务
service HALManagerService {
  // 获取当前所有可用的物理设备列表
  rpc ListAvailableDevices(ListAvailableDevicesRequest) returns (ListAvailableDevicesResponse);
  // 订阅设备热插拔事件
  rpc WatchDeviceEvents(WatchDeviceEventsRequest) returns (stream DeviceEvent);
}

// 2. 硬件状态消息
message HardwareStatus {
    // ...
}
// V1.1 新增：力/扭矩传感器数据
message Wrench {
  arf.v1.Header header = 1;
  arf.v1.Vector3 force = 2;
  arf.v1.Vector3 torque = 3;
}

// 硬件健康状态
message HardwareHealthStatus {
    // ...
    double health_score = 1; // 0.0 to 1.0
    string prediction_message = 2; // e.g., "High probability of overheating in next 40 hours"
}


// 3. 相机服务
service CameraSensorService {
  // 获取相机支持的所有高级参数列表
  rpc ListParameters(ListParametersRequest) returns (ListParametersResponse);
  // 获取一个或多个参数的当前值
  rpc GetParameters(GetParametersRequest) returns (GetParametersResponse);
  // 设置一个或多个参数的值
  rpc SetParameters(SetParametersRequest) returns (SetParametersResponse);
  // ... 其他方法
}

// 4. 执行器服务
service MotorActuatorService {
    // ...
}


// 电机执行器服务
service MotorActuatorService { /* ... */ }


// V1.1 新增：力/扭矩传感器服务
service ForceTorqueSensorService {
  rpc StreamWrenches(StreamWrenchesRequest) returns (stream Wrench);
}
message StreamWrenchesRequest {}

// V1.1 新增：校准服务
service CalibrationService {
    // 触发一个手眼标定流程
    rpc StartHandEyeCalibration(StartHandEyeCalibrationRequest) returns (StartHandEyeCalibrationResponse);
}

message StartHandEyeCalibrationRequest {
    string arm_id = 1;
    string camera_id = 2;
}
message StartHandEyeCalibrationResponse {
    bool success = 1;
    arf.v1.Pose result_transform = 2; // 校准结果
}
// ... 详细消息定义请参考核心接口文档 ...
```

## 🛠️ 5. 技术栈与开发环境

### 💻 核心技术栈

| **技术领域** | **选型**                              | **版本要求**            | **用途说明**           |
| ------------ | ------------------------------------- | ----------------------- | ---------------------- |
| **编程语言** | **Rust / C++**                        | `Rust 1.65+` / `C++17+` | 内存安全与底层性能     |
| **gRPC框架** | **Tonic (Rust)**, **grpc/grpc (C++)** | 最新稳定版              | 实现`.proto`定义的接口 |
| **构建系统** | **Cargo**, **CMake**                  | `3.15+`                 | 项目构建与依赖管理     |
| **厂商SDK**  | **Pylon, RealSense SDK, etc.**        | 厂商指定版本            | 与具体硬件交互         |
| **测试框架** | **Cargo Test**, **Google Test**       | `1.11+`                 | 单元测试和接口测试     |

### ⚙️ 系统配置要求

```
# Linux udev规则配置 (用于USB设备权限)
$ cat /etc/udev/rules.d/99-basler-cameras.rules
SUBSYSTEM=="usb", ATTR{idVendor}=="2676", MODE="0666"

# 网络配置 (用于GigE相机)
# 配置Jumbo Frames, 提高网络吞吐量
$ sudo ip link set eno1 mtu 9000
```

## 🔧 6. 开发实施细节

### 🏗️ 6.1 项目结构 V1

```
drivers/
└── camera/
    └── basler-pylon-driver/            # 一个驱动服务的项目目录
        ├── src/
        │   ├── server.rs             # gRPC服务实现
        │   ├── capture.rs            # Pylon API调用与图像捕获逻辑
        │   ├── parameters.rs         # 通用参数控制的实现
        │   └── lib.rs                # 库的根文件
        ├── api/                      # .proto文件的软链接
        ├── build.rs                  # 编译时生成Protobuf代码
        ├── configs/
        │   └── default.toml          # 默认配置文件
        ├── Cargo.toml
        ├── Dockerfile                # 包含Pylon SDK安装步骤
        └── README.md
```

### 🧪 6.2 测试与验证策略



- **单元测试:** 使用`mockall`等库，对非硬件相关的逻辑（如参数转换）进行单元测试。
- **硬件在环测试 (HIL):** 连接真实硬件，编写测试程序验证：
  - 图像数据流的帧率和稳定性。
  - 所有可写参数的Get/Set功能是否符合预期。
  - 热插拔后驱动能否自动重连。
  - 驱动在高负载下长时间运行（>24小时）的稳定性。

------



## 🚀 7. 开发任务 (Getting Started) 



#### **第一阶段：核心框架与虚拟驱动 (Foundation & Virtualization)**



**核心目标：** 搭建HAL的核心服务框架，并开发不依赖真实硬件的虚拟驱动，为上层模块（ACR, DIL）提供稳定的数据源进行并行开发。

- **任务1：实现`HALManagerService`原型**
  - **交付物:** 一个Go或Rust服务，实现`ListAvailableDevices`，至少能在当前操作系统下发现一个USB摄像头。
- **任务2：开发“虚拟摄像头”驱动**
  - **交付物:** 一个Rust服务，无需真实硬件，但能：
    1. 实现`CameraSensorService`。
    2. 实现通用的参数Get/Set接口，并暴露一个可写的虚拟参数（如`brightness`）。
    3. 以1Hz的频率向DMS发布`HardwareStatus`消息。
    4. 以10Hz的频率向DMS发布模拟的`ImageFrame`数据。
- **任务3：开发“虚拟电机”驱动**
  - **交付物:** 一个Rust服务，实现`MotorActuatorService`，在收到指令时仅打印日志。



#### **第二阶段：真实硬件集成与高级功能 (Integration & Advanced Features)**



**核心目标：** 将框架与真实世界的硬件连接起来，并实现热插拔、高级参数控制等关键功能，验证架构的鲁棒性。

- **任务1：集成第一个真实USB摄像头**
  - **交付物:** `webcam-v4l2-driver`，能驱动一个标准的USB摄像头，并实现参数控制。
- **任务2：集成第一个复杂SDK（如Basler Pylon）**
  - **交付物:** `basler-pylon-driver`，完整实现`CameraSensorService`，并支持硬件触发等高级功能。
- **任务3：实现热插拔处理**
  - **交付物:** `HALManagerService`能够通过`WatchDeviceEvents`广播设备连接/断开事件。
- **任务4：与RTS集成**
  - **交付物:** 提供一个电机驱动示例，其高频控制回路（如1kHz PID）由RTS模块调度，验证HAL与RTS的协同工作。



#### **第三阶段：鲁棒性与生态扩展 (Robustness & Ecosystem Expansion)**



**核心目标：** 专注于驱动的稳定性和错误处理，并开始构建可扩展的驱动生态。

- **任务1：完善错误处理与恢复机制**
  - **交付物:** 所有核心驱动在遇到硬件错误（如设备断开、SDK报错）时，能够优雅地处理异常，更新其`HardwareStatus`，并尝试自动重连。
- **任务2：开发驱动开发工具包 (DDK)**
  - **交付物:** 一个包含模板代码、工具和详细文档的GitHub仓库，指导第三方开发者如何为ARF开发新的硬件驱动。
- **任务3：压力与稳定性测试**
  - **交付物:** 提交一份测试报告，证明核心驱动在高负载下长时间运行（>72小时）的稳定性、内存占用和CPU使用率。



#### **第四阶段：性能优化与认证 (Performance & Certification)**



**核心目标：** 对关键驱动进行深度性能优化，并建立官方认证流程，为商业化部署做准备。

- **任务1：实现零拷贝数据传输**
  - **交付物:** 对于支持的硬件，实现从驱动到DMS的零拷贝（或接近零拷贝）数据传输路径，以最大化高分辨率相机的数据吞吐量。
- **任务2：建立驱动认证流水线**
  - **交付物:** 一套自动化的CI/CD流水线，可以对社区或厂商提交的驱动进行一系列标准化测试（API一致性、稳定性、性能基准）。
- **任务3：发布第一批官方认证驱动**
  - **交付物:** 在`插件市场`中，明确标识出哪些驱动是经过ARF官方认证的，并提供质量保证。



#### **第五阶段及以后：迈向自感知硬件平面 (Towards Self-Aware Hardware Plane)**



*此部分为V1.1及后续版本规划，旨在提升硬件的智能化水平。*

- **任务5.1: 实现自动校准管理器**
  - **交付物:** 完整实现`CalibrationService`和`自动校准管理器`。在一个真实的机器人上，能够通过一个gRPC调用，完成一次完整的手眼标定流程，并验证标定结果的精度。
- **任务5.2: 开发健康监控与预测代理原型**
  - **交付物:** 为一个核心驱动（如机械臂驱动）增加详细的遥测数据上报功能。在云端使用这些数据训练一个简单的故障预测模型（如基于LSTM的异常检测）。将模型部署回边缘，驱动能够根据模拟的异常数据，成功发布健康预警。
- **任务5.3: 建立硬件健康仪表盘**
  - **交付物:** 与`舰队管理`模块团队协作，在`舰队管理`的Web UI上，为每个机器人增加一个“硬件健康”页面，可视化展示各部件的健康分值和预测性维护建议。
- **任务5.4: 完善校准服务生态**
  - **交付物:** 增加对更多类型（如相机集群、IMU-轮速计）的自动校准支持，并为开发者提供清晰的文档，指导他们如何为新硬件添加自定义的校准流程。

