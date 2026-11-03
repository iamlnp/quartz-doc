> 本文详细解读 RFC 8435《Parallel NFS (pNFS) Flexible File Layout》第 2.3 节（State and Locking Models）及其关联的第 13 节（Recalling a Layout）、第 14 节（Client Fencing）、第 15.1 节（RPCSEC_GSS and Security Services）。
>
> 前置概念：
> - **MDS**（metadata server）：元数据服务器，负责元数据、布局（layout）的发放与回收、全部锁状态管理。
> - **DS**（storage device）：存储设备，pNFS 场景下通过 NFS（v3/v4.0/v4.1+）对外提供数据文件 I/O。
> - **stateid**：NFSv4 的 128 位状态标识，关联 openowner / lockowner / 委托 / 布局等状态。
> - **租约（lease）**：NFSv4 客户端通过续约（SEQUENCE 等）维持状态有效的时间窗口；租约过期则服务器可吊销状态。
> - **fencing**：MDS 阻止存储设备继续接受某客户端对某数据文件的 I/O 的机制（松耦合下靠合成 uid/gid 改属主实现，见"合成 uid/gid"笔记）。

---

# 1. 总览：锁模型由耦合模式决定

灵活文件布局的存储设备就是一台普通 NFS 服务器，**它没有能力在 NFS 协议里声明"我能参与紧耦合锁模型"**。因此 RFC 8435 用确定性规则来判定一台 DS 按哪种锁模型对待（第 2.3 节）：

| DS 的类型                                                             | 锁模型判定                                                                          |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| NFSv3、NFSv4.0 存储设备                                                 | **永远按松耦合对待**                                                                   |
| NFSv4.1+ 存储设备，EXCHANGE_ID 响应中**未置** `EXCHGID4_FLAG_USE_PNFS_DS` 标志 | 按松耦合对待；从锁的角度看与 NFSv4.0 存储设备**完全等同**                                            |
| NFSv4.1+ 存储设备，**置位** `EXCHGID4_FLAG_USE_PNFS_DS` 标志                | **有可能**参与紧耦合：通过后端控制协议（back-end control protocol）实现 [RFC 5661] 描述的全局 stateid 模型 |

注意第三行只是"有可能"：DS 还需要通过控制协议被 MDS **发现或通告**（"either discovered or advertised over the control protocol"），才能真正进入紧耦合模式。任何部署都可以退化为松耦合模式（"An implementation can always be deployed as a loosely coupled model"），松耦合是保底能力。

这条判定直接决定了布局中 `ffds_stateid` 字段的语义（第 5.1 节的两条铁律）：

- **松耦合**：客户端必须（MUST）用**匿名 stateid**（anonymous stateid）访问 DS。DS 不认识 MDS 的 stateid，全局 stateid 模型无从谈起。
- **紧耦合**：`ffds_stateid` 提供的是**全局 stateid**，在 DS 上同样有效，客户端直接用它做 I/O。

---

# 2. 松耦合锁模型（第 2.3.1 节）

## 2.1 基本原则：一切锁状态尽在 MDS

松耦合下，MDS 与 DS 之间只有 NFS 存储协议，没有任何控制协议。因此：

> 所有锁相关操作**全部由 MDS 处理**，stateid 由 MDS 生成，并且只关联"客户端到 MDS 的连接"（client ID）。
>
> 即使用 NFSv4 作为数据访问协议，MDS **可以**（但不必须）向 DS 发起 stateid 相关请求；产生的 stateid 也只有 MDS 和 DS 双方知道——**客户端永远拿不到 DS 生成的 stateid**。

## 2.2 各锁操作的归属

| 操作 | 松耦合下的处理位置 | 说明 |
|---|---|---|
| OPEN | MDS | stateid 由 MDS 选择，关联到客户端与 MDS 之间的 client ID。MDS 可能需要与 DS 交互来**定位要打开的文件**，但 DS 侧**不需要任何锁相关功能** |
| OPEN_DOWNGRADE / CLOSE | 仅在 MDS 本地执行 | DS 完全不参与 |
| 咨询式（advisory）字节范围锁 | MDS 本地实现 | 锁的 stateid 由 MDS 分配，且**只在 MDS 上使用** |
| 委托（delegation） | MDS 分配 | MDS 在处理冲突 OPEN 时发起召回（recall），DS 不参与 |
| TEST_STATEID / FREE_STATEID | MDS 本地处理 | DS 不参与 |

## 2.3 匿名 stateid 的两个直接后果

松耦合下，客户端对 DS 的所有 I/O 都使用**匿名 stateid**。DS 因此**完全无法分辨**某个 I/O 是代表哪个 openowner / lockowner 发出的。这带来两个结构性限制：

