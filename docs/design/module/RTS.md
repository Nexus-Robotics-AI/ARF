# 🔥 实时调度层 (RTS)



> 🎯 角色定位: ARF的"脊髓" - 确保关键任务的确定性响应。
>
> 📦 模块名: arf-edge-rts
>
> ⚡ 所属: ARF 边缘平台 (Edge Plane)

------



## 📋 1. 核心职责与设计理念





### 🎯 核心使命 (Core Mission)



作为ARF的"脊髓"，RTS为那些对时间有严苛要求的**关键任务**提供一个**确定性的、可预测的执行保障**。它的核心使命是确保任务能够在规定的时间窗（deadline）内被准时执行，不受非实时任务（如日志记录、网络通信）的干扰。

主要应用场景包括：

- **🎮 电机控制回路** - 确保机械臂、轮式底盘的精确控制。
- **🚨 安全保护系统** - 保证紧急停止、碰撞检测的实时响应。
- **🎯 遥操作指令** - 为遥操作提供低延迟的指令通道。
- **⏱️ 时间关键任务** - 传感器采样、控制算法执行的时序保证。



### 🏗️ 核心架构：双世界模型 (Two-World Model)



我们的RTS设计基于一个经典的实时系统模型，将整个系统划分为两个世界：

- **实时世界 (RT World):** 由少数几个高优先级的、运行在隔离CPU核心上的实时线程组成。这些线程执行的是对时间极其敏感的控制回路。
- **非实时世界 (NRT World):** 包含系统中的绝大多数线程，如gRPC服务线程、日志线程、业务逻辑线程等。它们运行在普通的CPU核心上。

**RTS的职责，就是这两个世界之间的“守护者”和“信使”**。



### ⚖️ 设计原则 (Design Principles)



- **⚡ 确定性优先:** 宁可牺牲吞吐量，也要保证延迟的可预测性。
- **🔒 零动态分配:** 运行时严禁任何可能导致不可预测延迟的操作，特别是动态内存分配（`new`, `malloc`），应使用预分配的内存池。
- **🎚️ 优先级分层:** 支持多级优先级调度，确保关键任务优先执行。
- **🔄 故障恢复:** 实现任务超时检测和系统降级机制。

------



## 📝 2. 核心需求 (Core Requirements)



| ID     | 需求描述                                         | 验收标准                                                     | 优先级   |
| ------ | ------------------------------------------------ | ------------------------------------------------------------ | -------- |
| **R1** | **确定性延迟 (Deterministic Latency)**           | RTS调度的任务，其执行延迟（Jitter）必须控制在微秒级（µs）。  | **最高** |
| **R2** | **高优先级抢占 (Preemption)**                    | 实时任务必须能够抢占任何非实时的、低优先级的系统任务，包括内核任务。 | **最高** |
| **R3** | **高频周期性任务 (High-Frequency Tasks)**        | 必须支持以极高频率（如1kHz或更高）稳定、周期性地执行任务，用于伺服控制等场景。 | **最高** |
| **R4** | **RT/NRT隔离与通信 (Isolation & Communication)** | 必须提供一个**实时安全**的通信机制，允许非实时（NRT）线程向实时（RT）线程传递数据，而不会破坏实时性。 | **高**   |
| **R5** | **CPU核心亲和性 (CPU Affinity)**                 | 必须支持将实时任务绑定到特定的、隔离的CPU核心上运行，以避免被其他进程干扰。 | **高**   |
| **R6** | **避免动态内存分配 (No Dynamic Memory)**         | 在实时任务的执行循环中，**严禁**任何可能导致不可预测延迟的操作，特别是动态内存分配（`new`, `malloc`）。 | **最高** |

------



## ⚙️ 3. 关键功能与子模块架构





### 🧠 3.1 实时任务调度器 (Real-time Task Scheduler)



**守护者角色**：调度器通过`PREEMPT_RT`内核补丁实现强大的抢占式调度。

