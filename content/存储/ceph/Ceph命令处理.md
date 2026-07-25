ceph使用/usr/bin/ceph这个python脚本解析ceph相关的命令行，比如ceph -s、ceph tfs add_data_pool等等

在该脚本中main函数通过`parse_cmdargs`解析命令行参数

前端比如上述ceph脚本将命令行参数解析为JSON对象，对应的处理程序比如monitor将该JSON对象解析为std::map格式，然后在做后续处理

monitor相关命令行定义格式位置：`mon/MonCommands.h`
