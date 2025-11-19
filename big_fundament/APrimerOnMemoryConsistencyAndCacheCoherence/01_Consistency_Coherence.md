# Consistency & Coherence

- Consistency: It is the job of consistency (memory consistency, memory consistency model, or memory model) to define shared memory correctness. Consistency definitions provide rules about loads and stores (or memory reads and writes) and how they act upon memory.
- Coherence: whether such out of order behavior is fine

![baseline system model](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CSFundations/system_model.png)

- 多核处理器芯片（Multicore Processor Chip）：
  - 多个单线程核心（Single-threaded Cores），每个核心配备独立私有数据缓存（Private Data Cache）。
  - 共享末级缓存（Last-level Cache, LLC）：所有核心共享，逻辑上是 “内存侧缓存”，用于降低内存访问延迟、提升有效带宽，同时承担内存控制器功能。
- 片外主存（Main Memory）：与 LLC 相连，存储所有数据的原始副本。
- 互连网络（Interconnection Network）：连接各核心的私有缓存、LLC，负责传输相干性请求、数据响应等消息。

## 缓存一致性接口（Coherence Interfaces）

缓存一致性接口是**缓存系统与内存（如处理器核心、DMA引擎）之间的交互规范**，定义了“核心如何发起内存操作请求”“缓存系统如何响应这些请求”以及“操作结果的可见性规则”。其核心作用是**隔离缓存一致性协议的实现细节**，使核心无需了解底层协议（如监听或目录），只需遵循接口规范即可保证内存操作的正确性（原书第2章）。

> “The coherence interface should make the presence of caches and coherence protocols invisible to the cores, presenting the illusion of a single shared memory.”

一致性接口的核心设计目标
1. **透明性（Transparency）**：核心无需感知缓存的存在及一致性协议的细节，只需按“无缓存的理想内存模型”发起Load/Store操作，接口负责适配底层协议（如自动触发相干性请求、等待数据响应）。  
2. **正确性（Correctness）**：接口必须保证：无论底层协议如何实现，核心的内存操作结果都符合第2章定义的两个一致性不变量（SWMR单写者多读者、数据值不变量）。
3. **性能（Performance）**：接口应允许核心与缓存系统高效交互（如流水线化请求、重叠处理），避免过度限制硬件优化（如乱序执行、预取）。

根据“写入操作的同步方式”，一致性接口分为两类，核心差异在于“写入操作何时对其他核心可见”：

| 维度                | 一致性无关接口（Consistency-agnostic）                          | 一致性导向接口（Consistency-directed）                          |
|---------------------|-----------------------------------------------------------------|-----------------------------------------------------------------|
| **核心思想**        | 写入操作在返回前**同步传播**到所有相关缓存，保证“写入完成即全局可见” | 写入操作**异步传播**（返回时可能未被所有核心可见），但最终可见顺序需符合内存一致性模型 |
| **核心感知度**      | 核心无需关注一致性，接口自动保证写入的全局可见性                 | 核心需通过显式同步指令（如FENCE）控制写入的可见顺序               |
| **硬件复杂度**      | 高（需实时处理相干性请求，保证同步传播）                         | 低（异步传播减少即时开销，适合高吞吐量场景）                     |
| **典型适用场景**    | 通用处理器（CPU），如x86、ARM的强一致性模式                     | 吞吐量优先的异构设备（GPU、加速器），如NVIDIA CUDA架构           |
| **关键实现机制**    | 写入时触发失效/更新请求，等待所有响应后才返回（如MSI协议的GetM操作） | 写入先更新本地缓存，通过背景线程/事件触发异步传播（如GPU的L2推送更新） |

> 原文示例：“Consistency-agnostic interfaces are typical in CPUs, where the coherence protocol ensures that a store appears to complete atomically with respect to all other cores. In contrast, consistency-directed interfaces are common in GPUs, where the hardware prioritizes throughput over immediate coherence.”  
> （一致性无关接口在CPU中常见，相干性协议保证Store操作相对于所有核心原子性完成；相反，一致性导向接口在GPU中常见，硬件优先考虑吞吐量而非即时一致性。）


## 2.3.3 一致性接口的核心操作与交互流程
无论接口类型，核心与缓存系统的交互均围绕三类基础操作展开：

1. **Load操作**  
   - 核心行为：发起读取请求（指定地址），等待返回数据。  
   - 接口响应：  
     - 若缓存命中且处于有效状态（M/O/E/S），直接返回数据。  
     - 若缓存未命中或状态无效（I），自动发起相干性请求（如GetS），等待数据返回后更新缓存并响应核心。

2. **Store操作**  
   - 核心行为：发起写入请求（指定地址和数据），等待操作完成。  
   - 接口响应（一致性无关接口）：  
     - 若缓存未持有写权限（非M状态），先发起相干性请求（如GetM或Upgrade），使其他副本失效。  
     - 获得写权限后更新本地缓存，返回“写入完成”信号（此时写入已全局可见）。  

3. **原子操作（Atomic Read-Modify-Write）**  
   - 核心行为：发起“读-改-写”原子请求（如test-and-set），要求操作不可中断。  
   - 接口响应：  
     - 原子性获取目标块的写权限（期间阻塞其他核心的请求）。  
     - 完成“读取旧值→修改→写入新值”的完整流程，确保无中间状态被其他核心观察到。


## 2.3.4 接口与内存一致性模型的关联
一致性接口与内存一致性模型是**协同关系**，二者共同保证共享内存的正确性：  
- **一致性接口**（硬件层）：保证“单地址副本的一致性”（通过相干性协议实现）。  
- **内存一致性模型**（架构层）：定义“多地址操作的顺序规则”（通过接口约束核心操作的可见性）。  

例如：在顺序一致性（SC）模型中，一致性无关接口需保证“Store操作按程序序全局可见”；而在松弛模型（如XC）中，一致性导向接口允许“Store操作延迟可见”，仅通过FENCE指令强制顺序。

> 原文总结：“The coherence interface implements the low-level mechanisms to maintain coherence, while the memory consistency model defines the high-level rules for ordering memory operations. Together, they ensure that shared memory behaves correctly.”  
> （一致性接口实现维护相干性的底层机制，而内存一致性模型定义内存操作排序的高层规则，二者共同保证共享内存的正确行为。）


## 关键结论
2.3章通过明确“核心与缓存系统的交互规范”，为后续章节讲解相干性协议（监听/目录）奠定了接口基础。一致性无关接口和一致性导向接口的划分，本质是**在“可编程性”与“性能”之间的权衡**——前者适合CPU等需要强一致性的场景，后者适合GPU等追求高吞吐量的异构设备。