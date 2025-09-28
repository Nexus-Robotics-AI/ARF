# 🔥 ARF 模块开发参考文档：仿真模块 (Simulation) 



> 🎯 **角色定位:** ARF的"数字孪生中心"与"智能进化熔炉" - 算法的沙盒、模型的熔炉、智能的加速器。
>
> 📦 **模块代号:** `arf-cloud-sim`
>
> ⚡ **所属:** ARF 云端平面 (Cloud Plane)

------



## 📋 1. 核心职责与设计理念





### 🎯 核心使命 (Core Mission)



作为ARF的“数字孪生中心”和“智能进化熔炉”，仿真模块的核心使命是**提供一个大规模、高并发、物理精准、并能与现实世界联动的虚拟环境**。它使得开发者和自动化系统可以在没有物理机器的情况下，安全、高效、大规模地进行算法测试、模型验证、强化学习训练、合成数据生成，并最终实现虚实闭环的智能进化。

主要应用场景包括：

- **🧪 算法回归测试:** 在每次代码提交后，自动在标准仿真场景中运行测试，确保核心算法（如导航、抓取）的行为符合预期。
- **🤖 模型性能评测:** 为`评测模块`提供标准化的测试环境，用于量化评估新训练模型的性能指标（如任务成功率、鲁棒性）。
- **🧠 强化学习 (RL) 训练:** 提供数千个并行的仿真实例，加速强化学习策略的样本采集和训练过程。
- **📷 合成数据生成:** 通过域随机化（Domain Randomization）等技术，在仿真中生成大量带精确标注的合成数据，用于训练感知模型。
- **🌐 实时数字孪生监控:** 创建与物理机器人实时同步的虚拟副本，用于远程沉浸式监控与诊断。
- **🔄 自动化课程学习:** 与训练、评测模块联动，自动发现模型的弱点并生成针对性的训练数据，加速模型迭代。



### 🏗️ 核心架构：“仿真即服务” 与 “数字孪生” (Simulation-as-a-Service & Digital Twin)



我们将仿真能力封装为一个**云原生的、可伸缩的“服务”**。

- **仿真农场 (Simulation Farm):** 我们在云端维护一个由Kubernetes管理的GPU集群，称之为“仿真农场”。
- **按需分配 (On-demand Allocation):** `评测模块`或开发者可以通过API，向“仿真农场”提交一个“仿真任务请求”。
- **任务隔离 (Task Isolation):** “仿真农场”为每一个请求动态地拉起一个或多个独立的、容器化的仿真实例。
- **数字孪生扩展 (Digital Twin Extension):** 在此基础上，架构进一步扩展，支持从真实世界到虚拟世界的数据流，实现高保真的状态同步。



### ⚖️ 设计原则 (Design Principles)



- **🌍 物理逼真度 (Physics Fidelity):** 仿真环境中的物理属性（如重力、摩擦力、碰撞）必须尽可能地接近真实世界。
- **🚀 可伸缩性 (Scalability):** 架构必须支持从单个仿真实例，到数千个并行实例的弹性伸缩。
- **🔌 接口标准化 (Standardized Interface):** 仿真世界中的机器人模型，必须通过与真实世界**完全相同**的ARF接口（`HAL`, `DMS`）来控制和感知，实现“虚实无缝切换”。
- **🔄 可复现性 (Reproducibility):** 任何一次仿真实验都应该是可复现的。给定相同的种子和输入，仿真结果必须完全一致。
- **🔗 虚实闭环 (Sim-to-Real & Real-to-Sim Loop):** 设计必须支持数据从仿真到现实（Sim-to-Real）的迁移，以及从现实到仿真（Real-to-Sim）的反馈，形成一个完整的智能进化闭环。

------



## 📝 2. 核心需求 (Core Requirements)