1. **不支持强制（mandatory）字节范围锁。**
   DS 没有锁信息，无法区分"代表锁拥有者发起的 I/O"和"其他人发起的 I/O"，强制锁无从执行。

2. **共享保留（share reservation）的检查责任落在客户端。**
   虽然 I/O 走匿名 stateid，客户端仍然必须保证自己持有与 openowner 关联的有效 stateid，并自行遵守 share reservation 语义（例如以 READ 方式打开的文件不得写入）。MDS 侧已经做过权限校验，客户端"代为执法"。

## 2.4 stateid 吊销：MDS 必须主动 fencing

如果某个 stateid 被吊销（revoke），**MDS 有责任阻止客户端继续访问**，因为 MDS 无法确定客户端是否已经知晓该 stateid 被吊销。松耦合下的手段就是第 2.2 节的 fencing：修改 DS 上数据文件的合成 uid/gid，使旧凭据失效。

## 2.5 租约模型：DS 上没有客户端租约

- 客户端从不持有 DS 生成的 stateid，因此**即使数据协议是 NFSv4，DS 上也不存在客户端租约，不存在租约过期**的问题。
- 客户端只在 **MDS 上有租约**。
- 当 MDS 侧发生租约过期时，MDS 可能需要通过 **fencing** 来阻止"尚不知晓 stateid 已被吊销"的客户端继续依赖这些 stateid 发 I/O。

**一句话概括松耦合锁模型**：DS 是一个"哑"的存储后端，锁的全部语义都被压缩到 MDS 单点；客户端与 DS 之间只有匿名 I/O，fencing 是唯一的事后吊销手段，能力上损失强制锁，责任上把 share reservation 推给了客户端。

---

# 3. 紧耦合锁模型（第 2.3.2 节）

## 3.1 基本原则：MDS 生成 stateid，控制协议负责同步

> 锁操作仍**主要由 MDS 处理**，由 MDS 生成相应的 stateid；但这些 stateid **必须通过控制协议的机制让 DS 知晓**（具体机制超出 RFC 8435 范围）。

这是与松耦合的本质区别：**stateid 是全局的**，客户端持有并使用的 stateid 在 DS 上同样有效，DS 能理解 stateid 背后的 openowner / lockowner / 委托关系。

## 3.2 各锁操作的归属

| 操作 | 紧耦合下的处理 | DS 侧需要的控制能力 |
|---|---|---|
| OPEN | 主要在 MDS。MDS 需与 DS 交互定位文件，并让 DS 知晓"MDS 选定的 stateid ↔ client + openowner"的关联 | 建立 stateid 与 client/openowner 的关联 |
| OPEN_DOWNGRADE / CLOSE | 先在 MDS 执行，**状态变更必须传播到 DS** | 同步 open 状态变更 |
| 咨询式字节范围锁 | MDS 本地实现；因 I/O 可携带锁 stateid，MDS 需让 DS 知晓"锁 stateid ↔ 对应 open stateid"的关联 | 建立锁 stateid 与 open stateid 的关联 |
| **强制字节范围锁** | **可以支持**（要求 MDS 与 DS 双方都具备相应能力） | 建立 stateid ↔ client + openowner + lock（lockowner、字节范围、锁类型）的完整关联 |
| 委托 | MDS 分配并发起召回；因 I/O 可携带委托 stateid | ① 建立 stateid ↔ filehandle + 委托类型的关联；② **解除**该关联 |
| TEST_STATEID | MDS 本地处理 | 不需要 DS 参与 |
| FREE_STATEID | MDS 处理，但**必须能把该请求传播到相应的 DS** | 接收 FREE_STATEID 传播 |

## 3.3 强制锁的关键细节：传播范围是"所有"DS

强制字节锁要求 DS 能执行锁检查，因此 MDS 必须把"stateid 与 client、openowner、lock（lockowner、字节区间、锁类型）的关联"通知 DS。RFC 特别强调：

> 因为 I/O 操作**允许携带锁 stateid**，这些信息必须传播到**所有可能承接该文件 I/O 的 DS**，而不仅仅是包含被锁字节区域的那台 DS。

原因：文件的布局可能把 I/O 分散到多台 DS 上，任何一台都可能收到携带该锁 stateid 的 I/O。

## 3.4 租约模型：DS 上也有客户端租约

紧耦合下客户端持有在 DS 上有效的 stateid，因此：

