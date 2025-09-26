# ARF核心接口定义 V1.1



**文档目的:** 本文件定义了ARF项目V1.1版本所需的所有核心gRPC服务和消息。所有模块间的交互都必须严格遵守此契约。**



## **Part 1: `common.proto` - 通用数据类型**



*定义了在整个生态系统中被广泛复用的基础数据结构。*

Protocol Buffers

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

// [V1.1 新增] 力与力矩
message Wrench {
    Header header = 1;
    Vector3 force = 2;
    Vector3 torque = 3;
}

// 插件/模型/驱动的唯一标识符
message PluginIdentifier {
  string name = 1; // e.g., "object-detector-yolov8"
  string version = 2; // e.g., "1.2.0"
}
```

------



## **Part 2: 边缘平面 (Edge Plane) 接口**





### **1. `dms.proto` - 数据采集与管理服务**



**V1.1 变更:** 增加了事件触发式记录的配置接口，以实现智能数据捕获。

Protocol Buffers

```protobuf
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

// [V1.1 新增] 事件触发器类型
enum TriggerType {
    TRIGGER_TYPE_UNSPECIFIED = 0;
    ON_TOPIC_MESSAGE = 1; // 当指定主题接收到消息时触发
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
  
  // [V1.1 新增] 配置一个事件触发式记录规则
  rpc ConfigureRecordingTrigger(ConfigureRecordingTriggerRequest) returns (ConfigureRecordingTriggerResponse);
}

message PublishResponse {}

message SubscribeRequest {
  string topic = 1;
}

message StartRecordingRequest {
    // 要记录的主题列表，支持通配符 e.g., ["/camera/front/image", "/robot/state"]
    repeated string topics = 1;
    // (可选) 输出文件路径
    optional string output_path = 2;
}

message StopRecordingRequest {}

message GetRecorderStateRequest {}

// [V1.1 新增] 事件触发式记录的配置请求
message ConfigureRecordingTriggerRequest {
    string trigger_id = 1; // 触发器的唯一标识
    TriggerType trigger_type = 2;
    // ... 触发器的详细参数，如topic, 消息内容匹配规则等
    int32 pre_event_duration_sec = 3;  // 记录事件前多少秒的数据
    int32 post_event_duration_sec = 4; // 记录事件后多少秒的数据
}

// [V1.1 新增] 配置事件触发式记录的响应
message ConfigureRecordingTriggerResponse {
    bool success = 1;
    string message = 2;
}
```



### **2. `hal.proto` - 硬件抽象层服务**



**V1.1 变更:** 正式加入力/扭矩传感器和在线自校准服务，为高级物理交互和自动化运维提供支持。

Protocol Buffers

```protobuf
// protos/arf/edge/v1/hal.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/v1/common.proto";

// 图像帧消息
message ImageFrame {
  arf.v1.Header header = 1;
  bytes data = 2;
  string encoding = 3; // e.g., "jpeg", "rgb8", "bgr8"
  uint32 height = 4;
  uint32 width = 5;
}

// 相机传感器服务
service CameraSensorService {
  rpc StreamFrames(StreamFramesRequest) returns (stream ImageFrame);
}
message StreamFramesRequest {
    string camera_id = 1;
    // ... 可选的帧率、分辨率等参数
}

// 电机指令
message MotorCommand {
  string actuator_id = 1;
  oneof command {
    float target_position = 2; // 目标位置（例如：弧度）
    float target_velocity = 3; // 目标速度（例如：弧度/秒）
  }
}

// 电机执行器服务
// ExecuteCommand的调用者可以是DIL（自主模式），也可以是Teleoperation模块（遥操作模式）
service MotorActuatorService {
  rpc ExecuteCommand(MotorCommand) returns (ExecuteCommandResponse);
}
message ExecuteCommandResponse {
  bool success = 1;
  string message = 2;
}

// [V1.1 新增] 力/扭矩传感器服务
service ForceTorqueSensorService {
  rpc StreamWrenches(StreamWrenchesRequest) returns (stream arf.v1.Wrench);
}
message StreamWrenchesRequest {
    string sensor_id = 1;
}

