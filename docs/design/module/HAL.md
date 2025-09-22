# ARF 模块深度解析：HAL (硬件抽象层) V1.1

文档版本: 1.1 (生产级设计)

模块负责人: 开发者 B (执行工程师)

状态: 设计完成 - 待开发

## 1. 模块概述 (Module Overview)

### 1.1 核心使命 (Core Mission)

HAL是ARF框架的“**感官与神经末梢**”。它的唯一使命是**屏蔽物理世界所有硬件的复杂性和差异性**，为上层数字世界提供一套**统一、标准、可靠**的硬件访问接口。无论底层连接的是ZED相机还是一个普通的USB摄像头，对于系统的其他部分来说，它们都应该“看起来”和“用起来”一样。

### 1.2 所属平面 (Plane)

ARF 边缘平面 (Edge Plane)

### 1.3 主选技术栈 (Primary Tech Stack)

- **`Rust`:** 作为新驱动开发的首选语言，利用其内存安全特性，从根本上保证驱动的稳定性和安全性。
- **`C++`:** 用于封装那些只提供C/C++ SDK的、复杂的、高性能的硬件。

## 2. 核心需求 (Core Requirements)

| **ID**             | **需求描述**                | **验收标准**                                                 | **优先级**          |
| ------------------ | --------------------------- | ------------------------------------------------------------ | ------------------- |
| **H1**             | **硬件设备支持**            | HAL必须能够支持多类别硬件，至少包括：相机、IMU、执行器（电机）。 | **最高**            |
| **H2**             | **统一接口**                | 每一种硬件**类型**（如“相机”）都必须通过一个统一的、标准化的gRPC接口对外提供服务，无论其品牌和型号。 | **最高**            |
| **H3**             | **高性能数据流**            | 对于高数据吞吐量的传感器（如相机），驱动必须能以低延迟、高帧率的方式，将数据发布到DMS数据总线。 | **最高**            |
| **H4**             | **资源隔离**                | 每个硬件驱动都必须作为一个独立的进程或容器运行。单个驱动的崩溃或错误，**绝不能**导致ARF核心服务或其他驱动的崩溃。 | **最高**            |
| **H5**             | **数据标准化**              | 所有驱动在输出数据前，**必须**将硬件的私有数据格式，转换为在`.proto`中定义的ARF标准消息格式（如`ImageFrame`）。 | **高**              |
| **H6**             | **配置驱动**                | 驱动的行为（如相机分辨率、电机ID）必须是可以通过外部配置（环境变量、配置文件）进行管理的，严禁硬编码。 | **高**              |
| **H7**             | **健康状态报告**            | 每个驱动服务都必须能报告自身的健康状况（如“已连接”、“错误”、“断开连接”）。 | **中**              |
| **H8**             | **动态设备发现与热插拔**    | 系统必须能够自动发现可用设备（如枚举DirectShow/V4L2设备），并在设备连接/断开时，能够处理相应事件并更新设备列表。 | **高**              |
| **H9**             | **复杂SDK集成**             | 架构必须支持深度集成功能复杂、依赖特殊的厂商SDK（如Basler Pylon API）。 | **高**              |
| **H10**            | **高级参数控制**            | 必须提供一个标准化的机制，用于查询和控制特定设备的高级参数（如自动曝光/增益、硬件触发模式）。 | **高**              |
| **<ins>H11</ins>** | **<ins>实时状态监控</ins>** | <ins>所有硬件驱动必须能以一定的频率，持续地将其关键状态（如温度、连接状态、负载）发布到DMS数据总线，用于实时监控和故障诊断。</ins> | **<ins>最高</ins>** |

## 3. 接口定义 (Interface Definition) - **重大扩展**

为了满足新的需求，我们必须对`hal.proto`进行重大扩展，引入**设备管理**、**通用参数控制**以及**标准化的状态消息**。

**文件路径:** `api/arf/edge/v1/hal.proto` (修订版)