- **DS 上存在客户端租约，租约过期是有可能发生的。**
- DS 的**最佳做法是把这些锁当作"礼节性保留"（courtesy locks）继续持有**；如果 DS 不这样做，控制协议就必须提供**在 MDS 与 DS 之间同步锁状态**的手段。
- 客户端在 MDS 上同样有租约、同样会过期。MDS 处理租约过期时，应当使用控制协议在 DS 上**作废（invalidate）已被吊销的 stateid**。
- 如果客户端不响应，MDS 可能仍需退回到 **fencing** 来阻止被吊销的 stateid 在 DS 上继续生效。

## 3.5 fencing 方式：作废 stateid 而不是改属主

紧耦合下 DS 上数据文件的属主/权限/ACL 与 MDS 上的元数据文件**保持一致**，客户端访问 DS 要走与访问 MDS **相同的认证与授权流程**。因此 fencing 不再依赖合成 uid/gid，而是控制协议的职责（第 2.2 节）：

> 紧耦合锁模型的实现需要提供一种方式，通过**在 DS 上作废相应 stateid** 来阻止特定客户端访问特定文件。此时客户端收到的错误是 **NFS4ERR_BAD_STATEID**（而不是松耦合下的 NFS4ERR_ACCESS）。

**一句话概括紧耦合锁模型**：stateid 全局化，DS 真正参与锁语义（包括强制锁和委托），fencing 精确到 stateid 粒度；代价是必须实现一套控制协议来同步全部锁状态，并且要解决 DS 侧租约过期的问题。

---

# 4. 两种锁模型对比

| 维度 | 松耦合（2.3.1） | 紧耦合（2.3.2） |
|---|---|---|
| DS 角色 | 哑存储，无锁语义 | 参与锁语义，能识别 stateid |
| I/O 使用的 stateid | 匿名 stateid | 全局 stateid（`ffds_stateid`） |
| 锁状态存放 | 仅 MDS | MDS + 所有相关 DS（经控制协议同步） |
| 强制字节锁 | **不支持** | 支持（双方都要具备能力） |
| 咨询式字节锁 | MDS 本地 | MDS 本地，但锁 stateid 关联要同步给 DS |
| 委托 | MDS 全权负责 | MDS 分配，委托 stateid 关联要同步/解除 |
| FREE_STATEID | MDS 本地 | 必须传播到 DS |
| share reservation | **客户端负责强制执行** | 由 DS 按全局 stateid 正常执行 |
| DS 上的客户端租约 | 不存在 | 存在；最佳实践为礼节性保留，否则靠控制协议同步 |
| 租约过期后的兜底 | MDS fencing（合成 uid/gid） | 控制协议作废 stateid；客户端不响应时再 fencing |
| fencing 的错误信号 | NFS4ERR_ACCESS | NFS4ERR_BAD_STATEID |
| 对 DS 的要求 | 任意 NFSv3/v4.0/v4.1+ | NFSv4.1+ 且置 `EXCHGID4_FLAG_USE_PNFS_DS` + 控制协议 |
| 安全机制上限 | 不支持 RPCSEC_GSS | 可与 RPCSEC_GSS/Kerberos 配合 |

---

# 5. 布局召回（第 13 节）：锁模型状态变更的入口

第 12.5.5 节（[RFC 5661]）给出了与布局类型无关的召回理由，RFC 8435 补充了灵活文件布局 MDS **应当召回（recall）在途布局**的四种场景：

1. **文件的安全策略变更**——设置了 ACL 或权限 mode bits；
2. **文件的布局变更**——使已发放的布局失效（例如条带策略调整）；
3. **现有布局与锁约束的执行不一致**——即锁状态的变化要求重新收敛客户端 I/O 路径；
4. **现有布局与重新同步（resilvering，见 8.3 节）的要求不一致**——例如镜像副本需要重建时，必须先收回客户端写权限。

## CB_RECALL_ANY（第 13.1 节）

MDS 用回调操作 **CB_RECALL_ANY** 通知客户端归还部分或全部布局，典型用途是 MDS 需要**回收资源**时：

```c
const RCA4_TYPE_MASK_FF_LAYOUT_MIN = 16;
const RCA4_TYPE_MASK_FF_LAYOUT_MAX = 17;

struct CB_RECALL_ANY4args {
    uint32_t craa_layouts_to_keep;   /* 允许客户端保留的灵活文件布局数量上限 */
    bitmap4  craa_type_mask;         /* 召回对象的类型掩码 */
};

enum ff_cb_recall_any_mask {
    PNFS_FF_RCA4_TYPE_MASK_READ = 16,  /* 召回 iomode=LAYOUTIOMODE4_READ 的布局 */
    PNFS_FF_RCA4_TYPE_MASK_RW   = 17   /* 召回 iomode=LAYOUTIOMODE4_RW 的布局 */
};
```