| ID      | 需求描述             | 验收标准                                                     | 优先级         |
| ------- | -------------------- | ------------------------------------------------------------ | -------------- |
| **S1**  | **高逼真度物理仿真** | 仿真引擎必须支持刚体动力学、精确的碰撞检测和可配置的物理材质。 | **最高**       |
| **S2**  | **可伸缩实例管理**   | 必须能够通过API动态地创建、监控和销毁大量的并行仿真实例。    | **最高**       |
| **S3**  | **无缝虚实切换**     | 仿真机器人对外暴露的gRPC接口和DMS数据主题，必须与真实机器人的`HAL`层100%兼容。 | **最高**       |
| **S4**  | **传感器模拟**       | 必须能够模拟主流的传感器，包括RGB-D相机、激光雷达(LiDAR)和IMU，并能输出符合ARF标准格式的数据。 | **高**         |
| **S5**  | **场景与模型管理**   | 必须提供一个机制，用于管理和版本化不同的仿真场景（Assets）和机器人模型（URDF/USD）。 | **高**         |
| **S6**  | **合成数据生成**     | 支持域随机化（Domain Randomization），能够在仿真中改变纹理、光照、物体位置等，以生成多样化的训练数据。 | **中**         |
| **S7**  | **实时数字孪生**     | 必须支持将物理机器人的实时状态（位姿、关节角）同步到仿真实例中。 | **高 (V1.1+)** |
| **S8**  | **动态场景操控**     | 必须提供API，允许在仿真运行时动态地增加、移除或修改环境中的物体。 | **高 (V1.1+)** |
| **S9**  | **程序化内容生成**   | 必须支持通过算法程序化地、大规模地生成多样化的仿真场景。     | **中 (V1.1+)** |
| **S10** | **自动化评测联动**   | 必须支持由`评测模块`触发的、基于算法反馈（如置信度）的、大规模并行的压力测试工作流。 | **高 (V1.1+)** |

------



## ⚙️ 3. 关键功能与子模块架构





### 🧠 3.1 仿真任务管理器 (Simulation Job Manager)



**农场主角色**：这是一个用`Go`实现的、运行在Kubernetes上的核心服务，是“仿真农场”的大脑。它的职责包括API端点、实例调度、资源编排和生命周期监控。



### 🤖 3.2 仿真实例 (Simulation Instance)



**虚拟世界**：每一个仿真实例都是一个运行在Kubernetes Pod中的**容器**。

- **仿真引擎 (Simulation Engine):** 容器内运行的核心程序，如NVIDIA Isaac Sim或Gazebo。
- **ARF仿真桥 (ARF Sim Bridge):** **这是实现虚实切换的关键**。它是一个“适配器”进程，负责模拟`HAL`接口，并将ARF指令翻译成仿真API调用，反之亦然。



### 🔗 3.3 高级功能扩展 (Advanced Feature Extensions)



- **数字孪生同步服务 (Digital Twin Sync Service):** 一个新的云端组件，负责接收来自边缘端`DMS`的实时状态流，并将其分发到对应的仿真实例中，由`ARF Sim Bridge`应用到虚拟机器人上。
- **动态场景管理器 (Dynamic Scene Manager):** `仿真任务管理器`的一部分，负责处理新的`UpdateScene` RPC，校验请求，并通知相关的`ARF Sim Bridge`执行场景修改。
- **程序化内容生成引擎 (PCG Engine):** 一组可由`数据转换模块`或`仿真任务管理器`调用的容器化算子，用于在启动仿真前按需生成场景文件。

------



## 🔗 4. 接口设计与数据流





### 📥 输入数据流



| **数据源**             | **数据类型**                  | **优先级** | **延迟要求** | **示例场景**                                |
| ---------------------- | ----------------------------- | ---------- | ------------ | ------------------------------------------- |
| **评测模块**           | `EvaluationTask` (gRPC)       | `NORMAL`   | `< 1s`       | 提交一个新模型的自动化评测任务              |
| **开发者(CLI)**        | `SimulationRunRequest` (gRPC) | `NORMAL`   | `< 5s`       | 开发者在本地通过CLI启动一个远程仿真进行调试 |
| **ACR/DIL (被测对象)** | `MotorCommand` (gRPC)         | `HIGH`     | `< 20ms`     | 被测试的算法控制仿真机器人移动              |
| **真实机器人 DMS**     | 实时状态流 (gRPC/MQTT)        | `HIGH`     | `< 100ms`    | 将物理机器人的位姿同步到数字孪生体          |
| **自动化工作流**       | `UpdateSceneRequest` (gRPC)   | `NORMAL`   | `< 1s`       | 评测流水线在运行时动态增加障碍物            |



