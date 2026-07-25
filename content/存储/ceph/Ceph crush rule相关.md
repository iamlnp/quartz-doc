---
写作年份:
  - "2025"
imageNameKey: crush rule
tags:
  - ceph
  - crush
  - 技术学习
---
# 1 命令行
1. ceph osd crush rule ls
```
[10:30:36] root@ceph-221:/home/rustfs# ceph osd crush rule ls
replicated_rule
tfs-rule
```
2. ceph osd crush rule dump
```
[10:30:22] root@ceph-221:/home/rustfs# ceph osd crush rule dump
[
    {
        "rule_id": 0,
        "rule_name": "replicated_rule",
        "ruleset": 0,
        "type": 1,
        "min_size": 1,
        "max_size": 10,
        "steps": [
            {
                "op": "take",
                "item": -1,
                "item_name": "default"
            },
            {
                "op": "choose_firstn",
                "num": 0,
                "type": "osd"
            },
            {
                "op": "emit"
            }
        ]
    },
    {
        "rule_id": 1,
        "rule_name": "tfs-rule",
        "ruleset": 1,
        "type": 1,
        "min_size": 1,
        "max_size": 10,
        "steps": [
            {
                "op": "take",
                "item": -8,
                "item_name": "tfs"
            },
            {
                "op": "chooseleaf_firstn",
                "num": 0,
                "type": "host"
            },
            {
                "op": "emit"
            }
        ]
    }
]
```
3. 获取crush map
- ceph osd getcrushmap -o [obj_path]
- crushtool -d [obj_path] -o [file_path]
```
ceph osd getcrushmap -o /tmp/crush_rule
crushtool -d /tmp/crush_rule -o /tmp/crush_rule.txt
```

```bash
# begin crush map
tunable choose_local_tries 0  # 定义在本地域（如同一主机、机架）内选择 OSD 时的最大尝试次数, 设为 0 表示不优先尝试本地域，直接从更大范围选择
tunable choose_local_fallback_tries 0  # 当本地域内选择 OSD 失败时，重试的最大次数, 设为 `0` 表示本地域选择失败后不重试，直接 fallback 到其他域
tunable choose_total_tries 50 # 选择 OSD 时的总尝试次数（跨所有域）,超过该次数仍未找到合适 OSD 会返回失败，50 是较常见的默认值
tunable chooseleaf_descend_once 1 # 控制 CRUSH 在选择叶节点（如 OSD）时是否只向下遍历一次层级（如从 root → datacenter → rack → host → OSD）, 1 表示启用，减少遍历次数以提高性能；0 表示允许回溯重试
tunable chooseleaf_vary_r 1 # 选择叶节点时是否随机化起始点（r 是 CRUSH 算法中的随机种子）, 1 表示启用随机化，避免数据分布倾斜；0 表示固定起始点。
tunable chooseleaf_stable 1 # 控制叶节点选择是否优先保证稳定性（即尽量保持数据映射不变）, 1 表示优先稳定，减少数据迁移；0 可能更倾向于均匀分布，但迁移更多
tunable straw_calc_version 1 # 指定 CRUSH 中 "straw" 选择算法的计算版本（straw 是 CRUSH 中用于在多个候选者中公平选择的机制）, 版本 1 是较新的实现，通常性能和公平性更好
tunable allowed_bucket_algs 54 # 限制 CRUSH 桶（bucket）可使用的算法类型（通过位掩码指定）, 54 对应的二进制是 110110，表示允许以下算法: 2（uniform）、4（list）、8（tree）、32（straw2）

# devices
# Ceph 集群中 OSD 设备与存储类（Storage Class）的绑定配置，核心作用是将 OSD 实例（存储数据的核心进程）与物理设备关联，并标记设备类型（这里是 hdd，机械硬盘），用于后续数据放置策略（如 CRUSH 规则）的调度。
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd

# types
# CRUSH 层级结构类型定义
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root

# buckets
host ceph-221 {  # 
	id -5		# do not change unnecessarily 唯一标识符（负数，Ceph 内部使用，不可随意修改）
	id -6 class hdd		# do not change unnecessarily 该节点关联的存储类（hdd）及对应 ID（-6）
	# weight 0.488
	alg straw2  # 该节点内选择子节点的算法（straw2 是默认高效算法）s
	hash 0	# rjenkins1 哈希函数（0 表示 rjenkins1，Ceph 推荐的哈希算法）
	item osd.1 weight 0.488 # 子节点（osd.1）及其实例权重（0.488，影响数据分配比例）
}
rack rack1 {
	id -3		# do not change unnecessarily
	id -4 class hdd		# do not change unnecessarily
	# weight 0.488  # 节点总权重
	alg straw2
	hash 0	# rjenkins1
	item ceph-221 weight 0.488
}
root default {
	id -1		# do not change unnecessarily
	id -2 class hdd		# do not change unnecessarily
	# weight 0.488
	alg straw2
	hash 0	# rjenkins1
	item rack1 weight 0.488
}
host tfs_ceph-221 {
	id -11		# do not change unnecessarily
	id -12 class hdd		# do not change unnecessarily
	# weight 0.781
	alg straw2
	hash 0	# rjenkins1
	item osd.0 weight 0.391
	item osd.2 weight 0.391
}
rack tfs_rack1 {
	id -7		# do not change unnecessarily
	id -9 class hdd		# do not change unnecessarily
	# weight 0.781
	alg straw2
	hash 0	# rjenkins1
	item tfs_ceph-221 weight 0.781
}
root tfs {
	id -8		# do not change unnecessarily
	id -10 class hdd		# do not change unnecessarily
	# weight 0.781
	alg straw2
	hash 0	# rjenkins1
	item tfs_rack1 weight 0.781
}

# rules
rule replicated_rule {
	id 0
	type replicated
	min_size 1
	max_size 10
	step take default # 步骤 1：从根节点 `default` 开始选择（限定数据只能在该根节点下的 OSD 分布）
	step choose firstn 0 type osd # 步骤 2：选择 OSD（firstn 0 表示选择与副本数相等的 OSD，type osd 直接选 OSD 级）
	step emit # 步骤 3：输出选中的 OSD 列表（完成选择）
}
rule tfs-rule {
	id 1
	type replicated
	min_size 1
	max_size 10
	step take tfs
	step chooseleaf firstn 0 type host # 步骤 2：选择叶节点（type host 表示副本需跨主机）
	step emit
}

# end crush map
```

