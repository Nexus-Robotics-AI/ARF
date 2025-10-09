# HAL模块架构设计文档 V1.1

**文档作者**: HAL模块开发团队  
**创建日期**: 2025-10-09  
**版本**: V1.1  
**状态**: 设计阶段

---

## 📋 目录

1. [设计原则与思想](#1-设计原则与思想)
2. [系统架构](#2-系统架构)
3. [接口设计](#3-接口设计)
4. [数据流设计](#4-数据流设计)
5. [开发计划](#5-开发计划)
6. [风险与挑战](#6-风险与挑战)

---

## 1. 设计原则与思想

### 1.1 核心设计原则回顾

在开始具体设计前,我首先明确遵守ARF项目的核心原则:

- ✅ **以瞎猜接口为耻,以认真查询为荣** - 所有接口严格遵循`api/edge/v1/hal.proto`
- ✅ **以模糊执行为耻,以寻求确认为荣** - 任何不确定的设计决策都会在文档中明确标注
- ✅ **以臆想业务为耻,以复用现有为荣** - 复用DMS数据总线、RTS调度能力
- ✅ **以创造接口为耻,以主动测试为荣** - 驱动开发必须包含硬件在环测试(HIL)
- ✅ **以破坏架构为耻,以遵循规范为荣** - 严格遵守Mono-repo结构和Rust/C++编码规范

### 1.2 HAL特有设计原则

根据HAL.md文档,额外遵守:

1. **🛡️ 安全第一** - 驱动稳定是物理安全的第一道防线,优先使用Rust
2. **🔌 统一接口** - 同类硬件必须暴露相同的gRPC接口
3. **📦 隔离性** - 单驱动崩溃不能影响系统其他部分
4. **⚙️ 自描述性** - 驱动必须能报告自己支持的参数和能力
5. **🤖 自我完善** - 具备自校准和自诊断能力
6. **⏱️ 时间感知** - 理解RTS任务优先级,上报性能统计

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         上层调用者                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   DIL    │  │   ACR    │  │  Teleop  │  │   API    │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                       │ gRPC调用
        ┌──────────────┴──────────────┐
        │                             │
┌───────▼─────────────────────────────▼──────────────────────────┐
│              HAL Manager Service (Go)                           │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │  Device Manager  │  │  Event Broadcast │                   │
│  │  (udev/DirectShow│  │  (WatchDevices)  │                   │
│  │   监听)           │  │                  │                   │
│  └──────────────────┘  └──────────────────┘                   │
└───────┬──────────────────────────────────────────────────────┘
        │ 设备发现与管理
        │
        ├───────────────────────────────────────────────────┐
        │                                                   │
┌───────▼────────────┐  ┌──────────────────┐  ┌────────────▼──────┐
│  Camera Drivers    │  │  Motor Drivers   │  │  Sensor Drivers   │
│  (Rust)            │  │  (Rust/C++)      │  │  (Rust)           │
├────────────────────┤  ├──────────────────┤  ├───────────────────┤
│ • webcam-v4l2      │  │ • dynamixel      │  │ • force-torque    │
│ • basler-pylon     │  │ • canopen-motor  │  │ • imu-9dof        │
│ • realsense-d435   │  │ • ethercat-servo │  │ • lidar-sick      │
└────────┬───────────┘  └────────┬─────────┘  └────────┬──────────┘
         │                       │                      │
         └───────────────────────┴──────────────────────┘
                                 │
                    每个驱动都实现标准gRPC接口
                    并发布数据到DMS总线
                                 │
                    ┌────────────▼────────────┐
                    │   DMS Data Bus          │
                    │   (Publish/Subscribe)   │
                    └─────────────────────────┘
```

### 2.2 核心组件说明

#### 2.2.1 HAL Manager Service

**职责:**
- 设备发现与枚举(Linux: udev, Windows: DirectShow)
- 驱动注册与管理
- 设备热插拔事件广播
- 驱动健康状态监控

**技术选型: Go**
- 原因: 高并发事件处理、成熟的系统API绑定
- 位置: `services/edge-plane/hal-manager/`
- 入口: `cmd/hal-manager/main.go`

**关键数据结构:**
```go
type DeviceRegistry struct {
    devices map[string]*DeviceInfo  // device_id -> info
    drivers map[string]*DriverInfo  // driver_id -> info
    mu      sync.RWMutex
}

type DeviceInfo struct {
    DeviceID   string
    DeviceType string  // "camera", "motor", "sensor"
    VendorID   string
    ProductID  string
    Status     DeviceStatus
    DriverID   string  // 负责此设备的驱动ID
}
```

#### 2.2.2 Driver Services

**设计原则:**
1. **一类设备一套接口** - 所有相机驱动实现`CameraSensorService`
2. **驱动即库** - 核心逻辑在`drivers/`中作为库实现
3. **容器化部署** - 每个驱动作为独立容器/进程运行

**标准驱动结构(以webcam-driver为例):**

```
drivers/camera/webcam-v4l2-driver/
├── src/
│   ├── lib.rs              # 库根文件,暴露pub fn run()
│   ├── server.rs           # gRPC服务实现
│   ├── capture.rs          # V4L2 API封装
│   ├── parameters.rs       # 参数控制实现
│   └── health.rs           # 健康监控
├── api/                    # .proto软链接
│   └── hal.proto -> ../../../api/edge/v1/hal.proto
├── build.rs                # Protobuf代码生成
├── Cargo.toml
├── Dockerfile
├── configs/
│   └── default.toml
└── README.md
```

**关键实现要点:**

```rust
// src/server.rs - gRPC服务实现
pub struct CameraServer {
    device_id: String,
    capture: Arc<Mutex<V4L2Capture>>,
    dms_client: DmsClient,
    health_monitor: HealthMonitor,
}

impl CameraSensorService for CameraServer {
    async fn stream_frames(
        &self,
        request: Request<StreamFramesRequest>,
    ) -> Result<Response<Self::StreamFramesStream>, Status> {
        // 1. 验证请求
        // 2. 配置捕获参数
        // 3. 启动帧捕获循环
        // 4. 每帧发布到DMS并通过gRPC流返回
    }
    
    async fn list_parameters(
        &self,
        request: Request<ListParametersRequest>,
    ) -> Result<Response<ListParametersResponse>, Status> {
        // 返回此相机支持的所有可调参数
        // 如: brightness, contrast, exposure, gain等
    }
}
```

#### 2.2.3 Calibration Manager (V1.1)

**职责:** 自动化校准任务的编排者

**实现方案:**
- 位置: `services/edge-plane/hal-calibration/`
- 语言: Go (便于编排多个gRPC调用)
- 算法容器: 调用专门的ACR校准算法容器

**手眼标定工作流:**
```
1. 接收StartHandEyeCalibration请求
   ├─ arm_id: "ur5_arm"
   └─ camera_id: "basler_cam_0"

2. 调用MotorActuatorService移动机械臂到预定位姿序列
   └─ 位姿序列存储在configs/calibration_poses.yaml

3. 在每个位姿调用CameraSensorService捕获图像
   └─ 检测标定板并提取角点

4. 收集到足够数据后,调用ACR校准算法容器
   ├─ 算法: algorithms/perception/hand-eye-calibration
   └─ 输入: 位姿+图像数据对

5. 获取计算结果(4x4变换矩阵)并持久化
   └─ 保存到: /etc/arf/robot_config/camera_arm_transform.yaml

6. 返回StartHandEyeCalibrationResponse
```

#### 2.2.4 Health Monitor (嵌入驱动)

**设计思想:** 每个驱动内置轻量级健康监控逻辑

**监控指标:**
- 硬件温度
- 连接状态
- 数据流帧率/延迟
- 错误率统计
- (可选) AI预测的健康分数

**数据流:**
```rust
// 驱动内部定期(如1Hz)发布健康状态到DMS
let health_status = HardwareStatus {
    device_id: self.device_id.clone(),
    status: if self.is_connected() { 
        DeviceStatus::Online 
    } else { 
        DeviceStatus::Offline 
    },
    temperature: self.read_temperature()?,
    metrics: Some(HardwareMetrics {
        frame_rate: self.compute_avg_framerate(),
        error_count: self.error_count.load(Ordering::Relaxed),
    }),
    health_score: self.predict_health(), // 可选:AI预测
};

self.dms_client.publish("/hal/status", &health_status).await?;
```

---

## 3. 接口设计

### 3.1 接口遵守承诺

**🔴 重要声明:** 本节所有接口**严格遵循**`docs/design/interface.md`中定义的`hal.proto`。
**绝不**私自创造新的RPC方法或消息类型。

### 3.2 核心接口清单

根据`interface.md`，HAL模块必须实现以下proto定义的服务:

#### 3.2.1 HAL Manager接口

```protobuf
// 来源: api/edge/v1/hal.proto
service HALManagerService {
  rpc ListAvailableDevices(ListAvailableDevicesRequest) 
      returns (ListAvailableDevicesResponse);
  
  rpc WatchDeviceEvents(WatchDeviceEventsRequest) 
      returns (stream DeviceEvent);
}
```

**实现说明:**
- `ListAvailableDevices`: 扫描系统所有可用硬件设备
- `WatchDeviceEvents`: 长连接流,实时推送设备插拔事件

#### 3.2.2 Camera Sensor接口

```protobuf
service CameraSensorService {
  rpc StreamFrames(StreamFramesRequest) returns (stream ImageFrame);
  rpc ListParameters(ListParametersRequest) returns (ListParametersResponse);
  rpc GetParameters(GetParametersRequest) returns (GetParametersResponse);
  rpc SetParameters(SetParametersRequest) returns (SetParametersResponse);
}
```

**实现说明:**
- 所有相机驱动(webcam, basler, realsense)必须实现此接口
- 参数系统采用自描述设计,支持动态UI生成

#### 3.2.3 Motor Actuator接口

```protobuf
service MotorActuatorService {
  rpc ExecuteCommand(MotorCommand) returns (ExecuteCommandResponse);
}

message MotorCommand {
  string actuator_id = 1;
  oneof command {
    float target_position = 2;
    float target_velocity = 3;
  }
}
```

**实现说明:**
- 执行器驱动的控制入口
- 调用者可以是DIL(自主)或Teleop(遥操作)
- 关键路径必须对接RTS实时调度层

#### 3.2.4 Force/Torque Sensor接口 (V1.1)

```protobuf
service ForceTorqueSensorService {
  rpc StreamWrenches(StreamWrenchesRequest) returns (stream Wrench);
}
```

**实现说明:**
- 为遥操作力反馈提供数据源
- 为在线残差学习算法提供输入

#### 3.2.5 Calibration接口 (V1.1)

```protobuf
service CalibrationService {
  rpc StartHandEyeCalibration(StartHandEyeCalibrationRequest) 
      returns (StartHandEyeCalibrationResponse);
}
```

**实现说明:**
- 由Calibration Manager服务实现
- 编排motor和camera驱动完成校准流程

### 3.3 通用参数控制系统设计

**设计目标:** 让任意相机/传感器的参数可被统一查询和设置

**自描述机制:**

```protobuf
// 参数定义(自描述)
message ParameterDescriptor {
  string name = 1;              // "exposure"
  ParameterType type = 2;       // INTEGER
  ParameterValue min_value = 3;
  ParameterValue max_value = 4;
  repeated ParameterValue allowed_values = 5; // 枚举型参数
  bool read_only = 6;
}

// 参数值(多态)
message ParameterValue {
  oneof value {
    int64 integer = 1;
    double floating = 2;
    string text = 3;
    bool boolean = 4;
  }
}
```

**驱动实现示例:**

```rust
impl CameraSensorService for BaslerPylonDriver {
    async fn list_parameters(&self, ...) -> Result<...> {
        let mut descriptors = vec![];
        
        // 曝光时间参数
        descriptors.push(ParameterDescriptor {
            name: "ExposureTime".into(),
            r#type: ParameterType::Integer as i32,
            min_value: Some(ParameterValue { 
                value: Some(Value::Integer(20)) 
            }),
            max_value: Some(ParameterValue { 
                value: Some(Value::Integer(100000)) 
            }),
            read_only: false,
        });
        
        // 像素格式参数(枚举型)
        descriptors.push(ParameterDescriptor {
            name: "PixelFormat".into(),
            r#type: ParameterType::Text as i32,
            allowed_values: vec![
                ParameterValue { value: Some(Value::Text("Mono8".into())) },
                ParameterValue { value: Some(Value::Text("RGB8".into())) },
            ],
            read_only: false,
        });
        
        Ok(Response::new(ListParametersResponse { parameters: descriptors }))
    }
}
```

**上层应用使用示例:**

```python
# 某个调试工具/WebUI后端
from arf_sdk.hal import CameraSensorClient

camera = CameraSensorClient("basler_cam_0")

# 1. 获取所有可调参数
params = camera.list_parameters()

# 2. 动态生成UI表单(前端框架根据参数类型生成滑块/下拉框等)
for param in params:
    if param.type == INTEGER:
        render_slider(param.name, param.min_value, param.max_value)
    elif param.allowed_values:
        render_dropdown(param.name, param.allowed_values)

# 3. 用户修改后设置参数
camera.set_parameter("ExposureTime", 5000)
camera.set_parameter("PixelFormat", "RGB8")
```

---

## 4. 数据流设计

### 4.1 数据流全景图

```
┌──────────────────────────────────────────────────────────────────┐
│                        数据生产者(HAL驱动)                         │
└───┬──────────────────┬────────────────────┬──────────────────┬───┘
    │                  │                    │                  │
    │ ImageFrame       │ IMUData           │ Wrench           │ HardwareStatus
    │ (30fps)          │ (100Hz)           │ (1kHz)           │ (1Hz)
    │                  │                    │                  │
    ▼                  ▼                    ▼                  ▼
┌───────────────────────────────────────────────────────────────────┐
│                     DMS Data Bus (Redis/ZeroMQ)                   │
│   Topic: /camera/front/image                                      │
│   Topic: /imu/raw                                                 │
│   Topic: /force_torque/wrist/wrench                               │
│   Topic: /hal/status                                              │
└───┬────────────────┬──────────────────────┬──────────────────┬───┘
    │                │                      │                  │
    ▼                ▼                      ▼                  ▼
┌────────┐   ┌────────────┐   ┌────────────────┐   ┌──────────────┐
│  ACR   │   │    DIL     │   │   Teleoperation│   │ DMS Recorder │
│算法容器│   │  决策层    │   │   遥操作模块   │   │  数据采集    │
└────────┘   └────────────┘   └────────────────┘   └──────────────┘
```

### 4.2 关键数据流详解

#### 4.2.1 传感器数据流(Sensor → DMS → Consumers)

**场景:** 相机图像采集与分发

```rust
// drivers/camera/basler-pylon-driver/src/capture.rs

pub async fn capture_loop(
    camera: Arc<PylonCamera>,
    dms_client: Arc<DmsClient>,
    device_id: String,
) -> Result<()> {
    let mut seq_id = 0u64;
    
    loop {
        // 1. 从相机抓取一帧
        let raw_image = camera.grab_frame()
            .await
            .context("Failed to grab frame")?;
        
        // 2. 构造标准ImageFrame消息
        let image_frame = ImageFrame {
            header: Some(Header {
                seq_id,
                stamp: Some(get_current_timestamp()),
                frame_id: format!("{}_optical_frame", device_id),
            }),
            data: raw_image.data,
            encoding: "bgr8".into(),
            height: raw_image.height,
            width: raw_image.width,
        };
        
        // 3. 发布到DMS数据总线
        let topic = format!("/camera/{}/image", device_id);
        dms_client.publish(&topic, &image_frame)
            .await
            .context("Failed to publish to DMS")?;
        
        seq_id += 1;
        
        // 4. 根据配置的帧率进行节奏控制
        tokio::time::sleep(Duration::from_millis(33)).await; // ~30fps
    }
}
```

**数据流特点:**
- **高频率:** 相机30fps，IMU 100Hz
- **低延迟要求:** < 5ms端到端延迟
- **DMS性能要求:** 必须支持零拷贝或共享内存传输

#### 4.2.2 控制指令流(DIL/Teleop → HAL → Hardware)

**场景1: 自主模式电机控制**

```
DIL决策层
  │
  │ MotorCommand {actuator_id: "arm_joint_1", target_position: 1.57}
  ▼
MotorActuatorService (gRPC)
  │
  │ 转换为硬件协议(CAN/EtherCAT)
  ▼
物理电机硬件
```

**关键设计点:**
```rust
// drivers/motor/dynamixel-driver/src/server.rs

impl MotorActuatorService for DynamixelServer {
    async fn execute_command(
        &self,
        request: Request<MotorCommand>,
    ) -> Result<Response<ExecuteCommandResponse>, Status> {
        let cmd = request.into_inner();
        
        // 1. 验证执行器ID
        let motor = self.motors.get(&cmd.actuator_id)
            .ok_or_else(|| Status::not_found("Motor not found"))?;
        
        // 2. 根据命令类型执行
        match cmd.command {
            Some(Command::TargetPosition(pos)) => {
                // 🚨 关键路径:可能需要与RTS集成以保证实时性
                motor.set_goal_position(pos).await?;
            }
            Some(Command::TargetVelocity(vel)) => {
                motor.set_goal_velocity(vel).await?;
            }
            None => return Err(Status::invalid_argument("No command specified")),
        }
        
        // 3. 发布电机状态到DMS(用于监控和记录)
        self.publish_motor_state(&cmd.actuator_id).await?;
        
        Ok(Response::new(ExecuteCommandResponse {
            success: true,
            message: "Command executed".into(),
        }))
    }
}
```

**场景2: 遥操作模式高优先级控制**

```
遥操作模块 (CRITICAL优先级)
  │
  │ TeleopCommand.arm_commands
  ▼
MotorActuatorService
  │
  │ 通过RTS高优先级通道(无锁队列)
  ▼
实时控制线程(1kHz) ← RTS调度
  │
  ▼
硬件
```

**与RTS集成示例:**

```rust
// drivers/motor/ethercat-servo/src/realtime.rs

use arf_rts::{RtTask, RtScheduler, Priority};

pub fn setup_realtime_control(
    motor_id: String,
    command_queue: Arc<SpscQueue<MotorCommand>>, // 无锁队列
) -> Result<()> {
    let rt_task = RtTask::new(
        format!("motor_control_{}", motor_id),
        Priority::HIGH, // 对应RTS的HIGH优先级
        Duration::from_millis(1), // 1kHz周期
        move || {
            // 这是在RTS调度的实时线程中执行
            if let Some(cmd) = command_queue.try_pop() {
                // 从队列取出最新指令并执行
                execute_motor_command_realtime(&cmd);
            }
        },
    );
    
    RtScheduler::global().submit_task(rt_task)?;
    Ok(())
}
```

#### 4.2.3 健康状态流(HAL → DMS → Fleet/Monitoring)

```rust
// 每个驱动内部的健康监控协程

async fn health_monitor_loop(
    device_id: String,
    dms_client: Arc<DmsClient>,
    metrics_collector: Arc<MetricsCollector>,
) {
    let mut interval = tokio::time::interval(Duration::from_secs(1));
    
    loop {
        interval.tick().await;
        
        let status = HardwareStatus {
            device_id: device_id.clone(),
            status: DeviceStatus::Online as i32,
            temperature: metrics_collector.get_temperature(),
            metrics: Some(HardwareMetrics {
                frame_rate: metrics_collector.get_avg_framerate(),
                error_count: metrics_collector.get_error_count(),
            }),
            health_score: metrics_collector.predict_health(),
        };
        
        let _ = dms_client.publish("/hal/status", &status).await;
    }
}
```

---

## 5. 开发计划

### 5.1 阶段划分(遵循HAL.md第7节)

#### **阶段一: 核心框架与虚拟驱动** (2周)

**目标:** 搭建HAL基础设施，提供虚拟设备供上层并行开发

**任务清单:**

- [ ] **Task 1.1: 实现HAL Manager Service原型**
  - 位置: `services/edge-plane/hal-manager/`
  - 功能:
    - [x] 实现`ListAvailableDevices` RPC
    - [x] 实现设备发现(Linux udev监听)
    - [x] 实现`WatchDeviceEvents`流式RPC
  - 测试: 插拔USB摄像头能收到事件
  - 交付物: `cmd/hal-manager/main.go`可运行

- [ ] **Task 1.2: 开发虚拟摄像头驱动**
  - 位置: `drivers/camera/virtual-camera-driver/`
  - 功能:
    - [x] 实现`CameraSensorService`全部RPC
    - [x] 以10Hz频率发布模拟的`ImageFrame`(彩色噪声图)
    - [x] 实现通用参数系统(暴露brightness/contrast虚拟参数)
    - [x] 以1Hz频率发布`HardwareStatus`到DMS
  - 测试: ACR/DIL能订阅到图像数据
  - 交付物: 可独立运行的Rust服务

- [ ] **Task 1.3: 开发虚拟电机驱动**
  - 位置: `drivers/motor/virtual-motor-driver/`
  - 功能:
    - [x] 实现`MotorActuatorService`
    - [x] 收到指令后仅打印日志(模拟执行)
    - [x] 发布虚拟的电机状态到DMS
  - 测试: DIL/Teleop能下发指令且收到响应
  - 交付物: 可独立运行的Rust服务

**阶段验收标准:**
- HAL Manager能发现虚拟设备
- 虚拟驱动能正常响应gRPC调用
- DMS能收到虚拟传感器数据
- ACR/DIL可以基于虚拟设备开始并行开发

#### **阶段二: 真实硬件集成** (3周)

**目标:** 连接第一批真实硬件，验证架构可行性

**任务清单:**

- [ ] **Task 2.1: 集成USB摄像头(V4L2)**
  - 位置: `drivers/camera/webcam-v4l2-driver/`
  - 功能:
    - [x] 使用`v4l`库访问Linux摄像头
    - [x] 实现参数控制(brightness, contrast, saturation等)
    - [x] 支持MJPEG/YUYV格式自动转换
  - 测试: 连接罗技C920摄像头,能以30fps稳定输出
  - 交付物: 包含Dockerfile的完整驱动项目

- [ ] **Task 2.2: 集成Basler工业相机**
  - 位置: `drivers/camera/basler-pylon-driver/`
  - 挑战: 复杂SDK集成、Dockerfile封装Pylon安装
  - 功能:
    - [x] 封装Pylon C++ API
    - [x] 支持硬件触发模式
    - [x] 实现高级参数(曝光、增益、白平衡等)
  - 测试: 连接Basler acA1300,能稳定60fps
  - 交付物: 包含完整SDK依赖的Docker镜像

- [ ] **Task 2.3: 集成第一个电机(Dynamixel)**
  - 位置: `drivers/motor/dynamixel-driver/`
  - 功能:
    - [x] 使用`dynamixel_sdk` crate
    - [x] 实现位置/速度控制模式切换
    - [x] 读取电机温度、电流等状态
  - 测试: 控制MX-28电机移动到指定位置
  - 交付物: 支持USB/RS485通信的驱动

- [ ] **Task 2.4: 实现设备热插拔处理**
  - 位置: `services/edge-plane/hal-manager/`
  - 功能:
    - [x] 设备连接时自动启动对应驱动
    - [x] 设备断开时优雅停止驱动
    - [x] 通过`WatchDeviceEvents`实时通知上层
  - 测试: 拔插USB设备时系统不崩溃
  - 交付物: 增强的HAL Manager

**阶段验收标准:**
- 至少一款真实相机和电机可用
- 驱动能在Docker中稳定运行
- 热插拔功能正常
- 性能指标达标(相机30fps+, 延迟<10ms)

#### **阶段三: 高级功能与V1.1特性** (4周)

**目标:** 实现自校准、健康监控等智能化特性

**任务清单:**

- [ ] **Task 3.1: 实现Calibration Manager**
  - 位置: `services/edge-plane/hal-calibration/`
  - 功能:
    - [x] 实现手眼标定流程编排
    - [x] 集成OpenCV标定算法(或调用ACR容器)
    - [x] 支持标定结果持久化
  - 测试: 完成一次完整的手眼标定
  - 交付物: `CalibrationService`的Go实现

- [ ] **Task 3.2: 集成力/扭矩传感器**
  - 位置: `drivers/sensor/force-torque-driver/`
  - 功能:
    - [x] 实现`ForceTorqueSensorService`
    - [x] 以1kHz频率发布`Wrench`数据
  - 应用: 为遥操作力反馈提供数据源
  - 交付物: ATI Axia传感器驱动

- [ ] **Task 3.3: 与RTS集成示例**
  - 位置: `drivers/motor/ethercat-servo-driver/`
  - 功能:
    - [x] 演示如何将电机控制回路放入RTS调度
    - [x] 使用无锁队列传递指令
    - [x] 监控RTS提供的性能统计
  - 测试: 1kHz控制回路抖动<100μs
  - 交付物: 与RTS深度集成的伺服驱动

- [ ] **Task 3.4: 健康监控与预测**
  - 位置: 嵌入各个驱动内部
  - 功能:
    - [x] 收集底层遥测数据(温度、电流波形)
    - [x] (可选)集成ONNX Runtime运行健康预测模型
    - [x] 异常时触发告警
  - 测试: 模拟设备过热场景
  - 交付物: 增强的健康监控逻辑

**阶段验收标准:**
- 手眼标定功能可用
- 力反馈数据可用于遥操作
- 至少一个驱动完成RTS集成
- 健康监控能预警潜在故障

### 5.2 人力资源需求

- **核心开发:** 2名Rust工程师 + 1名Go工程师
- **测试工程师:** 1名(负责HIL测试)
- **硬件工程师:** 1名(协助硬件集成调试)

### 5.3 外部依赖

- **硬件设备:** 
  - USB摄像头(罗技C920或类似)
  - Basler工业相机(acA1300-30gm)
  - Dynamixel电机(MX-28)
  - ATI Axia力/扭矩传感器(V1.1)
  
- **软件依赖:**
  - Rust 1.65+
  - Go 1.21+
  - Protobuf编译器
  - (可选) PREEMPT_RT内核(用于RTS集成测试)

---

## 6. 风险与挑战

### 6.1 技术风险

#### **Risk 1: 复杂SDK集成难度**

**描述:** Basler Pylon等厂商SDK依赖复杂，Dockerfile封装困难

**应对策略:**
1. 提前搭建SDK测试环境
2. 参考官方Docker示例
3. 必要时寻求厂商技术支持
4. 准备Plan B(先用USB摄像头验证架构)

#### **Risk 2: RTS集成的实时性保证**

**描述:** Rust驱动如何与C++ RTS库安全集成

**应对策略:**
1. 通过C FFI封装RTS核心API
2. 使用`cc` crate构建C++代码
3. 严格遵守RTS的零动态分配原则
4. 优先实现非实时路径，RTS集成作为优化

#### **Risk 3: DMS性能瓶颈**

**描述:** 高频传感器数据(IMU 100Hz, 力传感器1kHz)可能压垮DMS总线

**应对策略:**
1. 为DMS选择高性能后端(ZeroMQ > Redis)
2. 关键数据流使用共享内存(后续优化)
3. 对非关键数据进行降采样
4. 监控DMS延迟，及时发现瓶颈

### 6.2 项目风险

#### **Risk 4: 硬件设备采购延迟**

**应对策略:**
- 阶段一使用虚拟驱动，不依赖真实硬件
- 提前启动设备采购流程
- 准备多个备选型号

#### **Risk 5: 跨模块协作依赖**

**描述:** HAL依赖DMS、RTS等其他模块的完成

**应对策略:**
- 建立每周跨模块sync会议
- 使用Mock服务进行独立开发
- 明确各模块的接口稳定时间点

### 6.3 需要确认的问题

**🔴 需要与架构师/PM确认的问题:**

1. **DMS后端选择:** Redis还是ZeroMQ? (影响性能上限)
2. **RTS优先级:** 哪些电机驱动必须在阶段二集成RTS?
3. **校准算法:** 手眼标定是用ACR容器还是内置OpenCV?
4. **设备采购:** Basler相机和ATI传感器的采购进度?
5. **测试环境:** 是否有专门的HIL测试台?

---

## 附录

### A. 参考文档清单

✅ 已认真阅读并遵守以下文档:

1. `docs/design/struct.md` - Mono-repo结构规范
2. `docs/design/module/HAL.md` - HAL模块详细设计
3. `docs/design/hardware_adapt.md` - 跨平台适配策略
4. `docs/design/interface.md` - 核心接口定义
5. `docs/design/dev/workflow.md` - 开发工作流
6. `docs/design/dev/standard.md` - 统一开发规范
7. `docs/design/dev/Rust.md` - Rust编码规范
8. `docs/design/dev/C++.md` - C++编码规范
9. `docs/design/module/DMS.md` - DMS模块(数据总线)
10. `docs/design/module/RTS.md` - RTS模块(实时调度)
11. `docs/design/module/ACR.md` - ACR模块(算法容器)

### B. 代码仓库位置

```
arf-workspace/
├── api/edge/v1/hal.proto          # HAL接口定义
├── services/edge-plane/
│   ├── hal-manager/               # HAL Manager服务
│   └── hal-calibration/           # 校准管理服务(V1.1)
├── drivers/
│   ├── camera/                    # 相机驱动集合
│   │   ├── virtual-camera-driver/
│   │   ├── webcam-v4l2-driver/
│   │   └── basler-pylon-driver/
│   ├── motor/                     # 电机驱动集合
│   │   ├── virtual-motor-driver/
│   │   ├── dynamixel-driver/
│   │   └── ethercat-servo-driver/
│   └── sensor/                    # 传感器驱动集合
│       └── force-torque-driver/
└── cmd/
    ├── hal-manager/               # HAL Manager入口
    ├── virtual-camera-driver/     # 虚拟相机入口
    └── ...                        # 其他驱动入口
```

### C. 关键决策记录

| 决策ID | 决策内容 | 理由 | 日期 |
|--------|---------|------|------|
| HAL-D001 | HAL Manager使用Go实现 | Go的并发模型适合事件处理 | 2025-10-09 |
| HAL-D002 | 驱动优先使用Rust | 内存安全是硬件层的首要原则 | 2025-10-09 |
| HAL-D003 | 采用自描述参数系统 | 支持动态UI生成,提升易用性 | 2025-10-09 |
| HAL-D004 | 虚拟驱动先行策略 | 解除上层模块的硬件依赖,加速并行开发 | 2025-10-09 |

---

**文档状态:** ✅ 已完成初稿  
**下一步:** 提交评审,等待架构师和其他模块负责人反馈

---

**📝 作者承诺:**

作为HAL模块的开发者,我承诺:
- ✅ 严格遵守所有ARF项目规范
- ✅ 不私自创造接口,一切以`hal.proto`为准
- ✅ 任何不确定的设计都会寻求确认
- ✅ 优先编写测试,保证代码质量
- ✅ 主动与DMS、RTS等模块负责人沟通协作

**签名:** HAL开发团队  
**日期:** 2025-10-09