// [V1.1 新增] 在线校准服务
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
    arf.v1.Pose result_transform = 2; // 校准结果: 从机械臂末端到相机的变换矩阵
    string message = 3;
}
```



### **3. `teleop.proto` - 遥操作服务**



**V1.1 变更:** 接口全面升级，支持力反馈和AR辅助信息，以实现沉浸式协同触觉交互。

Protocol Buffers

```protobuf
// protos/arf/edge/v1/teleop.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/v1/common.proto";
import "arf/edge/v1/hal.proto";
import "arf/edge/v1/dms.proto";

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

// [V1.1 新增] 力反馈信号
message HapticFeedback {
    string device_id = 1; // 作用于哪个力反馈设备
    arf.v1.Wrench wrench = 2;
}

// [V1.1 新增] AR叠加层元数据
message AROverlayMetadata {
    // 此处可定义AR元素，例如：
    // 1. 在3D空间中绘制一个box
    // 2. 在屏幕特定位置显示文本
    // 3. 高亮显示某个检测到的物体
    string data_json = 1; // 使用JSON等灵活格式定义AR元素
}

// 机器人状态回传流
message RobotFeedback {
    // V1.1 将多个反馈信号整合到 oneof 中，以优化流式传输
    oneof feedback_data {
        // 视频流 (用于FPV第一人称视角)
        hal.ImageFrame video_stream = 1;
        // 机器人本体状态 (电量、速度等)
        RobotState robot_state = 2;
        // 数据记录器状态
        dms.RecorderState recorder_state = 3;
        // [V1.1 新增] 力反馈信号
        HapticFeedback haptic_feedback = 4;
        // [V1.1 新增] AR叠加层元数据
        AROverlayMetadata ar_metadata = 5;
    }
}

message RobotState {
    double battery_percentage = 1;
    Twist current_velocity = 2;
    // ... 可补充更多状态，如关节角度、里程计等
}

// 遥操作服务定义
service TeleoperationService {
    // 建立一个双向流，用于实时遥操作
    // 客户端持续发送控制指令，服务器持续回传机器人状态
    rpc CommandStream(stream TeleopCommand) returns (stream RobotFeedback);
}
```



### **4. `acr.proto` - 算法容器运行时服务**



**V1.1 变更:** 增加算法图（DAG）的编排能力，支持更复杂、高性能的算法流水线。

Protocol Buffers

```protobuf
// protos/arf/acr/v1/acr.proto
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

// 算法容器运行时服务
service ACRRuntimeService {
  // [主要方法] 启动并管理一个算法容器。
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
  
  // [辅助方法] 停止一个正在运行的算法容器。
  rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
  
  // [辅助方法] 获取当前运行时管理的所有容器的状态。
  rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
  
  // [辅助方法] 流式传输指定容器的日志。
  rpc StreamLogs(StreamLogsRequest) returns (stream LogEntry);

  // [V1.1 新增] 启动并管理一个算法图
  rpc StartGraph(StartGraphRequest) returns (StartGraphResponse);

  // [V1.1 新增] 停止一个正在运行的算法图
  rpc StopGraph(StopGraphRequest) returns (StopGraphResponse);
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
  google.protobuf.Timestamp timestamp = 3;
}

// [V1.1 新增] 启动算法图请求
message StartGraphRequest {
    string graph_id = 1;
    string graph_yaml_base64 = 2; // Base64编码的算法图YAML定义
}
message StartGraphResponse {
    string graph_id = 1;
    bool success = 2;
    string message = 3;
}

// [V1.1 新增] 停止算法图请求
message StopGraphRequest {
    string graph_id = 1;
}
message StopGraphResponse {
    string graph_id = 1;
    bool success = 2;
}
```



### **5. `dil.proto` - 决策智能层服务 (V1.1 新增)**



**V1.1 变更:** 新增决策智能层，负责任务的解析、规划与执行，并提供模式切换接口作为人机协同与失败学习的总开关。

Protocol Buffers

```protobuf
// protos/arf/edge/v1/dil.proto
syntax = "proto3";

package arf.edge.v1;

import "google/protobuf/struct.pb";

enum OperationMode {
    AUTONOMOUS = 0;    // 自主模式
    TELEOPERATION = 1; // 遥操作模式
}