4. 修改crush rule
- 手动绑定 OSD 与存储类: ceph osd crush class set osd.0 hdd
- 查看 OSD 与存储类的绑定关系: ceph osd tree
5. ceph osd pool get <pool_name> xxx
6. ceph osd crush dump


# 2 CRUSH 层级结构

| 类型编号 | 类型名（type）  | 含义说明                                                   |
|------|------------|--------------------------------------------------------|
| 0    | osd        | 对象存储设备（最底层），直接存储数据的物理 / 虚拟设备（如硬盘、SSD），是数据分布的最终目标。      |
| 1    | host       | 主机（服务器），包含一个或多个 osd（一台服务器可插多块硬盘，每块对应一个 OSD）。           |
| 2    | chassis    | 机框（机架上的物理机框），可包含多个 host（同一机框内的服务器）。                    |
| 3    | rack       | 机架，包含多个 chassis 或 host（同一机架上的设备）。                      |
| 4    | row        | 行（机房内的服务器行），包含多个 rack（同一行的机架）。                         |
| 5    | pdu        | 电源分配单元（Power Distribution Unit），通常关联到同一供电范围内的设备（可选层级）。 |
| 6    | pod        | 机柜组（一组相邻的机架），包含多个 rack 或 row（多见于大型数据中心）。               |
| 7    | room       | 机房（物理房间），包含多个 pod、row 或 rack。                          |
| 8    | datacenter | 数据中心，包含多个 room 或 pod（跨机房冗余的核心层级）。                      |
| 9    | zone       | 区域（如城市内的不同数据中心集群），包含多个 datacenter。                     |
| 10   | region     | 大区（如国家 / 洲），包含多个 zone（跨国 / 跨洲集群的顶层层级）。                 |
| 11   | root       | 根节点（最顶层），每个 CRUSH 映射表至少有一个 root，所有层级最终都归属于某个 root。     |
# 3 数据重平衡
## 3.1 迁移权重 - reweight
通过调整reweight的数值，可以使得一定数量的PG迁入或者迁出对应的OSD，从而使得OSD之间的PG数量趋于均衡。
1. osd使用率
```bash
ceph osd df tree
```
2. 调整reweight
```bash
ceph osd reweight {osd_num} {reweight}
```

## 3.2 weight-set
针对存储池中的每个OSD按照副本数设置一个权重组，称为weight-set

## 3.3 upmap
upmap作用在CRUSH完成选择之后，用于直接对CRUSH的选择结果进行调整[插图]。视调整粒度不同，当前有两种类型的upmap，一种针对CRUSH选择结果整体进行替换：
1. 一种针对CRUSH选择结果整体进行替换
```bash
ceph osd pg-upmap <pgid><osdname (id|osd.id)> [<osdname (id|osd.id)>...]
```
2. 一种则只针对某个或者某些副本进行替换（注意：需要同时指定源和目的OSD，以实现将源OSD替换为目标OSD的功能
```bash
ceph osd pg-upmap-items <pgid><osdname (id|osd.id)> [<osdname (id|osd.id)>...]
```
如果不再需要（或者已经失效）​，则可以通过如下命令删除
```bash
ceph osd rm-pg-upmap <pgid> 
ceph osd rm-pg-upmap-items <pgid>
```
由于upmap直接作用于客户端寻址过程，所以如果客户端无法理解upmap（例如使用老版本的内核客户端）​，则无法启用upmap机制。
## 3.4 balancer
在Luminous版本中，Ceph为Mgr组件引入了一个balancer模块，用于对集群的数据分布自动进行调整