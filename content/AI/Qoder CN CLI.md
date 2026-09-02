---
年份:
  - "2026"
月份:
  - 7月
imageNameKey: Qoder CN CLI
创建时间: 2026-07-30 15:01
修改时间: 2026-07-30 15:01
公司: LN
tags:
  - Qoder
---
```bash
# 直接运行完整程序名 
qoderclicn --version
```

设置别名  
```bash
# 1. 加入PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> /root/.bashrc
# 2. 设置别名
echo "alias qodercli='qoderclicn'" >> /root/.bashrc
# 3. 生效配置
source /root/.bashrc
# 4. 验证
qodercli --version
```

绕过 PATH，直接全路径运行  
```bash
/root/.local/bin/qoderclicn --version
```