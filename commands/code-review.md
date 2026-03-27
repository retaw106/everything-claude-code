# Code Review

对未提交更改进行全面的安全和质量审查：

1. 获取已更改的文件：git diff --name-only HEAD

2. 对于每个已更改的文件，检查：

**安全问题（CRITICAL）：**
- 硬编码的凭证、API 密钥、令牌
- SQL 注入漏洞
- XSS 漏洞
- 缺少输入验证
- 不安全的依赖项
- 路径遍历风险

**代码质量（HIGH）：**
- 函数超过 50 行
- 文件超过 800 行
- 嵌套深度超过 4 层
- 缺少错误处理
- console.log 语句
- TODO/FIXME 注释
- 公共 API 缺少 JSDoc

**最佳实践（MEDIUM）：**
- 可变模式（改用不可变）
- 代码/注释中使用 emoji
- 新代码缺少测试
- 可访问性问题（a11y）

3. 生成报告，包含：
   - 严重程度：CRITICAL、HIGH、MEDIUM、LOW
   - 文件位置和行号
   - 问题描述
   - 建议的修复

4. 如果发现 CRITICAL 或 HIGH 问题，阻止提交

永远不要批准有安全漏洞的代码！
