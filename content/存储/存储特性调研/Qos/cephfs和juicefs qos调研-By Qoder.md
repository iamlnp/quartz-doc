---
年份:
  - "2026"
月份:
  - 7月
imageNameKey: cephfs和juicefs qos调研
创建时间: 2026-07-24 15:03
修改时间: 2026-07-24 15:04
tags:
  - qos
---

## 一、CephFS QoS 实现方案

CephFS 的 QoS 涉及**多个层次**，从客户端到 MDS 到 OSD，各层有不同的机制：

### 1. MDS 层 — dmClock 调度器（元数据请求 QoS）

这是 CephFS 最核心的 QoS 机制，由日本 Line 公司开发，PR [#52147](https://github.com/ceph/ceph/pull/52147) 于 2026年7月合入 main。

**核心设计：**
- 基于 **dmClock 算法**（分布式 mClock），在 MDS 端对客户端的**元数据请求**（create、mkdir、lookup 等）进行调度
- 以 **subvolume（子卷）** 为粒度进行 QoS 控制
- 每个 subvolume 可配置三个参数：
  - `reservation`：预留 IOPS（保证最低性能）
  - `weight`：权重（超额资源按比例分配）
  - `limit`：上限 IOPS（不超过此值）

**工作方式：**
```
# 为某个 subvolume 设置 QoS
ceph --admin-daemon mds.a.asok qos set /volumes/_nogroup/subvolume 1000 1000 1000
#                                                      reservation  weight  limit
```

- MDS 维护一个 **dmClock 优先队列**，根据 reservation/weight/limit 选择下一个处理的请求
- 多个客户端共享同一 subvolume 时，QoS 是在这些 session 之间**聚合**的
- 这是**服务端（MDS）阻塞调度**方式——请求在 MDS 队列中排队等待，不是返回错误给客户端

### 2. RADOS/OSD 层 — dmClock 调度器（数据 I/O QoS）

- 从 Quincy (v17) 开始，dmClock 成为 OSD 的**默认调度器**
- 区分客户端 I/O、恢复 I/O、Scrub I/O 等，按优先级/权重调度
- 在 OSD 的 op 队列层面实现，属于**后端阻塞式调度**

### 3. RBD 层 — TokenBucketThrottle（块设备 I/O QoS）

- 在 librbd 层实现，基于 [TokenBucketThrottle](file:///home/code/ceph/src/common/Throttle.h#L335-L467)
- 支持 IOPS 和 BPS 限制，支持读写分离限制，支持 burst
- 实现在 `ImageRequestWQ`（工作队列）中，请求进入队列时先通过令牌桶检查
- **令牌不足时：请求在队列中等待**（阻塞），不是返回错误

### 4. CephFS 客户端层 — Quota（配额）

- 通过扩展属性 `ceph.quota.max_bytes` 实现目录级**空间配额**
- 配额用尽时，写入返回 **ENOSPC**（类似硬限制）
- 这是**唯一返回错误的场景**，但它限制的是空间而非速率

### CephFS 总结

| 层次 | 算法 | 限流对象 | 超限行为 | 阻塞 vs 返回错误 |
|------|------|---------|---------|----------------|
| MDS | dmClock | 元数据请求 | 排队等待 | **阻塞** |
| OSD | dmClock | 数据 I/O op | 排队等待 | **阻塞** |
| RBD | TokenBucket | 块设备 I/O | 队列等待 | **阻塞** |
| Client | Quota | 空间/inode | 返回 ENOSPC | **返回错误** |

---

## 二、JuiceFS QoS 实现方案

JuiceFS 的 QoS 分为**带宽限速**和**空间配额**两个维度：

### 1. 带宽限速 — 令牌桶（客户端侧）

JuiceFS 通过挂载参数实现：
```bash
juicefs mount --upload-limit 100 --download-limit 200 ...
```

**核心设计：**
- 使用 Go 的 `github.com/juju/ratelimit` 库，基于**令牌桶算法**
- 限速作用在**客户端到对象存储**之间的数据传输通道上
- 上传/下载分别独立限速，单位为 Mbps

**工作方式：**
- 每次向对象存储读写数据时，先从令牌桶中消耗对应字节数的令牌
- **令牌不足时：阻塞等待**（`ratelimit.Reader` / `ratelimit.Writer` 包装 io 流，在 Read/Write 前自动等待令牌）
- 这是**纯客户端行为**，不涉及服务端协调

### 2. 空间配额 — 分布式配额系统

**核心设计：**
- 支持 4 级配额：文件系统总配额、目录配额、用户配额、用户组配额
- 限制两类资源：Space（空间，4KiB 对齐）和 Inodes（文件数）
- 配额信息存储在**元数据引擎**（Redis/SQL/TKV）中

**工作方式：**
- 每个客户端在内存中缓存配额用量，**每 3 秒异步批量提交**到元数据引擎
- 客户端每 12 秒心跳从元数据引擎同步最新配额信息
- 写入前通过 `Quota.check()` 预检查（递归检查所有父目录配额）
- **配额用尽时：**
  - 文件系统级配额用尽 → 返回 **ENOSPC**
  - 目录级配额用尽 → 返回 **EDQUOT**

### JuiceFS 总结

| 维度 | 算法 | 作用位置 | 超限行为 | 阻塞 vs 返回错误 |
|------|------|---------|---------|----------------|
| 上传带宽 | 令牌桶 (juju/ratelimit) | 客户端 | io 流阻塞等待 | **阻塞** |
| 下载带宽 | 令牌桶 (juju/ratelimit) | 客户端 | io 流阻塞等待 | **阻塞** |
| 空间配额 | 分布式缓存 + 异步同步 | 客户端 + 元数据引擎 | 返回 ENOSPC/EDQUOT | **返回错误** |

---

## 三、对比总结

| 维度 | CephFS | JuiceFS |
|------|--------|---------|
| **元数据 QoS** | dmClock 调度（MDS 端，阻塞排队） | 无（FIFO 处理） |
| **数据 I/O 限速** | RBD TokenBucket（阻塞等待） | 客户端令牌桶（阻塞等待） |
| **OSD 级 QoS** | dmClock（区分客户端/恢复/scrub） | 无（依赖对象存储自身） |
| **空间配额** | xattr 扩展属性，返回 ENOSPC | 元数据引擎分布式配额，返回 ENOSPC/EDQUOT |
| **配额粒度** | 目录级（xattr） | 文件系统/目录/用户/用户组 |
| **一致性模型** | 强一致（MDS 集中控制） | 最终一致（客户端缓存 + 异步同步，~10s 延迟） |
| **限速算法** | dmClock + TokenBucket | TokenBucket (juju/ratelimit) |
| **超限策略** | 几乎全是**阻塞等待** | 带宽限速=**阻塞**，配额=**返回错误** |

### 关键设计差异

1. **CephFS 偏向"阻塞等待"**：无论是 MDS 的 dmClock 还是 RBD 的 TokenBucket，超限后请求排队等待而非返回错误。这保证了 POSIX 语义的兼容性，但需要服务端维护调度队列。
2. **JuiceFS 更简洁**：带宽限速在客户端用令牌桶阻塞 IO 流；空间配额用尽则直接返回错误。没有服务端的 QoS 调度器，架构更轻量。
3. **CephFS 的 dmClock 是"比例公平"调度**：reservation/weight/limit 三元组可以保证最低性能、按权重分配剩余资源、限制上限——这比简单的令牌桶限速更精细，但实现复杂度也更高。