```
// protos/arf/edge/v1/hal.proto
syntax = "proto3";

package arf.hal.v1;

import "arf/dms/v1/bus.proto";
import "google/protobuf/any.pb";

// ===================================================================
//  1. HAL 管理器服务 (新增)
//  负责设备的发现与生命周期管理
// ===================================================================

// 描述一个可用的物理设备
message PhysicalDevice {
  string device_id = 1;       // 唯一的设备标识符, e.g., "pylon://12345" or "v4l2:///dev/video0"
  string device_type = 2;     // 设备类型, e.g., "camera", "imu"
  string vendor_name = 3;     // 厂商名称, e.g., "Basler", "Logitech"
  string model_name = 4;      // 型号名称, e.g., "acA1920-40gc", "C920"
  string serial_number = 5;   // 序列号
}

// 设备事件类型
enum DeviceEventType {
  DEVICE_EVENT_TYPE_UNSPECIFIED = 0;
  DEVICE_ATTACHED = 1; // 设备已连接
  DEVICE_DETACHED = 2; // 设备已断开
}

// 设备事件消息
message DeviceEvent {
  DeviceEventType event_type = 1;
  PhysicalDevice device = 2;
}

service HALManagerService {
  // 获取当前所有可用的物理设备列表
  rpc ListAvailableDevices(ListAvailableDevicesRequest) returns (ListAvailableDevicesResponse);
  // 订阅设备热插拔事件
  rpc WatchDeviceEvents(WatchDeviceEventsRequest) returns (stream DeviceEvent);
}

message ListAvailableDevicesRequest {}
message ListAvailableDevicesResponse {
  repeated PhysicalDevice devices = 1;
}
message WatchDeviceEventsRequest {}


// ===================================================================
//  2. 硬件状态消息 (新增)
//  所有驱动都必须发布的状态消息
// ===================================================================
enum ComponentStatus {
    STATUS_UNKNOWN = 0;     // 状态未知
    STATUS_OK = 1;          // 正常工作
    STATUS_WARN = 2;        // 存在警告，但仍可工作 (如温度偏高)
    STATUS_ERROR = 3;       // 发生错误，无法工作
    STATUS_OFFLINE = 4;     // 设备离线/未连接
}

// 标准化的硬件状态消息，所有驱动都应以固定频率发布此消息到DMS
// Topic命名规范: hal.status.<device_type>.<device_id>
// e.g., hal.status.camera.basler_12345
message HardwareStatus {
    arf.dms.v1.Header header = 1;
    ComponentStatus status = 2;
    string message = 3; // 状态的详细描述信息
    // (可选) 附加的诊断信息
    map<string, google.protobuf.Any> diagnostics = 4; // e.g., {"temperature_celsius": 75.5, "voltage": 12.1}
}


// ===================================================================
//  3. 相机服务 (扩展)
//  增加了通用的参数控制接口
// ===================================================================

// 通用参数类型
enum ParameterType {
  PARAMETER_TYPE_UNSPECIFIED = 0;
  INTEGER = 1;
  FLOAT = 2;
  BOOLEAN = 3;
  STRING = 4;
  ENUM = 5; // 枚举类型
}

// 通用参数定义
message ParameterDef {
  string name = 1;                 // 参数名称, e.g., "exposure_auto"
  ParameterType type = 2;
  bool read_only = 3;
  // (可选) 对于数值类型，定义范围
  oneof range {
    IntRange int_range = 4;
    FloatRange float_range = 5;
  }
  // (可选) 对于枚举类型，定义可选值
  repeated string enum_values = 6;
}

message IntRange { int64 min = 1; int64 max = 2; }
message FloatRange { double min = 1; double max = 2; }

// 通用参数值
message ParameterValue {
  string name = 1;
  google.protobuf.Any value = 2; // 使用Any来承载不同类型的值
}

service CameraSensorService {
  // ... StreamFrames 和 GetCameraInfo 保持不变 ...
  
  // [新增] 获取相机支持的所有高级参数列表
  rpc ListParameters(ListParametersRequest) returns (ListParametersResponse);
  // [新增] 获取一个或多个参数的当前值
  rpc GetParameters(GetParametersRequest) returns (GetParametersResponse);
  // [新增] 设置一个或多个参数的值
  rpc SetParameters(SetParametersRequest) returns (SetParametersResponse);
}

message ListParametersRequest {}
message ListParametersResponse {
  repeated ParameterDef parameters = 1;
}
message GetParametersRequest {
  repeated string names = 1;
}
message GetParametersResponse {
  repeated ParameterValue values = 1;
}
message SetParametersRequest {
  repeated ParameterValue values = 1;
}
message SetParametersResponse {
  bool success = 1;
  string message = 2;
}

// ... MotorActuatorService 保持不变 ...
```





## 4. 设计思路与架构 (V1.1 详解)





### 4.1 解决设备发现与热插拔 (H8)



我们引入一个**`HALManagerService`**。这个服务是所有硬件交互的“门户”。

- **设计思路:**
  - **驱动即插件:** 每个驱动容器（如`basler-driver`, `webcam-driver`）在启动时，会向一个中心化的`HALManagerService`进行**“注册”**，报告自己能处理的设备类型。
  - **管理器负责枚举:** `HALManagerService`负责与底层操作系统交互。
    - 在**Linux**上，它通过监听`udev`事件来发现新设备和处理热插拔。
    - 在**Windows**上，它可以通过轮询**DirectShow**或**Windows Portable Devices** API来枚举设备。
  - **动态分配:** 当`HALManagerService`发现一个新设备时（如一个Basler相机插入USB口），它会查找已注册的、能处理该设备的驱动（`basler-driver`），然后通知该驱动去“认领”并初始化这个设备。
  - **事件广播:** `HALManagerService`通过`WatchDeviceEvents`这个流式RPC，向整个ARF系统广播设备连接/断开的事件，让上层应用（如UI）可以动态更新设备列表。



