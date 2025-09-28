# 🔥 ARF 模块开发参考文档：评测模块 (Evaluation) V1.1



> 🎯 **角色定位:** ARF的"机器人系统验证与认证中心" - 提供标准化的、可复现的、全面的算法与模型评测。
>
> 📦 **模块代号:** `arf-cloud-evaluation`
>
> ⚡ **所属:** ARF 云端平面 (Cloud Plane)

------



## 📋 1. 核心职责与设计理念





### 🎯 核心使命 (Core Mission)



作为ARF的“机器人系统验证与认证中心”，评测模块的核心使命是**提供一个标准化的、自动化的、可复现的评测流水线，以全面、客观地量化算法、模型乃至整个系统的性能、安全性和可靠性**。它的进化目标是超越简单的功能测试，成为一个能够**量化Sim-to-Real差距**、**验证长尾场景安全性**并**融入人类主观评估**的综合性质量保障平台。

主要应用场景包括：

- **✅ CI/CD持续集成测试:** 在代码提交后，自动触发回归测试，确保新的改动没有破坏核心功能。
- **📊 新模型性能基准测试:** 对`训练模块`产出的新模型，在标准测试集上进行性能评估，并与历史版本进行对比。
- **🌉 Sim-to-Real差距量化:** 在仿真和真实环境中运行相同的测试任务，并生成一份详细的“虚实差距”分析报告。
- **🛡️ 安全性与鲁棒性验证:** 通过对抗性测试，主动在仿真中创造罕见但危险的场景，验证系统的安全反应和鲁棒性。
- **👨‍⚖️ 人在环路的主观评测 (HIL-E):** 对于服务质量、交互自然度等主观指标，引入人类评估员进行打分，使评测更全面。



### 🏗️ 核心架构：“流水线即代码” + “评测即服务” (Pipeline-as-Code + Evaluation-as-a-Service)



我们将复杂的评测流程，抽象为一系列可编排、可配置的云原生流水线。

- **流水线即代码 (Pipeline-as-Code):** 每一个评测流程（如“导航算法评测”）都被定义为一个YAML文件，由一系列可复用的**“评测算子”**（容器化程序）组成一个有向无环图（DAG）。
- **评测即服务 (Evaluation-as-a-Service):** 开发者或自动化系统（如`训练模块`）不直接与底层算子交互，而是通过一个gRPC服务，提交“评测任务请求”。`评测模块`的核心服务负责解析请求，动态生成和执行流水线，并返回最终的评测报告。
- **核心协调者 (Core Orchestrator):** `评测模块`是ARF生态的“首席协调官”。一条评测流水线会按需调用`仿真模块`、`舰队管理`（用于真实机器人测试）、`数据中心`和`插件市场`等多个核心服务。



### ⚖️ 设计原则 (Design Principles)



- **🔄 可复现性 (Reproducibility):** 这是最高原则。任何一次评测，给定相同的被测插件、测试用例和环境，其结果必须完全一致。
- **📊 标准化 (Standardization):** 评测场景、性能指标（KPI）、报告格式都应是标准化的，以保证不同模型/版本之间的公平比较。
- **🤖 自动化 (Automation):** 从任务触发、环境准备、执行测试到报告生成，整个评测流程应尽可能地自动化。
- **📈 全面性 (Comprehensiveness):** 评测维度应覆盖功能、性能、安全性、鲁棒性乃至主观用户体验等多个方面。

------



## 📝 2. 核心需求 (Core Requirements)



| ID     | 需求描述                | 验收标准                                                     | 优先级         |
| ------ | ----------------------- | ------------------------------------------------------------ | -------------- |
| **E1** | **流水线编排与执行**    | 必须支持通过YAML文件定义和执行多步骤的评测流水线，后端基于Argo Workflows或Tekton。 | **最高**       |
| **E2** | **与仿真模块集成**      | 必须能够通过API调用`仿真模块`，在指定的虚拟场景中运行评测任务。 | **最高**       |
| **E3** | **自动化触发**          | 必须支持由`训练模块`成功完成训练任务后，自动触发对新模型的评测。 | **高**         |
| **E4** | **标准化报告生成**      | 必须能够自动生成结构化的评测报告（如JSON, PDF），包含关键性能指标和与历史版本的对比。 | **高**         |
| **E5** | **Sim-to-Real差距分析** | **[V1.1+ 新增]** 必须提供标准流水线，用于在仿真和真实环境中执行相同任务，并量化两者表现差异。 | **高 (V1.1+)** |
| **E6** | **对抗性安全测试**      | **[V1.1+ 新增]** 必须支持在评测中，调用`仿真模块`的动态场景API来引入对抗性干扰，以验证系统安全性。 | **高 (V1.1+)** |
| **E7** | **人在环路评测支持**    | **[V1.1+ 新增]** 必须提供标准算子，能将需要主观评估的任务数据发送到人工评测平台，并在收到评分后继续流水线。 | **中 (V1.1+)** |

------



## ⚙️ 3. 关键功能与子模块架构





### 🧠 3.1 评测任务管理器 (Evaluation Job Manager)



**总调度师角色**：这是一个用`Go`实现的、运行在Kubernetes上的核心服务。

- **API端点:** 对外提供一个gRPC服务（`evaluation.proto`），接收开发者或`训练模块`提交的评测任务。
- **流水线生成器:** 解析用户提交的评测配置，动态地翻译成一个Argo Workflows或Tekton的流水线CRD（Custom Resource Definition）。
- **多模块协调器:** 在流水线执行期间，负责调用`仿真模块`、`舰队管理`等外部服务的API。



### 🏭 3.2 评测算子库 (Evaluation Operator Library)



**工具箱角色**：一系列标准化的、可被流水线编排的容器化工具。