- `craa_type_mask` 指定召回哪类资源；两个标志按 **iomode** 区分，两者都置位表示两种 iomode 的布局都要召回。
- 客户端的响应策略（SHOULD）：归还该 iomode 下自己**最不需要**的布局，保留的灵活文件布局数量**不超过** `craa_layouts_to_keep`。

---

# 6. 客户端 fencing（第 14 节）

当客户端**失去通信且租约已过期**，或**在一个租约周期内未归还已被召回的布局**时，MDS 可以（MAY）**吊销（revoke）这些布局**并把资源重新分配给其他客户端（依据 [RFC 5661] 第 12.5.5 节）。

关键强制项：

> 为避免数据损坏，MDS **必须（MUST）** 按照第 2.2 节描述的方式对被吊销的客户端执行 **fencing**，将其与相应数据文件隔离开。

即：吊销布局只是"宣布作废"，fencing 才是"确保 DS 侧真正拒绝 I/O"的落地动作。松耦合靠改合成 uid/gid（客户端见 NFS4ERR_ACCESS），紧耦合靠作废 DS 上的 stateid（客户端见 NFS4ERR_BAD_STATEID）。

---

# 7. 与安全机制的关系（第 15.1 节）

锁模型与安全机制的选择强相关，因为合成 uid/gid 与全局 stateid 是两种完全不同的"凭据形态"：

## 7.1 松耦合：明确不支持 RPCSEC_GSS

- RPCSEC_GSSv3（[RFC 7861]）理论上提供了"MDS 代表客户端向 DS 授权"的能力（RPC 应用自定义的结构化特权断言，见 [RFC 7862] 第 4.9.1 节），但要求 MDS、DS、客户端三方都实现 RPCSEC_GSSv3。
- RFC 8435 明确放弃这条路：这会要求**种类繁多的存储设备做大量实现工作**，短期内不可能形成广泛可用的规范。
- 结论：**本布局类型不支持"松耦合 + RPCSEC_GSS"组合**，未来可以由新的布局类型（借助 RPCSEC_GSSv3 或其他方式）来补足。这也再次印证：松耦合世界就是 AUTH_SYS + 合成 uid/gid 的安全水位。

## 7.2 紧耦合：与 RPCSEC_GSS 无冲突

- 访问元数据文件与访问数据文件使用**完全相同的 principal**；DS 可以通过控制协议校验任意 RPC 凭据。
- 因此紧耦合下使用 Kerberos V5 GSS-API（[RFC 4121]）等机制没有安全问题。

---

# 8. 小结

| 问题 | 松耦合 | 紧耦合 |
|---|---|---|
| 锁在哪里 | 全部在 MDS | MDS 为主，DS 经控制协议同步 |
| 客户端拿什么访问 DS | 匿名 stateid + 合成 uid/gid | 全局 stateid + 真实 principal |
| 能力损失 | 无强制锁；share reservation 靠客户端 | 无损失（可强制锁、可委托检查） |
| 状态一致性代价 | 无（DS 无状态） | 高（全部锁状态需同步、租约需协调） |
| fencing | 改合成 uid/gid（粗粒度，NFS4ERR_ACCESS） | 作废 stateid（精确，NFS4ERR_BAD_STATEID） |
| 安全上限 | AUTH_SYS | 可与 Kerberos/RPCSEC_GSS 配合 |

设计取向非常清晰：**松耦合用"功能的阉割"换"部署的极简"**——任意 NFS 服务器都能当 DS，但锁语义退化为 MDS 单点、I/O 匿名化、fencing 只能大刀阔斧；**紧耦合用"控制协议的复杂度"换"完整的锁语义与安全性"**——DS 必须理解 stateid，必须同步锁状态，但换来强制锁、精确 fencing 和强安全机制。

---

# 参考资料

- [RFC 8435](https://www.rfc-editor.org/rfc/rfc8435.txt)：Parallel NFS (pNFS) Flexible File Layout（2018-08），第 2.2、2.3、5.1、13、14、15.1 节。
- [RFC 5661](https://www.rfc-editor.org/rfc/rfc5661.txt)：NFSv4.1（pNFS 基础、stateid/租约模型、第 12.5.5 节布局吊销）。
- [RFC 8434](https://www.rfc-editor.org/rfc/rfc8434.txt)：pNFS 实现的问题澄清（耦合模型、控制协议需求）。
- [RFC 7861](https://www.rfc-editor.org/rfc/rfc7861.txt)：RPCSEC_GSSv3。
- [RFC 7862](https://www.rfc-editor.org/rfc/rfc7862.txt)：NFSv4.2（第 4.9.1 节特权断言）。
- [RFC 4121](https://www.rfc-editor.org/rfc/rfc4121.txt)：Kerberos V5 GSS-API。