### 4.2 解决复杂SDK集成 (H9) - 以Basler Pylon为例



我们的“驱动即服务”模型非常适合封装这类复杂的SDK。

- **设计思路:** 我们将创建一个`arf-driver-basler`的独立项目。
  - **`Dockerfile`:** 这个容器的构建文件，其核心任务是在一个干净的Ubuntu镜像中，**完整地安装官方的Pylon SDK及其所有依赖**。这将创建一个完美的、隔离的“生态园”。
  - **`Rust/C++` gRPC服务:**
    1. 在服务启动时，使用Pylon API的`TlFactory.EnumerateDevices()`来发现所有连接的Basler相机。
    2. 将发现的设备信息，上报给`HALManagerService`。
    3. 实现`CameraSensorService`的gRPC接口。当`StreamFrames`被调用时，它内部会使用Pylon的`Pylon::CInstantCamera`来配置相机、抓取图像。
    4. 将Pylon的图像格式（`GrabResult`）转换为我们的ARF标准`ImageFrame`消息，然后发布到DMS。



### 4.3 解决高级参数控制 (H10)



我们引入了一套**通用的参数Get/Set机制**，而不是为每个参数（曝光、增益、触发）都定义一个专门的RPC。

- **设计思路:**
  - **参数自描述:** 每个驱动在初始化设备后，都有责任通过`ListParameters`这个RPC，向上层报告它支持的**所有高级参数**的名称、类型、范围和可选值。
    - **Basler驱动**可能会报告：`{name: "exposure_auto", type: ENUM, enum_values: ["Off", "Once", "Continuous"]}` 和 `{name: "trigger_mode", type: ENUM, enum_values: ["On", "Off"]}`。
    - **普通Webcam驱动**可能只会报告一个简单的`{name: "brightness", type: INTEGER, range: {min:0, max:255}}`。
  - **动态UI生成:** 上层的应用程序（如一个调试工具）可以调用`ListParameters`，然后**动态地为每个相机生成一个独有的、完整的参数设置界面**，无需为任何特定相机硬编码。
  - **统一控制:** 无论底层是哪个品牌的相机，上层应用都使用同一个`SetParameters` RPC来控制它。例如，要开启硬件触发，只需发送一个`{name: "trigger_mode", value: "On"}`的`ParameterValue`即可。驱动内部负责将这个标准请求，翻译成对应的Pylon SDK API调用。



### <ins>4.4 解决实时状态监控 (H11)</ins>



**这是本次修订的核心。** 我们将状态监控设计为一种**标准化的、基于DMS数据总线的发布/订阅模式**，而不是通过RPC轮询。

- **设计思路:**
  - **驱动的主动责任:** **每一个**HAL驱动服务，在成功初始化一个物理设备后，**必须**启动一个独立的后台任务（线程/goroutine）。
  - **周期性发布:** 这个后台任务的唯一职责，就是以一个**固定的频率**（例如1Hz），主动地、周期性地**发布一个`HardwareStatus`消息**到DMS数据总线。
  - **标准化的主题:** 发布的主题必须遵循严格的命名规范：`hal.status.<device_type>.<device_id>`。例如，一个序列号为`12345`的Basler相机的状态主题就是 `hal.status.camera.basler_12345`。
  - **上层统一订阅:** 任何需要监控硬件状态的模块（如`DIL`的安全模块、`API`层的WebUI），都只需要向`DMS`订阅`hal.status.*`这个通配符主题，即可接收到**所有**硬件的状态更新，无需知道具体的硬件类型和数量。
  - **心跳机制:** 这个周期性的状态发布，自然地成为了一个**心跳信号**。如果监控模块在超过一个预设的阈值（如5秒）内没有收到某个设备的状态消息，就可以判定该设备驱动已离线或发生严重故障。



## 5. 第一阶段（Phase 1）开发任务 (修订)



- **任务1 (不变):** 与团队共同敲定`hal.proto` **修订版**的最终细节。
- **任务2 (修订):** 开发`HALManagerService`原型。
  - **功能:** 实现`ListAvailableDevices`，至少能在当前操作系统下（Linux/Windows）发现一个USB摄像头。实现`WatchDeviceEvents`的骨架。
- **任务3 (修订):-** 开发“虚拟摄像头”和“虚拟电机”驱动服务，并：
  - **实现新的通用参数接口**，至少暴露一个可读写的参数（如`brightness`）。
  - **实现状态发布机制**，以1Hz的频率向`DMS`发布`HardwareStatus`消息。