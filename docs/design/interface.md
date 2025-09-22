# ARF核心接口定义

本文件定义了ARF项目第一阶段开发所需的所有核心gRPC服务和消息。**所有模块间的交互都必须严格遵守此契约。**


## **Part 1: `common.proto` - 通用数据类型**
*定义了在整个生态系统中被广泛复用的基础数据结构。*

```protobuf
// protos/arf/v1/common.proto
syntax = "proto3";

package arf.v1;

import "google/protobuf/timestamp.pb";

// 标准消息头，用于追踪数据流
message Header {
  uint64 seq_id = 1; // 序列ID
  google.protobuf.Timestamp stamp = 2; // 时间戳
  string frame_id = 3; // 坐标系ID
}

// 三维向量
message Vector3 {
  double x = 1;
  double y = 2;
  double z = 3;
}

// 四元数
message Quaternion {
  double x = 1;
  double y = 2;
  double z = 3;
  double w = 4;
}

// 位姿
message Pose {
  Vector3 position = 1;
  Quaternion orientation = 2;
}

// 插件/模型/驱动的唯一标识符
message PluginIdentifier {
  string name = 1; // e.g., "object-detector-yolov8"
  string version = 2; // e.g., "1.2.0"
}
```

------



## **Part 2: 边缘平面 (Edge Plane) 接口**

## **Part 2: 边缘平面 (Edge Plane) 接口**

### **`dms.proto` - 边缘数据采集与管理服务**(例)

**设计思路:** DMS是系统的“数据传输系统”。消息设计必须兼顾通用性与性能。

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

  // --- 数据采集控制接口 ---
  // 开始记录指定主题的数据
  rpc StartRecording(StartRecordingRequest) returns (RecorderState);
  // 停止记录
  rpc StopRecording(StopRecordingRequest) returns (RecorderState);
  // 获取当前记录器状态
  rpc GetRecorderState(GetRecorderStateRequest) returns (RecorderState);
}

message PublishResponse {}
message SubscribeRequest {
  string topic = 1;
}

// 开始记录的请求
message StartRecordingRequest {
    // 要记录的主题列表，支持通配符
    repeated string topics = 1;
    // (可选) 输出文件路径
    optional string output_path = 2;
}

// 停止记录的请求
message StopRecordingRequest {}
// 获取状态的请求
message GetRecorderStateRequest {}
```

### **`hal.proto` - 硬件抽象层服务**(例)

**设计思路:** HAL的接口必须清晰地反映物理世界的能力。传感器提供数据流，执行器接收指令并返回状态。

```
// protos/arf/edge/v1/hal.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/v1/common.proto";

// 图像帧消息
message ImageFrame {
  arf.v1.Header header = 1;
  bytes data = 2;
  string encoding = 3; // e.g., "jpeg", "rgb8"
}

// 相机传感器服务
service CameraSensorService {
  rpc StreamFrames(StreamFramesRequest) returns (stream ImageFrame);
}
message StreamFramesRequest {}

// 电机指令
message MotorCommand {
  string actuator_id = 1;
  oneof command {
    float target_position = 2;
    float target_velocity = 3;
  }
}

// 电机执行器服务
// ExecuteCommand的调用者可以是DIL（自主模式），也可以是Teleoperation模块（遥操作模式）
service MotorActuatorService {
  rpc ExecuteCommand(MotorCommand) returns (ExecuteCommandResponse);
}
message ExecuteCommandResponse {
  bool success = 1;
}
```

### **3.`teleop.proto` - 遥操作服务**(例)

**定义了遥操作终端与机器人之间的低延迟控制接口**。

```
// protos/arf/edge/v1/teleop.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/edge/v1/hal.proto"; // 导入硬件指令
import "arf/edge/v1/dms.proto"; // 导入数据记录控制

// 遥操作指令流，可以包含多种控制数据
message TeleopCommand {
    // 机器人底盘移动指令 (速度)
    optional Twist chassis_command = 1;
    // 机械臂关节指令
    repeated hal.MotorCommand arm_commands = 2;
    // (可选) 数据记录控制指令
    oneof recording_control {
        dms.StartRecordingRequest start_recording = 3;
        dms.StopRecordingRequest stop_recording = 4;
    }
}

// 机器人底盘速度指令
message Twist {
    // 线速度
    double linear_x = 1;
    double linear_y = 2;
    // 角速度
    double angular_z = 3;
}

// 机器人状态回传流
message RobotFeedback {
    // 视频流 (用于FPV第一人称视角)
    optional hal.ImageFrame video_stream = 1;
    // 机器人本体状态 (电量、速度等)
    optional RobotState robot_state = 2;
    // 数据记录器状态
    optional dms.RecorderState recorder_state = 3;
}

message RobotState {
    double battery_percentage = 1;
    Twist current_velocity = 2;
}


// 遥操作服务定义
service TeleoperationService {
    // 建立一个双向流，用于实时遥操作
    // 客户端持续发送控制指令，服务器持续回传机器人状态
    rpc CommandStream(stream TeleopCommand) returns (stream RobotFeedback);
}
```

### **4. `acr.proto` (算法容器运行时服务)**(例)

**设计思路:** ACR运行时是“算法进程管理器”。它的核心职责是根据指令，管理算法容器的生命周期（启动、停止、监控），并为其注入必要的配置。

```
// protos/acr/v1/runtime.proto
syntax = "proto3";

package arf.acr.v1;

