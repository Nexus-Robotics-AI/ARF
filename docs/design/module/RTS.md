# ARF 模块深度解析：RTS (实时调度层)

**文档版本:** 1.0
**模块负责人:** 开发者 B (执行工程师)
**状态:** **设计完成 - 待开发**

## 1. 模块概述 (Module Overview)

### 1.1 核心使命 (Core Mission)

RTS是ARF框架的“**脊髓反射中枢**”。它的核心使命是为整个系统提供一个**确定性的 (Deterministic)、可预测的**任务执行环境，确保那些对时间有严苛要求的关键任务（如电机控制回路、安全停机指令）能够**在规定的时间窗内（deadline）被准时执行**，不受非实时任务（如日志记录、网络通信）的干扰。

### 1.2 所属平面 (Plane)

ARF 边缘平面 (Edge Plane)

### 1.3 主选技术栈 (Primary Tech Stack)

* **`C++` (现代C++17/20):** 实时系统领域的绝对王者，提供了最底层的内存和线程控制能力，以及最丰富的实时生态系统。
* **`PREEMPT_RT` (实时抢占内核补丁):** 将标准的Linux内核转变为一个功能完备的硬实时操作系统。

## 2. 核心需求 (Core Requirements)

| ID | 需求描述 | 验收标准 | 优先级 |
| :--- | :--- | :--- | :--- |
| **R1** | **确定性延迟 (Deterministic Latency)** | RTS调度的任务，其执行延迟（Jitter）必须控制在微秒级（µs）。 | **最高** |
| **R2** | **高优先级抢占 (Preemption)** | 实时任务必须能够抢占任何非实时的、低优先级的系统任务，包括内核任务。 | **最高** |
| **R3** | **高频周期性任务 (High-Frequency Tasks)** | 必须支持以极高频率（如1kHz或更高）稳定、周期性地执行任务，用于伺服控制等场景。 | **最高** |
| **R4** | **RT/NRT隔离与通信 (Isolation & Communication)** | 必须提供一个**实时安全**的通信机制，允许非实时（NRT）线程（如gRPC服务线程）向实时（RT）线程传递数据，而不会破坏实时性。 | **高** |
| **R5** | **CPU核心亲和性 (CPU Affinity)** | 必须支持将实时任务绑定到特定的、隔离的CPU核心上运行，以避免被其他进程干扰。 | **高** |
| **R6** | **避免动态内存分配 (No Dynamic Memory)** | 在实时任务的执行循环中，**严禁**任何可能导致不可预测延迟的操作，特别是动态内存分配（`new`, `malloc`）。 | **最高** |

## 3. 接口定义 (Interface Definition)

与`HAL`等模块不同，`RTS`**不是一个gRPC服务**，而是一个**核心库**。它将被其他需要实时能力的底层模块（主要是`HAL`）直接链接和调用。因此，它的“接口”就是其C++头文件中定义的API。

**核心API草案 (`rts/include/rts/scheduler.hpp`):**
```cpp
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



## 4. 设计思路与架构 (Design Philosophy & Architecture)





### 4.1 核心架构：双世界模型 (Two-World Model)



我们的RTS设计基于一个经典的实时系统模型，将整个系统划分为两个世界：

- **实时世界 (RT World):** 由少数几个高优先级的、运行在隔离CPU核心上的实时线程组成。这些线程执行的是对时间极其敏感的控制回路。
- **非实时世界 (NRT World):** 包含系统中的绝大多数线程，如gRPC服务线程、日志线程、业务逻辑线程等。它们运行在普通的CPU核心上。

**RTS的职责，就是这两个世界之间的“守护者”和“信使”。**



### 4.2 守护者：基于`PREEMPT_RT`的抢占式调度



- **实现思路:**

  1. **内核准备:** 我们的机器人操作系统镜像，必须基于一个已经打上了`PREEMPT_RT`补丁的Linux内核进行构建。
  2. **CPU隔离:** 在系统启动时，通过内核启动参数（`isolcpus`）将一个或多个CPU核心从通用调度器中隔离出来，专门留给实时任务使用。
  3. **高优先级线程:** 当`TaskScheduler::schedule_periodic_task`被调用时，它内部会创建一个**pthreads**线程，并使用`pthread_setschedparam`系统调用，将其调度策略设置为`SCHED_FIFO`（先进先出），并赋予其极高的实时优先级（例如98）。同时，使用`pthread_setaffinity_np`将其绑定到我们隔离的CPU核心上。

  - **效果:** 这个线程将拥有“超级权限”，可以抢占系统上几乎任何其他进程，确保自己能被准时唤醒和执行。



### 4.3 信使：RT/NRT通信机制



**这是设计的关键难点 (R4)。** NRT世界的gRPC线程不能直接调用RT世界的方法，因为这会引入锁和不确定性。

- **设计方案：使用“无锁单生产者单消费者队列” (Lock-free SPSC Queue)。**
  - **数据流:**
    1. `HAL`的`MotorActuatorService`（NRT世界）在收到一个gRPC请求后，它不会去直接控制电机。
    2. 它会将指令（如`MotorCommand`）**`push`到一个无锁队列**中。这个`push`操作是**实时安全**的，不会阻塞。
    3. `RTS`调度的一个高频实时任务（RT世界），在其`on_execute`循环中，会从这个无锁队列中**`pop`**最新的指令。
    4. 然后，这个RT任务根据指令，通过总线或直接内存映射，向电机硬件发送控制信号。
  - **优点:** 这种方式完美地解耦了两个世界。NRT世界可以随时、安全地向RT世界“投递”指令，而RT世界则可以按照自己严格的节拍，从“信箱”中取出并执行指令，完全不受NRT世界时序抖动的影响。



## 5. 第一阶段（Phase 1）开发任务

- **任务1：设计并实现RTS核心API**
  - **语言:** C++。
  - **交付物:** 完整的`scheduler.hpp`和`task.hpp`头文件，定义好`TaskScheduler`, `RealTimeTask`等核心类和方法。
- **任务2：实现一个“软实时”调度器**
  - **功能:** 实现`TaskScheduler`的**非`PREEMPT_RT`版本**。它仍然使用pthreads，但使用标准的`SCHED_OTHER`调度策略。
  - **目的:** 这使得我们可以在**任何标准的Linux开发机**上进行RTS库的开发和测试，而无需立即准备一个完整的实时内核环境。
- **任务3：编写基准测试 (Benchmark)**
  - **交付物:** 一个C++测试程序，它会调度一个1kHz的周期性任务。
  - **验收标准:** 该程序需要测量并打印出任务执行周期的**抖动（Jitter）**。在软实时版本下，这个抖动可能在毫秒级；未来在硬实时内核上，它应该降低到微秒级。这个基准测试将是衡量我们RTS模块性能的黄金标准。