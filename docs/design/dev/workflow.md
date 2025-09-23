# ARF Mono-repo 开发工作流指南 V1.0

## 1. 文档目的

本指南为所有ARF开发者提供在`arf-workspace`这个Mono-repo中进行日常开发的标准工作流程。它旨在将我们宏观的架构设计和开发规范，转化为具体、可执行的日常操作步骤。

## 2. 核心开发哲学：“契约先行，由内而外”

无论您接到什么任务，请始终遵循以下开发顺序：

1.  **契约先行 (Interface First):** 如果您的更改涉及到跨模块通信，**永远第一步**都是去`api/`目录下修改或创建`.proto`文件。接口是整个系统的法律。
2.  **由内而外 (Inside-Out):**
    * 先在`internal/`或`pkg/`中实现可复用的**内部共享库**。
    * 然后在`services/`, `drivers/`, `algorithms/`中实现**核心业务逻辑**（作为库）。
    * 最后才是在`cmd/`目录下编写一个极简的**`main`函数**来启动您的服务。

---

## 3. 常见开发任务SOP (Standard Operating Procedures)

### **任务场景一：为现有服务增加一个新功能**

**示例:** 为`dms-service`增加一个`GetTopicStatistics`的RPC方法，用于获取某个主题的统计信息。

* **第1步：修改契约 (`api/`)**

  * 打开 `api/arf/edge/v1/dms.proto` 文件。
  * 在`BusService`中增加新的RPC方法，并定义好请求和响应的消息体。

  ```protobuf
  service BusService {
    // ... 已有的 Publish 和 Subscribe 方法 ...
    
    // 新增方法
    rpc GetTopicStatistics(GetTopicStatisticsRequest) returns (GetTopicStatisticsResponse);
  }
  
  message GetTopicStatisticsRequest {
    string topic = 1;
  }
  
  message TopicStatistics {
    string topic = 1;
    uint64 message_count = 2;
    double message_rate_per_sec = 3;
  }
  
  message GetTopicStatisticsResponse {
    TopicStatistics statistics = 1;
  }
  ```

* **第2步：生成代码**

  * 在仓库根目录运行 `make proto` 或 `buf generate`。这将自动更新所有语言的gRPC客户端和服务端桩代码。

* **第3步：实现核心逻辑 (`services/`)**

  * 打开 `services/edge-plane/dms-service/internal/server/handlers.go`。
  * 为`BusService`服务器实现新的`GetTopicStatistics`方法。

  ```go
  // services/edge-plane/dms-service/internal/server/handlers.go
  func (s *Server) GetTopicStatistics(ctx context.Context, req *dmsv1.GetTopicStatisticsRequest) (*dmsv1.GetTopicStatisticsResponse, error) {
      // ... 实现获取统计信息的核心逻辑 ...
      stats, err := s.bus.GetStats(req.GetTopic())
      if err != nil {
          return nil, status.Errorf(codes.NotFound, "topic not found: %s", req.GetTopic())
      }
      return &dmsv1.GetTopicStatisticsResponse{Statistics: stats}, nil
  }
  ```

* **第4步：编写测试 (`services/`)**

  * 在`dms-service`的测试目录中，为`GetTopicStatistics`方法编写单元测试和接口测试。

* **第5步：(可选) 在客户端调用 (`sdk/` 或 `tools/`)**

  * 如果需要，可以在`arf-cli`或Python SDK中增加调用这个新功能的客户端代码。

* **第6步：提交Pull Request**

  * 您的PR将包含`api/`, `services/`等目录下的文件更改。提交PR并指定审查者。

### **任务场景二：从零创建一个全新的服务**

**示例:** 创建一个`weather-service`，用于从外部API获取天气信息。

* **第1步：定义契约 (`api/`)**
  * 在`api/arf/edge/v1/`目录下，创建新文件`weather.proto`，并定义好`WeatherService`及其方法。

* **第2步：创建服务目录结构 (`services/`)**
  * 在`services/edge-plane/`目录下，创建新文件夹`weather-service`。
  * **最佳实践:** 拷贝`arf-service-template`的内容到这个新文件夹，作为项目的起点。

* **第3步：实现核心逻辑 (`services/`)**
  * 在`weather-service`目录下，实现获取天气的核心逻辑和gRPC服务。

* **第4步：创建程序入口 (`cmd/`)**
  * 在`cmd/`目录下，创建新文件夹`weather-service`。
  * 在`cmd/weather-service/`中，创建`main.go`，其唯一职责就是导入并运行`services/edge-plane/weather-service`中的服务。

* **第5步：添加到本地部署 (`deployments/`)**
  * 打开`deployments/local/docker-compose.yml`文件。
  * 为新的`weather-service`增加一个服务条目，以便在本地开发环境中一键启动。

* **第6步：编写文档和测试**
  * 为`weather-service`编写`README.md`和必要的测试。

* **第7步：提交Pull Request**

### **任务场景三：修复一个算法中的Bug**

**示例:** `object-detector-yolov8`算法的后处理逻辑有误。

* **第1步：定位核心逻辑 (`algorithms/`)**
  * 直接进入`algorithms/perception/object-detector-yolov8/src/`目录。
  * 打开`infer.py`，找到并修复后处理部分的Bug。

* **第2步：完善单元测试 (`algorithms/`)**
  * 打开`algorithms/perception/object-detector-yolov8/tests/`目录。
  * 修改或增加一个单元测试，确保该Bug已被修复且不会再出现。

* **第3步：重新构建容器**
  * 在`algorithms/perception/object-detector-yolov8/`目录下，运行`docker build`来构建一个带有修复的新版本镜像。

* **第4步：提交Pull Request**
  * 您的PR将只包含`algorithms/perception/object-detector-yolov8/`目录下的文件更改。这体现了我们模块化开发的优势。

### **任务场景四：为CLI增加一个新命令**

**示例:** 为`arf-cli`增加一个`arf cluster status`命令。

* **第1רוב：定义命令 (`tools/`)**
  * 进入`tools/arf-cli/cmd/`目录。
  * 创建新文件`cluster_status.go`，使用Cobra库定义新的`clusterStatusCmd`。

* **第2步：实现命令逻辑 (`tools/`)**
  * 在`cluster_status.go`中，编写命令的执行逻辑。这通常涉及到调用其他ARF服务的gRPC客户端（例如，调用`FleetManagementService`）。
  * 客户端的实现应放在`tools/arf-cli/internal/clients/`目录下。

* **第3步：注册命令 (`tools/`)**
  * 在`tools/arf-cli/cmd/root.go`中，将新的`clusterStatusCmd`注册到根命令上。

* **第4步：提交Pull Request**

---

### **总结：工作流快速参考**

| 当您想...                  | **第一步应该打开的目录是...** |
| :------------------------- | :---------------------------- |
| **修改或增加API**          | `api/`                        |
| **实现一个新的后台服务**   | `services/`                   |
| **实现一个新的硬件驱动**   | `drivers/`                    |
| **实现一个新的算法**       | `algorithms/`                 |
| **为算法开发者提供工具**   | `sdk/`                        |
| **为所有开发者提供工具**   | `tools/`                      |
| **让一个服务跑起来**       | `cmd/`                        |
| **在本地集成测试所有服务** | `deployments/`                |