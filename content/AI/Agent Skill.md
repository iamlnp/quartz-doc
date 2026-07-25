# 概念
通俗的讲，Agent Skills是大模型随时可以翻阅的智能文档

# 工具
## Cline
需在 Cline（VS Code 插件 / CLI）中开启对应 Skill，且模型需支持 MCP 协议（Claude 3/4 Code、DeepSeek-V3、GPT-4o 均支持。

Cline 高频常用官方 Skills（适配编程核心场景）：  
Cline 官方提供了大量开箱即用的 Skills，**编程开发中最常用的有以下 8 个**，覆盖本地开发全流程，国内对接 OpenRouter 调用 Claude Code/DeepSeek 均可直接使用：

| Skill 名称           | 核心能力                                               | 典型使用场景                                                   |
|--------------------|----------------------------------------------------|----------------------------------------------------------|
| File System        | 本地文件读写、创建 / 删除 / 重命名文件 / 文件夹、遍历目录                  | 读取本地代码库、生成文件到指定目录、批量重命名项目文件                              |
| Terminal           | 调用系统终端执行命令（shell/powershell/cmd）、返回执行结果 / 报错信息     | 安装依赖（pip install/npm install）、运行测试（pytest/npm test）、构建项目 |
| Code Analysis      | 代码语法检查、依赖分析、函数 / 类提取、代码复杂度检测                       | 分析项目代码结构、排查语法错误、快速定位核心函数                                 |
| Code Generation    | 按指定语言 / 规范生成代码、补全代码片段、重构现有代码                       | 生成接口代码、重构老旧代码、补全单元测试                                     |
| Search             | 本地代码内容搜索（按关键词 / 正则）、跨文件检索                          | 项目中搜索指定函数 / 变量、查找硬编码配置、定位 bug 相关代码                       |
| Diff & Merge       | 对比文件差异、合并代码修改、生成 diff 文件                           | 对比 AI 修改后的代码与原代码、合并多文件修改结果                               |
| Documentation      | 自动生成代码注释、接口文档、README.md、按规范生成文档（如 RESTful/JavaDoc） | 为项目补全注释、生成接口文档、完善项目说明                                    |
| Linter & Formatter | 调用代码格式化工具（black/prettier/flake8）、自动修复格式问题          | 格式化 AI 生成的代码、统一项目代码风格、修复简单的格式 / 语法问题                     |