- **内核准备:** 操作系统镜像必须基于一个打上了`PREEMPT_RT`补丁的Linux内核进行构建。
- **CPU隔离:** 通过内核启动参数（`isolcpus`）将一个或多个CPU核心从通用调度器中隔离出来，专门留给实时任务使用。
- **线程管理:** 预创建专用的实时pthreads线程，避免运行时创建开销。通过`pthread_setschedparam`将其调度策略设置为`SCHED_FIFO`（先进先出），并赋予其极高的实时优先级。同时，使用`pthread_setaffinity_np`将其绑定到隔离的CPU核心上。
- **优先级映射:**
  - `CRITICAL`: RT Priority 99 - 紧急停止、安全系统。
  - `HIGH`: RT Priority 80 - 电机控制、传感器采样。
  - `NORMAL`: RT Priority 50 - 一般实时任务。



### 📡 3.2 高优先级通信通道 (High-Priority Communication)



**信使角色**：这是实现需求 **R4** 的关键，用于解决RT/NRT世界的通信难题。

- **设计方案：** 使用“**无锁单生产者单消费者队列**” (Lock-free SPSC Queue)。通过**共享内存** + **原子操作**实现零拷贝、无锁消息传递。

- **数据流:**

  1. `HAL`模块的gRPC服务线程（NRT世界）收到请求后，将指令（如`MotorCommand`）`push`到一个无锁队列中。此`push`操作是实时安全的，不会阻塞。
  2. RTS调度的一个高频实时任务（RT世界），在其`on_execute`循环中，从队列中`pop`最新的指令。
  3. RT任务根据指令，通过总线或直接内存映射，向硬件发送控制信号。

- **内存布局优化示例:**

  C++

  ```
  struct RTMessageQueue {
      std::atomic<uint32_t> write_index;
      std::atomic<uint32_t> read_index;
      RTMessage messages[QUEUE_SIZE];  // 预分配消息槽
  };
  ```



### ⏰ 3.3 高精度时间同步 (Precision Time Sync)



- **时钟源:** 使用`CLOCK_MONOTOTIC_RAW`，这是一个单调时钟，不受NTP等网络时间协议的影响。
- **精度目标:** 时钟同步误差小于 `1ms`。
- **硬件集成:** 通过HAL的gRPC接口读取硬件时间戳，实现与硬件时钟的同步。

------



## 🔗 4. 接口设计与数据流



`RTS`**不是一个gRPC服务**，而是一个**核心库**。它将被其他需要实时能力的底层模块（主要是`HAL`）直接链接和调用。因此，它的“接口”就是其C++头文件中定义的API。



### 📥 输入数据流



| **数据源**                 | **数据类型**    | **优先级** | **延迟要求** | **示例场景**     |
| -------------------------- | --------------- | ---------- | ------------ | ---------------- |
| **HAL模块 HAL 模块**       | `MotorCommand`  | `CRITICAL` | `< 1ms`      | 伺服电机位置控制 |
| **DIL决策层**              | `SafetyCommand` | `CRITICAL` | `< 2ms`      | 紧急停止指令     |
| **Teleoperation 远程操作** | `TeleopCommand` | `HIGH`     | `< 5ms`      | 遥操作手柄输入   |
| **ACR算法 ACR 算法**       | `ControlSignal` | `HIGH`     | `< 10ms`     | 视觉伺服控制信号 |



### 📤 输出数据流



| **目标模块**                  | **数据类型**   | **保证**   | **性能指标**   |
| ----------------------------- | -------------- | ---------- | -------------- |
| **HAL执行器**                 | `TimedCommand` | 确定性执行 | 抖动 `< 100μs` |
| **DMS数据总线**               | `RTMetrics`    | 实时监控   | 延迟统计上报   |
| **Fleet Management 车队管理** | `HealthStatus` | 状态上报   | 系统健康度指标 |



### 🧬 核心API草案 (`rts/scheduler.hpp`)



C++

```C++
#pragma once

#include <functional>
#include <memory>
#include <chrono>

namespace arf {
namespace rts {

// 实时任务的基类，开发者需要继承它并实现 on_execute
class RealTimeTask {
public:
    virtual ~RealTimeTask() = default;
    // 任务的执行逻辑
    virtual void on_execute() = 0;
};

class TaskScheduler {
public:
    // 调度一个周期性任务
    // - task: 要执行的任务实例
    // - period: 任务的执行周期
    // - priority: 任务的实时优先级 (1-99)
    // - cpu_core: 要绑定的CPU核心ID
    // 返回一个任务句柄，或在失败时返回nullptr
    static std::unique_ptr<TaskHandle> schedule_periodic_task(
        std::shared_ptr<RealTimeTask> task,
        std::chrono::nanoseconds period,
        int priority,
        int cpu_core
    );

    // ... 其他调度方法，如调度一次性任务 ...
};

// 任务句柄，用于管理任务的生命周期
class TaskHandle {
public:
    // 取消并销毁任务
    void cancel();
};

} // namespace rts
} // namespace arf
```

