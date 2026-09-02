# 安装  
1. 方式 1：官网图形版
- 打开官网：[https://ollama.com/](https://link.wtturl.cn/?target=https%3A%2F%2Follama.com%2F&scene=im&aid=582478&lang=zh "autolink") 点击 Download for macOS
- 解压，把 Ollama.app 拖入「应用程序」
- 首次打开，授权权限；顶部菜单栏出现🦙图标代表运行成功
2. Homebrew CLI 版本
```bash
brew install --cask ollama 
# 开机自启后台服务 
brew services start ollama
# 验证
ollama --version
```
# 根据你的 Mac 内存选择模型  
Apple Silicon 统一内存，Metal 自动加速，无需额外驱动  

| Mac 内存 | 推荐模型（中文友好）   | 拉取命令                  | 用途            |
|--------|--------------|-----------------------|---------------|
| 8GB    | qwen3:4b     | ollama pull qwen3:4b  | 简单总结、改写笔记     |
| 16GB   | qwen3:8b（首选） | ollama pull qwen3:8b  | 日常笔记问答、思考、RAG |
| 24GB+  | qwen3:14b    | ollama pull qwen3:14b | 复杂长文分析、技术文档解读 |
额外必下向量嵌入模型（实现本地 RAG，让 AI 读取你的笔记库）  
```bash
ollama pull nomic-embed-text
```

下载完成后查看本地模型清单：  
```bash
ollama list
```