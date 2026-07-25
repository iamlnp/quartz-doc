# 心跳消息类型

- beacon消息
- failure消息
- alive消息

<img src="osd心跳机制调研/image/image-20220318101339602.png" alt="image-20220318101339602" style="zoom: 50%;" />





# beacon消息

beacon就是osd/mon之间的心跳

```c++
void OSD::send_beacon()

//文件路径：src/messages/MOSDBeacon.h
class MOSDBeacon : public PaxosServiceMessage {

}  


```

# failure消息

场景：failure是上报别的osd挂了

```c++
//文件路径：src/messages/MOSDFailure.h
class MOSDFailure : public PaxosServiceMessage {
    enum {
    FLAG_ALIVE = 0,      // use this on its own to mark as "I'm still alive"
    FLAG_FAILED = 1,     // if set, failure; if not, recovery
    FLAG_IMMEDIATE = 2,  // known failure, not a timeout
  };
}
```

monitor端处理函数：`bool OSDMonitor::prepare_failure()`

处理逻辑：



# alive消息

场景：alive是被误杀后，发给mon的，比如osd.1与osd.2/3的连接有问题，osd.2/3就会上报mon，mon会把osd.1踢掉，osd.1收到后，要喊冤。

```c++
//文件路径：src/message/MOSDAlive.h
class MOSDAlive : public PaxosServiceMessage {
public:
  epoch_t want = 0;  
}
```