### 📤 输出数据流



| **目标模块**       | **数据类型**               | **保证** | **性能指标**       |
| ------------------ | -------------------------- | -------- | ------------------ |
| **数据中心**       | 仿真传感器数据, 任务日志   | 可靠上传 | PB级数据存储       |
| **评测模块**       | `EvaluationResult` (gRPC)  | 可靠返回 | 包含量化的性能分数 |
| **DMS (仿真内部)** | `ImageFrame`, `LidarScan`  | 实时流   | 仿真帧率 > 60fps   |
| **远程监控UI**     | 数字孪生状态流 (WebSocket) | 实时     | 延迟 < 200ms       |



### 🧬 核心API草案 (`simulation.proto`)



Protocol Buffers

```protobuf
// protos/arf/cloud/v1/simulation.proto
syntax = "proto3";

package arf.cloud.v1;

// ... import statements ...

// 仿真服务定义
service SimulationService {
  // 提交一个新的仿真任务
  rpc RunSimulation(RunSimulationRequest) returns (RunSimulationResponse);
  // 获取一个仿真任务的状态
  rpc GetSimulationStatus(GetSimulationStatusRequest) returns (GetSimulationStatusResponse);
  // 停止一个正在运行的仿真任务
  rpc StopSimulation(StopSimulationRequest) returns (StopSimulationResponse);
  
  // [V1.1 新增] 更新一个正在运行的仿真场景
  rpc UpdateScene(UpdateSceneRequest) returns (UpdateSceneResponse);
}

message RunSimulationRequest {
    string scene_id = 1;         // e.g., "kitchen_v1" or "pcg:kitchen_simple_v1"
    string robot_model_id = 2;   // e.g., "franka_panda_v3"
    string acr_plugin_id = 3;    // 要在仿真中运行的算法容器插件ID
    // ... 其他参数 ...
}

message UpdateSceneRequest {
    string job_id = 1;
    repeated SceneObject objects_to_add = 2;
    repeated string object_ids_to_remove = 3;
}
message SceneObject {
    string object_id = 1;
    string model_id = 2; // e.g., "apple_v1"
    // ... position, orientation, scale ...
}
message UpdateSceneResponse {
    bool success = 1;
}

// ... 其他消息定义 ...
```

------



## 🛠️ 5. 技术栈与开发环境





### 💻 核心技术栈



| **技术领域**       | **选型**                                        | **版本要求** | **用途说明**                        |
| ------------------ | ----------------------------------------------- | ------------ | ----------------------------------- |
| **仿真引擎**       | **NVIDIA Isaac Sim** / **Gazebo**               | 最新稳定版   | 提供高逼真度的物理和传感器模拟      |
| **编排**           | **Kubernetes**                                  | `1.25+`      | 管理大规模的仿真实例                |
| **核心服务语言**   | **Go**                                          | `1.21+`      | 构建高并发的仿真任务管理器          |
| **仿真桥语言**     | **Python**                                      | `3.10+`      | 与Isaac Sim等仿真器的Python API交互 |
| **机器人模型格式** | **USD (Universal Scene Description)**, **URDF** | -            | 描述机器人和场景                    |

------



## 🔧 6. 开发实施细节





### 🏗️ 6.1 项目结构 V1



```
services/cloud-plane/simulation-manager/ # 仿真任务管理器 (Go)
├── internal/
│   ├── scheduler/                   # Kubernetes Job调度逻辑
│   └── server/                      # gRPC服务器实现
├── Dockerfile
└── ...

templates/
└── simulation-pod/                  # 仿真实例的Pod模板
    ├── Dockerfile                   # 包含Isaac Sim和ARF Sim Bridge
    ├── arf_sim_bridge/              # 仿真桥 (Python)
    │   ├── main.py
    │   ├── hal_servicer.py          # 模拟HAL服务的实现
    │   └── dms_publisher.py         # 向DMS发布模拟数据的实现
    └── ...
```