// 启动容器时可以传入的环境变量
message EnvironmentVariable {
  string key = 1;
  string value = 2;
}

// 容器的运行状态
enum ContainerStatus {
  STATUS_UNSPECIFIED = 0;
  RUNNING = 1;
  STOPPED = 2;
  ERROR = 3;
}

// 容器信息
message ContainerInfo {
  string container_id = 1;
  string project_path = 2;
  ContainerStatus status = 3;
}

service ACRRuntimeService {
  // [主要方法] 启动并管理一个算法容器。
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
  
  // [辅助方法] 停止一个正在运行的算法容器。
  rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
  
  // [辅助方法] 获取当前运行时管理的所有容器的状态。
  rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
  
  // [辅助方法] 流式传输指定容器的日志。
  rpc StreamLogs(StreamLogsRequest) returns (stream LogEntry);
}

message StartContainerRequest {
  // 容器的唯一ID，由调用者指定。
  string container_id = 1;
  // 算法项目的本地路径。
  string project_path = 2;
  // 需要注入到容器环境中的变量。
  repeated EnvironmentVariable env_vars = 3;
}

message StartContainerResponse {
  string container_id = 1;
  bool success = 2;
  string message = 3;
}

message StopContainerRequest {
  string container_id = 1;
}
message StopContainerResponse {
  string container_id = 1;
  bool success = 2;
}

message ListContainersRequest {}
message ListContainersResponse {
  repeated ContainerInfo containers = 1;
}

message StreamLogsRequest {
  string container_id = 1;
  // 是否持续追踪日志（类似 tail -f）。
  bool follow = 2;
}

message LogEntry {
  string container_id = 1;
  string line = 2;
}
```



## **Part 3: 云端平面 (Cloud Plane) 接口**

### **`data_hub.proto` - 数据中心服务**(例)

*负责云端海量数据的存储与管理。*

Protocol Buffers

```
// protos/arf/cloud/v1/data_hub.proto
syntax = "proto3";

package arf.cloud.v1;

// 数据中心服务
service DataHubService {
  // 流式上传机器人数据
  rpc UploadData(stream DataPacket) returns (UploadDataResponse);
  // 查询并下载数据集
  rpc QueryData(QueryDataRequest) returns (stream DataPacket);
}

message DataPacket {
  string device_id = 1;
  // 可以是序列化后的 arf.edge.v1.BusMessage
  bytes data = 2;
}
message UploadDataResponse {
  bool success = 1;
  uint64 bytes_received = 2;
}
message QueryDataRequest {
  string query = 1; // e.g., "SELECT * FROM camera_data WHERE timestamp > '...'"
}
```



### **`training.proto` - 训练模块服务**(例)



*负责调度和管理机器学习模型训练任务。*

Protocol Buffers

```
// protos/arf/cloud/v1/training.proto
syntax = "proto3";

package arf.cloud.v1;

import "arf/v1/common.proto";

// 训练模块服务
service TrainingService {
  // 启动一个新的训练任务
  rpc StartTrainingJob(StartTrainingJobRequest) returns (StartTrainingJobResponse);
  // 获取训练任务的状态
  rpc GetTrainingJobStatus(GetTrainingJobStatusRequest) returns (JobStatus);
}

message StartTrainingJobRequest {
  arf.v1.PluginIdentifier training_code_plugin = 1; // 包含训练脚本的插件
  string dataset_query = 2; // 用于从Data Hub查询数据的SQL
  map<string, string> hyperparameters = 3;
}
message StartTrainingJobResponse {
  string job_id = 1;
}
message GetTrainingJobStatusRequest {
  string job_id = 1;
}
message JobStatus {
  string job_id = 1;
  string status = 2; // e.g., "PENDING", "RUNNING", "SUCCEEDED", "FAILED"
  float progress = 3; // 0.0 to 1.0
}
```



### **`marketplace.proto` - 插件市场服务**(例)



*负责所有数字资产（驱动、算法、模型、应用）的管理和分发。*

Protocol Buffers

```
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



### **`fleet.proto`**(例)

*负责对边缘设备（机器人）进行远程管理、部署和监控。*

Protocol Buffers

```
// protos/arf/cloud/v1/fleet.proto
syntax = "proto3";

package arf.cloud.v1;

import "arf/v1/common.proto";

// 舰队管理服务
service FleetManagementService {
  // 向指定设备或设备组部署一个插件
  rpc DeployPlugin(DeployPluginRequest) returns (DeployPluginResponse);
  // 获取一个设备的实时状态
  rpc GetDeviceStatus(GetDeviceStatusRequest) returns (DeviceStatus);
  // 向指定设备发送一个远程指令（如重启、运行诊断）
  rpc SendCommandToDevice(SendCommandRequest) returns (SendCommandResponse);
}

message DeployPluginRequest {
  repeated string device_ids = 1;
  arf.v1.PluginIdentifier plugin = 2;
}
message DeployPluginResponse {
  string deployment_id = 1;
}
message GetDeviceStatusRequest {
  string device_id = 1;
}
message DeviceStatus {
  string device_id = 1;
  string status = 2; // e.g., "ONLINE", "OFFLINE"
  double battery_level = 3;
  // ... 其他遥测数据
}
message SendCommandRequest {
  string device_id = 1;
  string command = 2; // e.g., "reboot", "run_diagnostics"
}
message SendCommandResponse {
  bool success = 1;
}
```