enum TaskExecutionStatus {
    TASK_STATUS_UNSPECIFIED = 0;
    PENDING = 1;
    EXECUTING = 2;
    SUCCEEDED = 3;
    FAILED = 4;
    CANCELLED = 5;
}

message TaskStatus {
    string task_id = 1;
    TaskExecutionStatus status = 2;
    string message = 3;
    float progress = 4; // 0.0 to 1.0
}

// 决策智能层服务
service DILService {
  // 提交一个高级任务（例如："把桌子上的红色杯子拿到厨房"）
  rpc SubmitTask(SubmitTaskRequest) returns (SubmitTaskResponse);
  // 获取任务状态
  rpc GetTaskStatus(GetTaskStatusRequest) returns (TaskStatus);
  // 取消一个任务
  rpc CancelTask(CancelTaskRequest) returns (CancelTaskResponse);

  // [V1.1 新增] 手动设置机器人操作模式
  rpc SetOperationMode(SetOperationModeRequest) returns (SetOperationModeResponse);
}

message SubmitTaskRequest {
    string task_description = 1; // 自然语言或结构化任务描述
    google.protobuf.Struct parameters = 2; // 任务参数
}
message SubmitTaskResponse {
    string task_id = 1;
}

message GetTaskStatusRequest {
    string task_id = 1;
}

message CancelTaskRequest {
    string task_id = 1;
}
message CancelTaskResponse {
    bool success = 1;
}

message SetOperationModeRequest {
    OperationMode mode = 1;
}
message SetOperationModeResponse {
    bool success = 1;
}
```



### **6. `api_gateway.proto` - 应用层网关服务 (V1.1 新增)**



**V1.1 变更:** 新增应用层网关，为上层应用（如App）提供统一、简化的入口，并为“技能商店”提供后端接口，实现应用的动态扩展和个性化。

Protocol Buffers

```protobuf
// protos/arf/edge/v1/api_gateway.proto
syntax = "proto3";

package arf.edge.v1;

import "arf/edge/v1/dil.proto";
import "arf/edge/v1/teleop.proto";

// 应用层网关服务
service ApiGatewayService {
  // --- 任务管理代理 ---
  rpc StartTask(dil.SubmitTaskRequest) returns (dil.SubmitTaskResponse);
  rpc GetTaskStatus(dil.GetTaskStatusRequest) returns (dil.TaskStatus);

  // --- 状态订阅 ---
  rpc StreamRobotState(StreamRobotStateRequest) returns (stream RobotState);

  // [V1.1 新增] 技能商店接口
  rpc ListAvailableSkills(ListAvailableSkillsRequest) returns (ListAvailableSkillsResponse);
  rpc ManageSkill(ManageSkillRequest) returns (ManageSkillResponse);

  // [V1.1 新增] 用户偏好接口
  rpc GetUserPreferences(GetUserPreferencesRequest) returns (UserPreferences);
  rpc UpdateUserPreferences(UpdateUserPreferencesRequest) returns (UserPreferences);
}

message StreamRobotStateRequest {}

// --- 技能商店消息 ---
message Skill {
    string skill_id = 1;
    string name = 2;
    string description = 3;
    bool is_enabled = 4;
}

message ListAvailableSkillsRequest {}
message ListAvailableSkillsResponse {
    repeated Skill skills = 1;
}

message ManageSkillRequest {
    string skill_id = 1;
    bool enable = 2;
}
message ManageSkillResponse {
    bool success = 1;
}

// --- 用户偏好消息 ---
message UserPreferences {
    map<string, string> preferences = 1; // e.g., "language": "zh-CN"
}
message GetUserPreferencesRequest {}
message UpdateUserPreferencesRequest {
    UserPreferences preferences = 1;
}
```

------



## **Part 3: 云端平面 (Cloud Plane) 接口**





### **1. `data_hub.proto` - 数据中心服务**



**V1.1 变更:** 全面增强数据治理与发现能力，从“数据湖”升级为“主动数据引擎”。

Protocol Buffers

```protobuf
// protos/arf/cloud/v1/data_hub.proto
syntax = "proto3";

package arf.cloud.v1;

