# ARF 合作开发仓库结构规范 V1.1

## 1. 核心思想

本文档定义了ARF项目统一的Mono-repo目录结构。其核心设计思想是**“按领域划分，按层级组织”**，确保代码库的清晰性、可维护性和可扩展性。

## 2. 顶层目录结构

```

arf-workspace/
├── .github/              \# CI/CD 与项目模板 (e.g., Pull Request templates)
├── api/                  \# 所有 .proto 接口定义 (The Contract)
├── build/                \# 编译脚本与构建工具 (e.g., common CMake modules)
├── cmd/                  \# 最终可执行程序的入口 (Entrypoints)
├── deployments/          \# 部署配置文件 (e.g., Docker Compose, Kubernetes YAML)
├── docs/                 \# 项目文档 (The Manual)
├── internal/             \# 项目内部私有共享库
├── pkg/                  \# 可被外部引用的公共库 (Public Libraries)
├── third\_party/          \# 第三方依赖库 (e.g., submodules)
├── tools/                \# 开发工具 (e.g., helper scripts, arf-cli)
├── services/             \# 核心后台服务 (The Services)
├── drivers/              \# HAL 硬件驱动 (The Drivers)
├── algorithms/           \# ACR 算法容器 (The Algorithms)
├── sdk/                  \# 开发者SDK (The SDKs)
└── .gitignore
└── README.md

```


## 3. 各目录详解

### `.github/`
* **用途：** 存放与GitHub平台深度集成的配置文件，是项目自动化和协作规范的核心。
* **子目录结构:**
    * `workflows/`: 存放所有GitHub Actions的CI/CD流水线定义（如`ci.yml`, `release.yml`）。
    * `ISSUE_TEMPLATE/`: 存放用于创建Issue的模板（如`bug_report.md`, `feature_request.md`），规范化问题反馈。
    * `pull_request_template.md`: Pull Request的描述模板，引导开发者提供清晰的PR信息。

### `api/`
* **用途：** **这是整个项目的“法律”和“契约”**。存放所有模块间通信的`.proto`接口定义文件。
* **子目录结构:** `api/<plane>/<module>/<version>/`

### `build/`
* **用途：** 存放可复用的构建逻辑和工具。
* **子目录结构:**
    * `cmake/`: 存放共享的CMake模块，供C++项目使用。
    * `package/`: 存放用于打包发布的脚本（如生成`.deb`, `.rpm`或PyPI包的脚本）。
    * `docker/`: 存放可被多个服务复用的基础`Dockerfile`或脚本片段。

### `cmd/`
* **用途：** **这是所有“可执行文件”的统一入口**。为了保持项目整洁，我们将所有服务的`main.go`, `main.cpp`等入口文件统一放在这里。
* **好处：** `services/`和`drivers/`目录下的代码都可以作为纯粹的“库”来编写和测试，不包含`main`函数，增加了代码的可复用性。
* **子目录结构:** `cmd/<service-name>/`

### `deployments/`
* **用途：** **这是将系统变为现实的部署配置**。存放所有与部署相关的文件。
* **子目录结构:** `deployments/<environment>/`
    * `local/`: `docker-compose.yml`等，用于本地一键启动所有服务进行开发和测试。
    * `production/`: Kubernetes YAML, Terraform, Helm Charts等，用于生产环境部署。

### `docs/`
* **用途：** **这是项目的集体智慧沉淀**。存放所有设计文档、开发规范、教程和会议纪要。
* **子目录结构:** `docs/<category>/`

### `internal/`
* **用途：** **项目内部的、私有的共享代码**。这是Go语言的一个约定，但我们将其推广到整个Mono-repo。
* **规则：** 位于`internal/`目录下的代码，**不应该**被项目外部的任何其他代码库所引用。这是一种强制的封装，用于保护我们内部不稳定的API，避免它们被外部依赖。
* **示例：** `internal/database/`（一个所有服务共享的数据库客户端），`internal/auth/`（一个内部使用的认证库）。

### `pkg/`
* **用途：** **项目外部的、公开的共享代码**。与`internal/`相反。
* **规则：** 存放在这里的代码，是我们精心设计和维护的、希望被其他项目（甚至社区）引用的公共库。
* **示例：** `pkg/arf-math/`（一个机器人数学库），`pkg/arf-utils/`（一些通用的工具函数）。

### `third_party/`
* **用途：** 管理那些无法通过标准包管理器（如`go mod`, `pip`）管理的第三方依赖。
* **场景：** 通常用于存放Git submodules，或者需要我们进行少量修改才能使用的开源库（vendoring）。

### `tools/`
* **用途：** **这是提升开发效率的工具集**。存放各种辅助开发的命令行工具和脚本。
* **子目录结构:** `tools/<tool-name>`
    * `arf-cli/`: 我们的核心命令行工具 (Go)。

### `services/`
* **用途：** **这是ARF系统的核心后台服务**。存放边缘平面和云端平面的核心业务逻辑服务。
* **子目录结构:** `services/<plane>/<module-name>`

