# </> htmx ~ Codin' Dirty
- URL: https://htmx.org/essays/codin-dirty/
- Added At: 2024-11-26 14:58:06
- [Link To Text](2024-11-26-htmx-~-codin'-dirty_raw.md)

## TL;DR
文章介绍了作者的“脏”编程方法，包括使用大函数、偏好集成测试和最小化类与接口，通过实际项目案例说明这些方法的有效性，并鼓励开发者不必拘泥于传统编程规范。

## Summary
1. **引言**：
   - 作者介绍自己的编程方法为“codin’ dirty”，即与《Clean Code》推荐的方法相反。
   - 作者不认为自己的代码“脏”，而是认为其易于维护且质量合理。
   - 目的不是说服读者采用“脏”编程，而是展示这种方法也能成功，并提供软件方法论讨论的平衡视角。

2. **编程实践**：
   - **大函数**：
     - 作者认为大函数在某些情况下是好的，尤其是核心功能。
     - 与《Clean Code》推荐的小函数相反，作者认为重要功能应大，不重要功能应小。
     - 通过SQLite、Chrome、Redis和IntelliJ等成功项目的例子，说明大函数在实际项目中的应用。
   - **集成测试优于单元测试**：
     - 作者偏好集成测试而非单元测试，认为集成测试更能保持稳定性和长期有用性。
     - 早期项目中避免过度单元测试，等待核心API和概念明确后再进行集成测试。
   - **最小化类和接口**：
     - 作者倾向于减少类和接口的数量，避免过度分解。
     - 通过Active Record的例子，说明单一类处理多种功能可以简化系统。

3. **结论**：
   - 总结了三种“脏”编程实践：大函数、集成测试、最小化类和接口。
   - 强调这些方法并非最优，而是展示不同方法也能成功，鼓励年轻开发者不必拘泥于传统方法。
