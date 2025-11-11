# Consistency & Coherence

- Consistency: It is the job of consistency (memory consistency, memory consistency model, or memory model) to define shared memory correctness. Consistency definitions provide rules about loads and stores (or memory reads and writes) and how they act upon memory.
- Coherence: whether such out of order behavior is fine

![baseline system model](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CSFundations/system_model.png)

- 多核处理器芯片（Multicore Processor Chip）：
  - 多个单线程核心（Single-threaded Cores），每个核心配备独立私有数据缓存（Private Data Cache）。
  - 共享末级缓存（Last-level Cache, LLC）：所有核心共享，逻辑上是 “内存侧缓存”，用于降低内存访问延迟、提升有效带宽，同时承担内存控制器功能。
- 片外主存（Main Memory）：与 LLC 相连，存储所有数据的原始副本。
- 互连网络（Interconnection Network）：连接各核心的私有缓存、LLC，负责传输相干性请求、数据响应等消息。