// 数据中心服务
service DataHubService {
  // 流式上传机器人数据
  rpc UploadData(stream DataPacket) returns (UploadDataResponse);
  // 查询并下载数据集
  rpc QueryData(QueryDataRequest) returns (stream DataPacket);

  // [V1.1 新增] 数据治理与发现接口
  rpc SemanticSearch(SemanticSearchRequest) returns (QueryDataResponse);
  rpc GetDataLineage(GetDataLineageRequest) returns (DataLineageResponse);
  rpc GetDataQualityReport(GetDataQualityReportRequest) returns (DataQualityReport);
}

message DataPacket {
  string device_id = 1;
  // 可以是序列化后的 arf.edge.v1.BusMessage 或其他数据块
  bytes data = 2;
  google.protobuf.Timestamp timestamp = 3;
}

message UploadDataResponse {
  bool success = 1;
  uint64 bytes_received = 2;
}

message QueryDataRequest {
  string query = 1; // e.g., "SELECT * FROM camera_data WHERE timestamp > '...'"
}

// V1.1 新增消息
message SemanticSearchRequest {
    string text_query = 1; // e.g., "Find all data segments with failed grasps in warehouse A"
}
message QueryDataResponse {
    repeated string result_urls = 1; // 返回匹配的数据集URI
}

message GetDataLineageRequest {
    string data_asset_id = 1;
}
message DataLineageResponse {
    string lineage_graph_json = 1; // 返回数据血缘关系的图表示
}

message GetDataQualityReportRequest {
    string data_asset_id = 1;
}
message DataQualityReport {
    string report_json = 1; // 返回数据质量报告
}
```



### **2. `training.proto` - 训练模块服务**



**V1.1 变更:** 增加对Sim-to-Real和联邦学习的支持，迈向自动化模型进化平台。

Protocol Buffers

```protobuf
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

  // [V1.1 新增] 联邦学习客户端接口
  rpc SubmitModelUpdate(SubmitModelUpdateRequest) returns (SubmitModelUpdateResponse);
  rpc GetGlobalModel(GetGlobalModelRequest) returns (GetGlobalModelResponse);
}

message StartTrainingJobRequest {
  arf.v1.PluginIdentifier training_code_plugin = 1; // 包含训练脚本的插件
  string dataset_query = 2; // 用于从Data Hub查询数据的SQL
  map<string, string> hyperparameters = 3;

  // [V1.1 新增] 训练模式
  oneof training_mode {
      SimToRealStrategy sim_to_real_strategy = 4;
      FederatedLearningConfig federated_learning_config = 5;
  }
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
  string details = 4;
}

// V1.1 新增消息
message SimToRealStrategy {
    bool domain_randomization_enabled = 1;
    // ... 其他Sim-to-Real相关参数
}
message FederatedLearningConfig {
    int32 num_rounds = 1;
    float client_learning_rate = 2;
}

message SubmitModelUpdateRequest {
    string job_id = 1;
    string client_id = 2;
    bytes model_diff = 3; // 客户端计算出的模型更新（梯度或权重差异）
}
message SubmitModelUpdateResponse {
    bool accepted = 1;
}

message GetGlobalModelRequest {
    string job_id = 1;
}
message GetGlobalModelResponse {
    int32 model_version = 1;
    bytes global_model_data = 2; // 全局聚合后的模型
}
```



### **3. `marketplace.proto` - 插件市场服务**



**V1.1 变更:** 无重大变更。其V1.0设计已具备良好的扩展性。

Protocol Buffers

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
  rpc GetPlugin(GetPluginRequest) returns (PluginDetails);
  // 下载插件（获取下载链接或直接流式传输）
  rpc DownloadPlugin(DownloadPluginRequest) returns (DownloadPluginResponse);
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
  string description = 3;
  string author = 4;
  // 插件包（如.zip, .tar.gz）的数据
  bytes package_data = 5;
}

message GetPluginRequest {
    arf.v1.PluginIdentifier id = 1;
}

message PluginDetails {
  arf.v1.PluginIdentifier id = 1;
  PluginType type = 2;
  string description = 3;
  repeated string versions = 4; // 列出所有可用版本
  string author = 5;
}

message DownloadPluginRequest {
    arf.v1.PluginIdentifier id = 1;
}

message DownloadPluginResponse {
  string download_url = 1;
  // 或者 oneof { string download_url = 1; bytes data = 2; }
}
```