- **环境准备算子:**
  - `SimRunner`: 调用`仿真模块`API，启动一个或多个仿真实例。
  - `RealBotAcquirer`: 调用`舰队管理`API，预约并准备一台用于测试的物理机器人。
- **执行与监控算子:**
  - `TaskExecutor`: 通过`DIL`或`APP`的API，向机器人（虚拟或真实）下发测试任务。
  - `LogMonitor`: 监控`DMS`，收集评测过程中的日志和数据。
- **分析与报告算子:**
  - `MetricCalculator`: 解析日志，计算KPI（如成功率、耗时）。
  - `Sim2RealComparator`: **[V1.1+ 新增]** 对比仿真和真实测试的结果，计算差距指标。
  - `ReportGenerator`: 将所有结果汇总，生成最终报告。
- **高级评测算子:**
  - `AdversarialAgent`: **[V1.1+ 新增]** 在测试期间，调用`仿真模块`API动态添加干扰。
  - `HumanFeedbackCollector`: **[V1.1+ 新增]** 与人工众包平台交互的算子。

------



## 🔗 4. 接口设计与数据流





### 🧬 核心API草案 (`evaluation.proto`) V1.1



Protocol Buffers

```
// protos/arf/cloud/v1/evaluation.proto
syntax = "proto3";

package arf.cloud.v1;

import "arf/v1/common.proto";

// 评测服务
service EvaluationService {
  // 提交一个新的评测任务
  rpc RunEvaluation(RunEvaluationRequest) returns (RunEvaluationResponse);
  // 获取一个评测任务的状态
  rpc GetEvaluationStatus(GetEvaluationStatusRequest) returns (EvaluationStatus);
  // 获取最终的评测报告
  rpc GetEvaluationReport(GetEvaluationReportRequest) returns (EvaluationReport);
}

enum EvaluationMode {
    SIMULATION_ONLY = 0;
    REAL_WORLD_ONLY = 1;
    SIM_TO_REAL_COMPARISON = 2; // [V1.1 新增]
}

message RunEvaluationRequest {
    string display_name = 1;
    arf.v1.PluginIdentifier plugin_to_evaluate = 2; // 要评测的插件（如ACR算法）
    string test_suite_id = 3; // 测试用例集ID, e.g., "navigation_level_1"
    EvaluationMode mode = 4;
    optional string real_robot_group_id = 5; // 如果包含真实世界测试，指定机器人组
}

message RunEvaluationResponse {
    string job_id = 1;
}

// ... 其他请求、状态和报告的消息定义 ...
```

------



## 🛠️ 5. 技术栈与开发环境



| **技术领域**     | **选型**                    | **版本要求** | **用途说明**                     |
| ---------------- | --------------------------- | ------------ | -------------------------------- |
| **核心服务语言** | **Go**                      | `1.21+`      | 构建高并发的任务管理器           |
| **算子开发语言** | **Python**                  | `3.10+`      | 利用丰富的数据分析和科学生态     |
| **工作流引擎**   | **Argo Workflows / Tekton** | 最新稳定版   | 编排和执行评测DAG                |
| **测试框架**     | **Pytest**                  | 最新稳定版   | 用于编写评测算子内部的逻辑和测试 |
| **报告生成**     | **WeasyPrint / Matplotlib** | -            | 从数据生成PDF/HTML报告           |

------



## 🚀 7. 开发路线图 (Development Roadmap)





#### **第一阶段：核心引擎与仿真评测 (Core Engine & Sim-Evaluation)**



- **任务1：搭建工作流引擎**
  - **交付物:** 在Kubernetes集群上成功部署Argo Workflows或Tekton。
- **任务2：开发评测任务管理器原型**
  - **交付物:** 一个Go服务，能接收gRPC请求，并动态生成一个简单的Argo流水线。
- **任务3：实现与`仿真模块`的集成**
  - **交付物:** 开发`SimRunner`和`MetricCalculator`算子，能够完成一个完整的“启动仿真 -> 运行任务 -> 计算成功率”的评测流程。



#### **第二阶段：自动化与报告 (Automation & Reporting)**



- **任务1：实现与`训练模块`的联动**
  - **交付物:** `训练模块`在训练成功后，能够自动调用`EvaluationService`的`RunEvaluation`接口。
- **任务2：开发报告生成器**
  - **交付物:** `ReportGenerator`算子能够根据上游算子的输出，生成一份包含关键指标和图表的HTML评测报告。
- **任务3：与`插件市场`集成**
  - **交付物:** 评测成功后，流水线能将评测报告的链接更新到`插件市场`中对应插件的元数据中。



#### **第三阶段及以后：迈向综合性验证中心 (Towards Comprehensive Validation Center)**



- **任务3.1: 实现Sim-to-Real差距分析流水线**
  - **交付物:** 开发`RealBotAcquirer`和`Sim2RealComparator`算子。提供一个标准流水线，能够对同一个导航任务在仿真和现实中分别进行评测，并输出一份包含轨迹差异、成功率差异的对比报告。
- **任务3.2: 开发对抗性安全测试流水线**
  - **交付物:** 开发`AdversarialAgent`算子，在一个仿真评测流程中，它能在机器人路径上动态生成障碍物，并验证机器人的安全停车或避障行为是否被正确触发。
- **任务3.3: 集成“人在环路”主观评测**
  - **交付物:** 开发`HumanFeedbackCollector`算子，能将一段机器人服务的视频发送到标注平台，并取回人类评分，计入最终报告。
- **任务3.4: 建立标准评测基准 (Benchmark)**
  - **交付物:** 设计并发布一套官方的、涵盖多种核心能力（导航、抓取、交互等）的标准评测集（`TestSuite`），作为ARF生态系统衡量算法水平的“黄金标准”。
