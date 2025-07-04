# Everything I know about good system design
- URL: https://www.seangoedecke.com/good-system-design/
- Added At: 2025-06-21 05:55:43
- [Link To Text](2025-06-21-everything-i-know-about-good-system-design_raw.md)

## TL;DR


本文总结了系统设计的关键原则：优先采用简洁稳定架构，减少复杂性；核心状态集中管理，无状态服务提升容错性；数据库需灵活设计索引并利用读副本分担压力；合理使用缓存与异步处理优化热路径；通过监控关键指标（如p95延迟）和故障处理机制（断路器、幂等性）保障稳定性；强调避免过早复杂，优先使用成熟组件而非过度创新。

## Summary


- **好的系统设计表现**  
  - 长期稳定，用户感觉“比预期更容易”或“无需关注该部分”。  
  - 避免复杂系统，复杂系统应从简单架构逐步演化而来，盲目复杂性可能导致潜在缺陷。

- **状态与无状态原则**  
  - 优先减少有状态组件，因其易陷入不良状态且难以自动恢复。  
  - 核心状态管理应集中在单一服务（如数据库服务），其他服务保持无状态或简单读操作。  
  - 无状态服务（如PDF渲染）可通过容器重启自动修复；有状态服务需手动修复故障。

- **数据库设计要点**  
  - 模式设计需灵活但不过度抽象，避免应用层复杂性。  
  - 根据高频查询建立索引，而非全字段索引，以平衡写入负载。  
  - 高流量下，数据库易成为瓶颈，解决方法包括：  
    - 让数据库处理多表查询而非内存拼接（如JOIN而非多次查询）。  
    - 尽量使用读副本减少主节点压力，允许容忍滞后时延。  
    - 通过限流应对尖峰写操作，避免雪崩效应。

- **缓存与异步处理策略**  
  - 缓存易带来状态同步问题，优化查询（如索引）优先于缓存。  
  - 对于高消耗查询，可使用Redis/Memcached等缓存，或用文档存储(S3)实现持久缓存。  
  - 背景作业的核心模式是分离热路径计算（如PDF渲染首页后异步处理剩余部分）。  
  - 长期任务（如月后触发）可用数据库表记录时间戳，每日扫描执行。

- **事件驱动与API的选择**  
  - 事件总线（如Kafka）适用于解耦和异步处理（如新建账户后触发多个服务动作）。  
  - 特性间强耦合或需即时响应时，优先用API调用而非事件。

- **热路径优先优化**  
  - 关注高频或关键流程（如计费系统的决策路径），因其对系统稳定性影响更大。  
  - 非热路径的设计灵活性更高，但主要精力需投入热路径瓶颈消除。

- **监控与日志实践**  
  - 异常路径需记录失败原因，便于定位用户问题（如422错误需说明具体原因）。  
  - 监控指标包括CPU/内存、队列长度、请求延迟百分位数（关注p95/p99而非平均）。  
  - 用户端延迟超过数百毫秒需警惕，但实际应用中部分场景可接受稍慢响应。

- **故障处理机制**  
  - 杀毒开关：系统崩溃时的快速应急控制。  
  - 重试设计需谨慎，高负载API应配合“断路器”防止加重依赖服务。  
  - 写操作用幂等性键避免重复处理（如无论重试多少次，重复请求仅执行一次）。  
  - 接口失败应区分场景选择策略：  
    - 率限制应失败开放，避免用户受到牵连。  
    - 认证等关键功能需失败关闭，防止数据泄露。  

- **系统设计核心原则**  
  - 优先使用成熟基础组件（如标准队列、缓存服务）而非过度创新。  
  - 技术选型需符合实际需求，盲目追求微服务、容器等可能引发复杂性。  
  - 复杂性需由问题驱动，避免先发制人的架构设计，保持系统简洁。