### **4. `fleet.proto` - 舰队管理服务**



**V1.1 变更:** 增加A/B测试实验接口，赋能数据驱动的群体智能优化。

Protocol Buffers

```protobuf
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

  // [V1.1 新增] A/B测试实验接口
  rpc StartExperiment(StartExperimentRequest) returns (StartExperimentResponse);
  rpc GetExperimentStatus(GetExperimentStatusRequest) returns (ExperimentStatus);
}

message DeployPluginRequest {
  repeated string device_ids = 1;
  arf.v1.PluginIdentifier plugin = 2;
}
message DeployPluginResponse {
  string deployment_id = 1;
  bool success = 2;
}

message GetDeviceStatusRequest {
  string device_id = 1;
}
message DeviceStatus {
  string device_id = 1;
  string status = 2; // e.g., "ONLINE", "OFFLINE", "MAINTENANCE"
  double battery_level = 3;
  // ... 其他遥测数据
  arf.v1.Pose pose = 4;
}

message SendCommandRequest {
  string device_id = 1;
  string command = 2; // e.g., "reboot", "run_diagnostics"
  map<string, string> args = 3;
}
message SendCommandResponse {
  bool success = 1;
  string result = 2;
}

// V1.1 新增消息
message ExperimentGroup {
    string group_name = 1; // e.g., "control_group", "treatment_group_A"
    repeated string device_ids = 2;
    arf.v1.PluginIdentifier plugin_to_deploy = 3; // 该组部署的插件版本
}

message StartExperimentRequest {
    string experiment_name = 1;
    repeated ExperimentGroup groups = 2;
    string kpi_to_measure = 3; // e.g., "task_success_rate"
}
message StartExperimentResponse {
    string experiment_id = 1;
}

message GetExperimentStatusRequest {
    string experiment_id = 1;
}
message ExperimentStatus {
    string experiment_id = 1;
    string status = 2; // "RUNNING", "COMPLETED"
    string results_summary_json = 3;
}
```



### **5. `evaluation.proto` - 评测模块服务 (V1.1 新增)**



**V1.1 变更:** 正式定义评测模块的gRPC服务，支持多种评测模式，是连接仿真与现实的“质量守门员”。

Protocol Buffers

```protobuf
// protos/arf/cloud/v1/evaluation.proto
syntax = "proto3";

package arf.cloud.v1;

import "arf/v1/common.proto";

// 评测模块服务
service EvaluationService {
  // 运行一个新的评测任务
  rpc RunEvaluation(RunEvaluationRequest) returns (RunEvaluationResponse);
  // 获取评测任务状态
  rpc GetEvaluationStatus(GetEvaluationStatusRequest) returns (EvaluationStatus);
  // 获取评测报告
  rpc GetEvaluationReport(GetEvaluationReportRequest) returns (EvaluationReport);
}

enum EvaluationMode {
    SIMULATION_ONLY = 0;         // 纯仿真评测
    REAL_WORLD_ONLY = 1;         // 纯真实世界评测
    SIM_TO_REAL_COMPARISON = 2;  // 仿真与真实世界对比评测
}

message EvaluationScenario {
    string scenario_id = 1;
    // ... 定义场景的参数，如环境、任务等
}

message RunEvaluationRequest {
    string name = 1;
    EvaluationMode mode = 2;
    arf.v1.PluginIdentifier plugin_to_evaluate = 3; // 要评测的算法或模型插件
    repeated EvaluationScenario scenarios = 4;
}
message RunEvaluationResponse {
    string evaluation_id = 1;
}

message GetEvaluationStatusRequest {
    string evaluation_id = 1;
}
message EvaluationStatus {
    string evaluation_id = 1;
    string status = 2; // "PENDING", "RUNNING", "COMPLETED", "FAILED"
    float progress = 3;
}

message GetEvaluationReportRequest {
    string evaluation_id = 1;
}
message EvaluationReport {
    string evaluation_id = 1;
    string report_url = 2;  // 指向详细报告的链接
    string summary_json = 3; // 关键指标的JSON摘要
}
```
