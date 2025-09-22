# ARF 模块开发参考文档：训练模块 (Training)

所属平面: ARF 云端平面 (Cloud Plane)

模块代号: arf-cloud-training

### 1. 核心职责 (Core Mission)

提供一个可伸缩的、自动化的“AI模型工厂”，让开发者可以高效地利用`数据中心`的数据进行分布式模型训练。

### 2. 关键功能与子模块

- **训练任务调度器:** 接收训练请求，并将其调度到GPU集群。
- **分布式训练引擎:** 支持数据并行、模型并行等多种分布式训练策略。
- **实验跟踪与版本控制:** 与MLflow等工具集成，自动记录每次训练的超参数、性能指标和产出的模型。

### 3. 主要接口与数据流

- **输入:**
  - `训练代码`: 以插件形式提供的训练脚本。
  - `数据集查询语句`: 用于从`数据中心`获取训练数据的查询。
  - `超参数配置`: 本次训练任务的超参数。
- **输出:**
  - `训练好的模型文件`: 版本化地存入`插件市场`。
  - `训练日志与性能指标`: 记录到实验跟踪系统。
- **交互协议:** 通过gRPC服务（`training.proto`）接收和管理训练任务。

### 4. 主选技术栈与开发参考

- **编排:** Kubeflow, Ray on Kubernetes
  - **参考:** [Kubeflow Documentation](https://www.kubeflow.org/docs/), [Ray Documentation](https://docs.ray.io/)
- **框架:** PyTorch, Horovod
  - **参考:** [PyTorch Distributed Overview](https://pytorch.org/tutorials/beginner/dist_overview.html)