### 🧪 6.2 测试与验证策略



- **单元测试:** 对`ARF Sim Bridge`中的数据格式转换逻辑进行单元测试。
- **集成测试:**
  - **虚实一致性测试:** 运行同一个`DIL`决策逻辑，分别在真实机器人和仿真机器人上执行同一个任务（如前进1米）。对比两者的最终位置、轨迹和传感器数据，量化其差异。
  - **大规模调度测试:** 编写脚本，在短时间内向`SimulationService`提交100个仿真任务，验证所有任务是否都能被成功调度、执行和清理。

------



## 🚀 7. 开发任务 (Getting Started)





#### **第一阶段：本地单实例运行 (Local Single Instance)**



- **任务1：构建基础仿真容器**
  - **交付物:** 一个包含Isaac Sim（或Gazebo）和ARF依赖的`Dockerfile`，能够在开发者的本地机器上（需有NVIDIA GPU）成功运行。
- **任务2：开发`ARF Sim Bridge`原型**
  - **交付物:** 一个Python程序，能够启动一个简单的仿真场景（如一个房间里有一个机器人），并实现`CameraSensorService`，将仿真摄像头的画面通过gRPC流式传出。
- **任务3：实现本地控制**
  - **交付物:** `ARF Sim Bridge`能够实现`MotorActuatorService`，接收gRPC指令并控制仿真机器人移动。



#### **第二阶段：云端服务化 (Cloud Service Enablement)**



- **任务1：开发`Simulation Job Manager`原型**
  - **交付物:** 一个Go服务，能接收`RunSimulation` gRPC请求，并使用`kubectl`命令行或Kubernetes Go客户端，在云端集群上创建一个运行第一阶段仿真容器的Pod。
- **任务2：实现场景与模型管理**
  - **交付物:** 建立一个对象存储桶(S3/MinIO)，用于存放场景(USD)和机器人(URDF)文件。`Job Manager`能根据请求，将这些文件挂载到仿真Pod中。



#### **第三阶段：与评测模块集成 (Integration with Evaluation)**



- **任务1：实现标准评测接口**
  - **交付物:** `ARF Sim Bridge`能够根据环境变量，在仿真结束后，将任务结果（成功/失败、耗时等）写入一个标准格式的JSON文件。
- **任务-2：与`评测模块`打通**
  - **交付物:** `评测模块`能够成功调用`SimulationService`来运行一个评测任务，并在任务结束后获取到结果JSON文件。



#### **第四阶段：高级功能与可伸缩性 (Advanced Features & Scalability)**



- **任务1：实现合成数据生成**
  - **交付物:** `ARF Sim Bridge`增加域随机化功能，能够在每次启动时，随机改变场景中的光照、纹理和物体位置。
- **任务2：实现并行强化学习支持**
  - **交付物:** `SimulationService`能够一次性创建数千个并行的仿真实例，并提供一个聚合服务，用于高效收集所有实例的训练样本(trajectories)。
- **任务3：压力测试**
  - **交付物:** 提交一份测试报告，展示“仿真农场”在高并发请求下的调度性能和资源利用率。



#### **第五阶段及以后：迈向数字孪生 (Towards Digital Twin)**



- **任务5.1: 实现动态场景API**
  - **交付物:** 在`simulation.proto`和`ARF Sim Bridge`中完整实现`UpdateScene`功能，并提供CLI工具进行调用测试。
- **任务5.2: 开发实时数字孪生原型**
  - **交付物:** 实现从边缘`DMS`到云端`数字孪生同步服务`，再到`仿真实例`的单向状态同步链路，并提供一个简单的Web前端进行可视化验证。
- **任务5.3: 集成程序化内容生成框架**
  - **交付物:** 开发至少一个场景生成器（例如，随机生成不同布局的货架场景），并与`仿真任务管理器`集成，允许通过`scene_id`前缀（如`pcg:`）来触发。
- **任务5.4: 建立置信度驱动的评测流水线**
  - **交付物:** 与`评测模块`团队协作，改造评测流水线，使其能够根据`ACR`算法的反馈，动态触发大规模并行仿真任务，并提交一份展示其有效性的评测报告。