### `drivers/`
* **用途：** **这是连接物理世界的HAL驱动**。存放所有硬件抽象层的驱动实现。每个驱动都是一个独立的服务。
* **子目录结构:** `drivers/<device-type>/<device-name>`

### `algorithms/`
* **用途：** **这是所有ACR算法容器的家园**。存放所有被封装成标准ACR容器的算法。
* **子目录结构:** `algorithms/<category>/<algorithm-name>`

### `sdk/`
* **用途：** **这是赋能开发者的软件开发工具包**。存放提供给内部和外部开发者编写的SDK。
* **子目录结构:** `sdk/<language>`



## 4. 各模块标准文件树

### api
各目录详解api/这是整个项目的“法律”和“契约”。
用途： 存放所有模块间通信的.proto接口定义文件。子目录结构: api/<plane>/<module>/<version>/api/
└── arf/
 ├── v1/
 │   └── common.proto
 ├── cloud/
 │   └── v1/
 │       ├── data_hub.proto
 │       └── training.proto
 └── edge/
     └── v1/
         ├── dms.proto
         ├── hal.proto
         └── acr.proto


### `services` - 核心后台服务
*存放服务的核心业务逻辑，作为库被`cmd`目录调用。*

#### 示例：`dms-service` (Go)
```

services/edge-plane/dms-service/
├── internal/
│   ├── bus/
│   │   └── redis\_bus.go
│   └── config/
│       └── config.go
├── server/                \# gRPC服务器实现 (但不包含main)
│   ├── handlers.go
│   └── server.go          \# 提供一个 Run() 函数
└── go.mod

```

### `drivers` - HAL硬件驱动
*存放硬件驱动的实现，作为库被`cmd`目录调用。*

#### 示例：`webcam-driver` (Rust)
```

drivers/camera/webcam-driver/
├── src/
│   ├── server.rs            \# gRPC服务实现
│   ├── capture.rs           \# 摄像头捕获逻辑
│   └── lib.rs               \# 库的根文件，提供一个 run() 函数
├── build.rs
└── Cargo.toml

```

### `algorithms` - ACR算法容器
*存放算法的实现，作为库被`cmd`目录调用。*

#### 示例：`object-detector-yolov8` (Python)
```

algorithms/perception/object-detector-yolov8/
├── models/
│   └── yolov8n.pt
├── src/
│   ├── server.py            \# gRPC服务实现 (提供一个 run() 函数)
│   └── infer.py             \# 核心推理脚本
└── requirements.txt

```

### `tools` - 开发工具
*存放CLI等工具的核心逻辑，作为库被`cmd`目录调用。*

#### 示例：`arf-cli` (Go)
```

tools/arf-cli/
├── cmd/                     \# Cobra命令的定义
│   ├── root.go
│   ├── init.go
│   └── run.go
└── internal/
└── clients/
└── acr\_client.go    \# 调用ACR运行时的gRPC客户端

```

### `sdk` - 开发者SDK
*这是一个纯粹的库，没有对应的`cmd`入口。*

#### 示例：`python-sdk`
```

sdk/python/
├── arf\_sdk/
│   ├── **init**.py
│   ├── node.py
│   └── clients/
│       └── dms\_client.py
├── examples/
│   └── talker\_listener.py
└── pyproject.toml

```

---
### **cmd - 统一入口**
**这是将所有库代码“启动”为可执行程序的唯一地方。**

```

cmd/
├── arf-cli/
│   └── main.go              \# arf-cli 的主入口
├── dms-service/
│   └── main.go              \# dms-service 的主入口
└── webcam-driver/
└── main.rs              \# webcam-driver 的主入口

````

#### `cmd` 目录代码示例

**示例：`cmd/dms-service/main.go`**
```go
package main

import (
    // 导入我们自己开发的、位于services目录下的dms-service库
    "arf-workspace/services/edge-plane/dms-service/server"
    "log"
)

func main() {
    // 调用库中定义的Run函数来启动服务
    if err := server.Run(); err != nil {
        log.Fatalf("Failed to run DMS service: %v", err)
    }
}
````

**示例：`cmd/arf-cli/main.go`**

```go
package main

import (
    // 导入我们开发的、位于tools目录下的arf-cli库
    "arf-workspace/tools/arf-cli/cmd"
)

func main() {
    // 调用库中定义的Cobra命令的Execute方法
    cmd.Execute()
}
```

### deployments/这是将系统变为现实的部署配置。用途: 存放所有与部署相关的文件。
子目录结构: 
deployments/<environment>/deployments/
├── local/
│   └── docker-compose.yml  # 本地一键启动配置
└── production/
 ├── kubernetes/
 │   ├── dms-deployment.yaml
 │   └── hal-daemonset.yaml
 └── terraform/

### docs/这是项目的集体智慧沉淀。用途: 存放所有设计文档、开发规范、教程和会议纪要。
子目录结构:
docs/
├── design/         # 架构设计文档
├── standards/      # 开发规范
└── tutorials/      # 教程