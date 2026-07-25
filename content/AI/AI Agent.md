---
写作年份:
  - "2025"
imageNameKey: AI Agent
tags:
  - Agent
---
# 2026-7-21   
Hermes Agent：NousResearch 开源 CLI 本地智能体，**框架本身永久免费**，只需要自己买第三方大模型 API 套餐提供算力，能自主读写文件、执行命令、批量改工程、自动写单元测试、提交 Git PR，专门给后端 / 全栈开发者做自动化编码。  

一键安装  
```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```
安装后重载环境  
```bash
# bash 用户 
source ~/.bashrc 
# zsh（Mac默认） 
source ~/.zshrc
```

验证安装  
```bash
hermes --version 
# 环境自检（排查依赖缺失） 
hermes doctor
```

国内 API 配置  
Hermes 本身免费，**必须配置大模型 API 密钥才能读写代码、执行任务**  
