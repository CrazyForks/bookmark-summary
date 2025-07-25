# Why reliability is hard at scale: learnings from infrastructure outages
- URL: https://newsletter.pragmaticengineer.com/p/why-reliability-is-hard-at-scale
- Added At: 2025-07-23 13:25:40
- [Link To Text](2025-07-23-why-reliability-is-hard-at-scale-learnings-from-infrastructure-outages_raw.md)

## TL;DR


本文分析了Heroku、Google Cloud与Neon的停机事件，揭示系统规模扩大带来的可靠性挑战。Heroku因Ubuntu自动更新致网络失效，23小时停机且响应滞后；Google全局配置更新未分阶段，扩大影响；Neon扩展时遭遇PostgreSQL瓶颈。核心教训包括禁用生产环境自动更新、隔离关键工具、优化故障响应、重视技术债务及行业经验共享。文章强调，系统可靠性需主动设计而非依赖历史经验，透明快速的故障处理是服务竞争力的关键。

## Summary


文章通过近期3家基础设施服务商的停机事件，揭示了规模化系统中可靠性面临的挑战及经验教训。事件案例及核心问题如下：  

### Heroku  
- **事件描述**：因Ubuntu 22.04的自动系统更新，触发systemd重启，导致网络路由规则失效，丢失外部网络连接，造成服务完全中断23小时，为该平台史上最长停机。  
- **问题表现**：团队8小时后才公开承认全球性故障，5天后发布含少量细节的总结，且一个月后未落实改进措施。  
- **深层原因**：  
  1. 生产环境未禁用自动操作系统更新，且未隔离关键工具（如状态页面）与普通服务，导致自身工具瘫痪时无法与客户沟通；  
  2. 排查过程依赖外部供应商，未并行启动多排查路径，延误了故障定位；  
  3. 对行业案例（如2023年Datadog的同类问题）未充分重视，团队可靠性和紧急响应能力显著退化。  
- **行业关联**：其系统问题与Datadog的$5M停机事故高度相似，暴露了自动更新机制和systemd重启带来的网络风险长期未被解决。  

### Google Cloud  
- **事件描述**：全球同步更新配置导致部分服务崩溃，持续3小时。  
- **问题表现**：更新未按“Fail-Open”原则设计，未通过Feature Flags分阶段推送，导致影响扩大。  
- **改进方向**：若采用分阶段部署，可将故障恢复时间缩短66%。  

### Neon  
- **事件描述**：在扩展规模时遭遇PostgreSQL典型问题，如查询计划失效和真空进程变慢。  
- **问题表现**：作为数据库专家仍未能避免这些问题，证明即使技术团队经验丰富，规模化后仍会出现意料之外的技术瓶颈。  
- **行业警示**：数据库性能问题可能随数据量激增而出现，需提前优化和监控。  

### 共同教训  
1. **自动更新风险**：系统级组件（如Ubuntu的systemd）的自动更新可能对依赖其功能的企业级服务造成致命影响，需严格禁用生产环境的自动更新。  
2. **基础设施依赖单一性**：关键工具（如状态页面、监控系统）与普通服务共用同一配置，故障时导致自我妨碍，需隔离关键路径。  
3. **延迟应对成本**：故障处理流程的迟缓（如Heroku耗时8小时才公开）和沟通不透明会引发用户不满，需优化响应速度和透明度。  
4. **技术债务积累**：规模增长后未持续改进可靠性，导致基础架构薄弱环节暴露（如路由规则未自动复位）。  
5. **知识共享不足**：工程师可能未能关注同类事故的经验总结（如Datadog案例），团队需建立行业事件学习机制。  
6. **人员结构问题**：应对复杂故障时缺乏跨团队协作与专家技能，可能反映组织内技术能力下降或资源分配不当。  

### 行业趋势与警示  
- 大型公司因可靠性不足可能失去市场优势，新兴服务商（如Render、Fly.io、Railway）正争夺其份额。  
- 系统可靠性需主动维护，包括闭环监控、独立运行关键工具、分阶段更新策略、以及持续吸收行业事件经验。  
- 用户对企业服务质量的感知显著影响品牌忠诚度，处理故障的透明度和效率是关键的竞争力指标。  

这些案例表明，随着系统规模扩大，简单错误可能引发连锁反应，可靠性工程需系统化设计而非依赖历史经验。
