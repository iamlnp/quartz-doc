---
状态:
  - 进行中
年份:
  - "2025"
imageNameKey: ceph tell命令
tags:
  - ceph
---
需要从客户端命令解析、消息发送和守护进程命令处理三个层面入手，涉及多个代码文件。以下是关键代码路径和核心实现的梳理
# 1 命令行注册
Ceph 客户端通过 `register_commands()` 函数注册所有支持的命令，`tell` 命令的定义也在这里。核心逻辑是解析 `tell <target> <cmd>` 格式的参数，提取目标守护进程（如 `osd.0`）和具体指令（如 `status`）。




# 备忘
```c++
bool is_tell() const {
    return has_flag(MonCommand::FLAG_TELL);
}

void MonClient::_send_command(MonCommand *r) {}
void MonClient::_check_tell_commands() {}

```

ceph.in

ceph_parse.py
```bash
def find_cmd_target(childargs):
```