------



## 🛠️ 5. 技术栈与开发环境





### 💻 核心技术栈



| **技术领域** | **选型**        | **版本要求**             | **用途说明**                |
| ------------ | --------------- | ------------------------ | --------------------------- |
| **编程语言** | **C++17/20**    | `GCC 9.0+` / `Clang 10+` | 现代C++特性，智能指针、RAII |
| **实时内核** | **PREEMPT_RT**  | `Linux 5.10+`            | 硬实时调度支持              |
| **构建系统** | **CMake**       | `3.15+`                  | 支持`find_package`模块化    |
| **通信框架** | **gRPC**        | `1.40+`                  | 与其他模块的服务接口        |
| **测试框架** | **Google Test** | `1.11+`                  | 单元测试和基准测试          |



### ⚙️ 系统配置要求



Bash

```bash
# 实时内核配置检查
$ uname -r | grep rt
5.10.78-rt55

# CPU隔离配置 (isolcpus)
$ cat /proc/cmdline
... isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3

# 实时调度权限
$ ulimit -r
99
```

------



## 🔧 6. 开发实施细节





### 🏗️ 6.1 项目结构 V1



```
arf-edge-rts/
├── include/arf/rts/
│   ├── scheduler.h          # 实时调度器接口
│   ├── message_bus.h        # 高优先级消息总线
│   └── time_sync.h          # 时间同步服务
├── src/
│   ├── scheduler/
│   ├── communication/
│   └── timing/
├── tests/
│   ├── unit/                # 单元测试
│   └── benchmark/           # 性能基准测试
├── examples/
│   └── motor_control/       # 电机控制示例
└── docs/
```



### 🧪 6.2 测试与验证策略



- **单元测试:** 使用 Google Test 编写单元测试。
- **基准测试:** 编写一个C++测试程序，调度一个1kHz的周期性任务，并测量打印其执行周期的**抖动（Jitter）**。这将是衡量RTS模块性能的黄金标准。
- **集成测试:**
  - **电机控制回路测试:** 1000Hz控制频率下的延迟抖动测试。
  - **系统负载测试:** 高CPU负载下的实时性能保证测试。
  - **故障恢复测试:** 任务超时和系统降级机制验证。

------



## 🚀 7. 开发任务 (Getting Started) V1

#### 第一阶段

- **任务1：设计并实现RTS核心API**
  - **语言:** C++。
  - **交付物:** 完整的`scheduler.hpp`和`task.hpp`头文件，定义好`TaskScheduler`, `RealTimeTask`等核心类和方法。
- **任务2：实现一个“软实时”调度器**
  - **功能:** 实现`TaskScheduler`的**非`PREEMPT_RT`版本**。它仍然使用pthreads，但使用标准的`SCHED_OTHER`调度策略。
  - **目的:** 使得我们可以在**任何标准的Linux开发机**上进行RTS库的开发和测试，而无需立即准备一个完整的实时内核环境。
- **任务3：编写基准测试 (Benchmark)**
  - **交付物:** 一个C++测试程序，调度一个1kHz的周期性任务。
  - **验收标准:** 该程序需要测量并打印出任务执行周期的**抖动（Jitter）**。在软实时版本下，抖动可能在毫秒级；在硬实时内核上，它应该降低到微秒级。

#### **第二阶段：硬实时核心能力实现 (Hard Real-Time Enablement)**

**核心目标：** 将第一阶段的“软实时”原型迁移到真实的`PREEMPT_RT`内核环境，并实现微秒级的确定性调度，完成RTS的核心承诺。

- **任务1：环境与内核部署**
  - **交付物：** 构建并部署一个带有`PREEMPT_RT`实时补丁的Linux操作系统镜像。
  - **验收标准：** 目标硬件启动后，通过`uname -r`可验证内核为`-rt`版本。系统日志中无实时相关的严重错误。
