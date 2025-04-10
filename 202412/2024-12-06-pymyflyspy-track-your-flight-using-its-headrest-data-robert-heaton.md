# PyMyFlySpy: track your flight using its headrest data | Robert Heaton
- URL: https://robertheaton.com/pymyflyspy/
- Added At: 2024-12-06 14:46:18
- [Link To Text](2024-12-06-pymyflyspy-track-your-flight-using-its-headrest-data-robert-heaton_raw.md)

## TL;DR
作者为满足儿子在飞机上询问飞行位置的需求，开发了PyMyFlySpy应用，通过分析机上Wi-Fi数据端点，实时计算并显示飞行路径和指标，支持多航空公司和自定义解析器，未来计划增加事件订阅功能。

## Summary
1. **背景故事**：
   - 作者与五岁儿子在飞机上，儿子询问飞行位置。
   - 作者尝试通过机上Wi-Fi连接到FlightRadar获取信息，但因使用PySkyWiFi项目而拒绝付费。

2. **发现数据端点**：
   - 作者连接到机上Wi-Fi网络，发现支付页面显示飞行信息（速度、方向、预计到达时间）。
   - 通过Chrome开发者工具，发现浏览器定期请求`/info`端点，该端点返回大量数据，但经度和纬度字段为空。

3. **解决方案构思**：
   - **方案一**：利用速度和方向数据，动态计算飞行位置。
   - **方案二**：开发一个Web应用，实时显示动态计算的位置，并提供自动更新的图表和查询界面。

4. **项目开发**：
   - 作者在假期期间开发了PyMyFlySpy应用，代码发布在GitHub上，支持“dummy”模式用于非飞行时演示。

5. **功能概述**：
   - **地图和图表**：显示飞行路径和当前飞行指标的变化。
   - **查询界面**：允许用户对记录的数据进行查询，如最大速度和风速。
   - **多航空公司支持**：支持不同航空公司的Wi-Fi系统，用户可添加自定义解析器。

6. **系统设计**：
   - **组件**：Firefox扩展、本地Web服务器、SQLite数据库、Web前端。
   - **设计选择**：使用Firefox扩展而非直接抓取数据，以避免与航空公司服务器直接交互。

7. **未来工作**：
   - 计划实现事件订阅功能，如根据飞行数据触发特定事件（如限制访问特定网站或发送Slack消息）。

8. **实际应用**：
   - 作者在回程航班上启动PyMyFlySpy，尽管儿子已入睡，作者仍监控和调试应用，确保其正常运行。
