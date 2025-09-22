# ARF 模块开发参考文档：数据中心 (Data Hub)

所属平面: ARF 云端平面 (Cloud Plane)

模块代号: arf-cloud-datahub

### 1. 核心职责 (Core Mission)

作为ARF生态系统的“中央数据湖”，安全、可靠、高效地统一收集、存储、管理和查询所有来自真实世界与仿真环境的数据。

### 2. 关键功能与子模块

- **数据采集接口:** 提供高吞吐量的数据上传端点（gRPC/HTTP）。
- **数据存储层:** 基于对象存储，管理PB级的多模态数据。
- **数据管理与版本控制:** 提供类似Git的数据集版本管理能力（如DVC）。
- **数据查询API:** 提供类SQL的查询接口，方便`训练模块`等下游服务高效检索数据。

### 3. 主要接口与数据流

- **输入:**
  - `真实机器人数据`: 从边缘端`DMS`上传的数据包。
  - `仿真数据`: 从`仿真模块`生成的日志和传感器数据。
  - `第三方数据集`: 外部导入的公开或私有数据集。
- **输出:**
  - `结构化的、可供训练/分析的数据集`: 响应`训练模块`等的数据查询请求。
- **交互协议:** 通过gRPC服务（`data_hub.proto`）对外提供数据上传和查询能力。

### 4. 主选技术栈与开发参考

- **存储后端:** AWS S3, MinIO
  - **参考:** [MinIO Documentation](https://min.io/docs/minio/kubernetes/upstream/index.html)
- **数据格式:** Apache Parquet, Apache Arrow
  - **参考:** [Apache Parquet Format](https://parquet.apache.org/), [Apache Arrow Documentation](https://arrow.apache.org/docs/)