- **任务2：实现硬实时调度器后端**
  - **交付物：** 在`TaskScheduler`中实现使用`SCHED_FIFO`调度策略和高优先级的代码路径。
  - **验收标准：** 调度器能够创建具有`RT Priority 99`的线程，并成功将其绑定到由`isolcpus`指定的隔离CPU核心上。
- **任务3：实现高优先级通信通道**
  - **交付物：** 完成“无锁单生产者单消费者队列”（Lock-free SPSC Queue）的C++实现，用于RT/NRT线程间的安全通信。
  - **验收标准：** NRT线程可以无阻塞地向队列中推送10,000条消息，同时RT线程可以无错误地消费所有消息。
- **任务4：硬实时基准测试**
  - **交付物：** 在`PREEMPT_RT`环境上运行第一阶段编写的1kHz基准测试程序。
  - **验收标准：** 任务调度的**抖动（Jitter）\**必须从毫秒级降低到\**100微秒（μs）以内**，符合R1需求。

#### **第三阶段：系统集成与监控 (Integration & Monitoring)**

**核心目标：** 将RTS模块与ARF框架中的其他关键模块（如HAL）深度集成，并提供必要的监控和诊断工具，使其在系统中真正可用。

- **任务1：与HAL模块集成**
  - **交付物：** 提供一个稳定的集成层，允许HAL模块（如电机控制器）将其高频控制回路注册为RTS中的`RealTimeTask`。
  - **验收标准：** HAL的电机控制指令可以通过RTS的通信通道下发，并由一个1kHz的实时任务稳定执行，实现端到端的闭环控制。
- **任务2：实现实时性能监控**
  - **交付物：** 实现`RTSMetrics`数据结构的收集与上报机制，用于统计调度延迟、最大/最小延迟和任务超时（deadline misses）次数。
  - **验收标准：** 数据可以通过DMS数据总线上报，或通过`arf-cli`命令行工具查询。
- **任务3：开发`arf-cli`诊断工具**
  - **交付物：** 实现`arf-cli rts status`和`arf-cli rts metrics`等命令行工具。
  - **验收标准：** 运维人员可以通过CLI工具实时查看RTS的运行状态、CPU核心占用率以及关键性能指标。

#### **第四阶段：鲁棒性与高级功能 (Robustness & Advanced Features)**

**核心目标：** 专注于系统的稳定性和容错能力，增加高级功能，使RTS模块达到生产环境部署标准。

- **任务1：实现任务超时与故障恢复**
  - **交付物：** 为实时任务增加超时检测机制。当一个任务错过其执行截止时间时，系统能够记录该事件并触发预定义的降级策略（例如，安全停止电机）。
  - **验收标准：** 在集成测试中，手动阻塞一个实时任务后，系统能在2-3个周期内检测到超时并执行相应的安全停机指令。
- **任务2：高精度时间同步**
  - **交付物：** 实现RTS与硬件时钟（如主板或外接高精度时钟源）的同步逻辑。
  - **验收标准：** 系统时间与硬件时间的同步误差小于1毫秒。
- **任务3：压力与稳定性测试**
  - **交付物：** 设计并执行长时间（例如72小时）的系统压力测试。
  - **验收标准：** 在高CPU和IO负载下，RTS核心任务的调度抖动依然保持在目标阈值内，且系统无内存泄漏或崩溃。
- **任务4 K：文档与API最终化**
  - **交付物：** 撰写详细的开发者文档，说明如何在其他模块中使用RTS库，包括API使用示例、最佳实践和常见陷阱。
  - **验收标准：** 文档清晰、准确，能够指导一个新开发者在一天内成功地将一个新任务集成到RTS中。

------



## 📚 8. 开发参考资源





### 📖 核心参考文档



- **[Real-Time Linux Wiki](https://wiki.linuxfoundation.org/realtime/start)** - 实时Linux配置指南。
  实时 Linux Wiki - 实时 Linux 配置指南。
- **[PREEMPT_RT 补丁](https://wiki.linuxfoundation.org/realtime/documentation/start)** - 实时内核补丁文档。
- **[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/)** - 现代C++最佳实践。



### 🔧 开发工具链



Bash

```bash
# 实时性能分析工具
sudo apt install rt-tests trace-cmd kernelshark

# 延迟测试
sudo cyclictest -p 80 -m -Sm -q -D 1h

# 系统追踪
sudo trace-cmd record -e sched_switch -e sched_wakeup
```