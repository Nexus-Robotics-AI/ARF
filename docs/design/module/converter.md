# ARF 模块开发参考文档：数据转换模块 (Converter)

所属平面: ARF 云端平面 (Cloud Plane)

模块代号: arf-cloud-converter

### 1. 核心职责 (Core Mission)

作为数据进入`数据中心`前的“标准化处理车间”，将各种来源、各种格式的异构数据，转换为统一的、可供训练的结构化格式。

### 2. 关键功能与子模块

- **数据格式解析器:** 支持解析ROS Bag, CSV, JSON等多种常见数据格式。
- **数据清洗与增强:** 提供数据去重、异常值处理、数据增强等能力。
- **数据标注工具接口:** 与主流数据标注平台（如Label Studio）集成，实现标注流程自动化。
- **标准化格式写入器:** 将处理后的数据写入为Parquet/TFRecord等标准格式。

### 3. 主要接口与数据流

- **输入:**
  - `原始数据流`: 如ROS Bag文件、CSV文件等。
  - `转换规则配置`: 定义了如何解析和处理特定数据流的YAML或JSON文件。
- **输出:**
  - `标准化的训练数据集`: 写入到`数据中心`。
- **交互协议:** 可以作为独立的微服务，监听特定的存储位置，或通过gRPC服务（`converter.proto`）接收转换任务。

### 4. 主选技术栈与开发参考

- **核心语言:** Python, Go
- **编排:** Kubernetes (K8s Job), Argo Workflows
  - **参考:** [Argo Workflows Documentation](https://argoproj.github.io/argo-workflows/)