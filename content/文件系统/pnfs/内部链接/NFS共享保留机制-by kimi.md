> 本文详细解读 NFSv4 的共享保留机制（Share Reservation）：设计动机、OPEN/OPEN_DOWNGRADE/CLOSE 全生命周期语义、冲突判定规则、I/O 阶段的强制执行、与委托及字节范围锁的关系，以及 pNFS 场景下的实现差异。主要依据 [RFC 5661](https://www.rfc-editor.org/rfc/rfc5661.txt)（NFSv4.1）第 9、16.11、18.18 节及第 15.1 节错误码定义，语义与 RFC 7530（NFSv4.0）、RFC 8881（NFSv4.1 修订版）一致。
>
> 前置概念：
> - **OPEN / open-owner**：NFSv4 的原子"打开"操作，每次 OPEN 归属一个 open-owner（由 client ID + owner 标识）。
> - **stateid**：OPEN 成功后服务器返回的状态标识，后续 READ/WRITE/LOCK 携带它表明 I/O 归属哪个打开状态。
> - **字节范围锁（byte-range lock）**：LOCK/LOCKT/LOCKU 管理的区间锁，与共享保留是**两套相互独立**的机制。
> - **委托（delegation）**：服务器把某文件的打开/读写作弊权下放给客户端的缓存授权。

---

# 1. 机制定位与设计动机

共享保留是 NFSv4 为支持 **Win32 共享语义**（Win32 share reservations）而引入的机制，NFSv3 中不存在（[RFC 5661] §9 开头）：

> 为了支持 Win32 共享保留，必须提供**原子地打开或创建文件**的操作。单独的 share/unshare 操作无法正确实现 Win32 的 OpenFile API。为正确实现共享语义，NFS 早期版本在打开/创建文件时使用的机制（LOOKUP、CREATE、ACCESS 相互分离）必须被取代。因此 NFSv4.1 定义了 OPEN 操作，能在一个操作中原子地完成 lookup、create 和 lock。

要点：

- **动机来自 Windows/CIFS 世界**：Windows 的 CreateFile API 允许调用者声明"我要以什么方式访问文件"以及"我拒绝别人以什么方式访问"，即 share mode。NFSv3 只有 LOOKUP+ACCESS+CREATE 的分离流程，无法原子地表达和检查这种互斥关系。
- **NFSv4 的解法**：把"查找文件、可能创建文件、声明访问与拒绝模式"合并进一个 OPEN 操作，在 OPEN 时原子地完成冲突检查。任何 OPEN 本身就构成一份共享保留。
- POSIX/UNIX 的 open(2) 没有 deny 概念，因此共享保留对 UNIX 客户端是可选使用的（固定用 DENY_NONE，见第 8 节）。

---

# 2. 基本语义：share_access 与 share_deny（§9.7）

## 2.1 定义

共享保留是**控制文件访问的机制，与字节范围锁相互独立**。客户端打开文件时，在 OPEN 参数中声明两个位掩码字段：

- **share_access**：本次打开要求的访问类型——`READ`、`WRITE` 或 `BOTH`；
- **share_deny**：本次打开**拒绝他人**使用的访问类型——`NONE`、`READ`、`WRITE` 或 `BOTH`。

常量定义（[RFC 5661] §9.7）：

```c
const OPEN4_SHARE_ACCESS_READ   = 0x00000001;
const OPEN4_SHARE_ACCESS_WRITE  = 0x00000002;
const OPEN4_SHARE_ACCESS_BOTH   = 0x00000003;

const OPEN4_SHARE_DENY_NONE     = 0x00000000;
const OPEN4_SHARE_DENY_READ     = 0x00000001;
const OPEN4_SHARE_DENY_WRITE    = 0x00000002;
const OPEN4_SHARE_DENY_BOTH     = 0x00000003;
```

（NFSv4.2 起 `share_access` 还包含 `OPEN4_SHARE_ACCESS_WANT_*` 委托提示位，校验时需用 `~OPEN4_SHARE_ACCESS_WANT_DELEG_MASK` 屏蔽，见 [RFC 5661] §16.11、§18.18。）

OPEN 失败（包括共享冲突）则客户端必须让应用程序的 open 请求失败。

## 2.2 冲突判定规则

RFC 以伪代码形式给出了精确语义（§9.7）：

```c
if (request.access == 0) {
    return (NFS4ERR_INVAL);
} else {
    if ((request.access & file_state.deny) ||
        (request.deny & file_state.access)) {
        return (NFS4ERR_SHARE_DENIED);
    }
}
return (NFS4ERR_OK);
```

两个判据：

| 判据 | 含义 | 例子 |
|---|---|---|
| `request.access & file_state.deny ≠ 0` | 你要的访问，被某个现存 OPEN 拒绝 | A 以 WRITE 打开但 deny WRITE；B 再请求 WRITE/BOTH → B 被拒 |
| `request.deny & file_state.access ≠ 0` | 你要拒绝的访问，正被某个现存 OPEN 使用 | A 以 READ 打开（DENY_NONE）；B 请求 READ + deny READ → B 被自己与 A 的冲突拒绝 |

三个关键细节：

1. **file_state 是全集**：参与计算的 `file_state.access/deny` 反映该文件**所有**当前 OPEN——**包括发起本次请求的 open-owner 自己的其他 OPEN**。即同一进程两次打开同一文件也要遵守冲突规则（服务器通过升级机制合并，见第 3 节，但合并前仍受约束）。
2. **冲突只发生在 OPEN 时刻**：检查针对新 OPEN 请求与现存打开状态的组合，结果只有两种——放行（返回 open stateid）或 `NFS4ERR_SHARE_DENIED`（错误码 10015，§15.1.8.9）。
3. **参数合法性先行**：`share_access`（屏蔽 WANT 位后）必须是 READ/WRITE/BOTH 之一，`share_deny` 必须是四个合法值之一，否则服务器必须返回 `NFS4ERR_INVAL`（§16.11）。

## 2.3 冲突矩阵速查

新 OPEN 的 access 与现存 OPEN 的 deny 组合（"×"= NFS4ERR_SHARE_DENIED）：

| 新 access ↓ / 现存 deny → | NONE | READ | WRITE | BOTH |
|---|---|---|---|---|
| READ | ✓ | × | ✓ | × |
| WRITE | ✓ | ✓ | × | × |
| BOTH | ✓ | × | × | × |

新 OPEN 的 deny 与现存 OPEN 的 access 组合：

| 新 deny ↓ / 现存 access → | READ | WRITE | BOTH |
|---|---|---|---|
| NONE | ✓ | ✓ | ✓ |
| READ | × | ✓ | × |
| WRITE | ✓ | × | × |
| BOTH | × | × | × |

---

# 3. 生命周期：建立、升级、降级、释放（§9.8–9.10）

## 3.1 建立：OPEN

- OPEN 是获取文件句柄的**必经路径**：为了获得正确的共享语义，客户端必须经 OPEN 拿到普通文件的 filehandle 并声明 access/deny（§9.8）。编程接口没有 deny 概念的客户端（如 UNIX）应一律请求 `OPEN4_SHARE_DENY_NONE`。
- OPEN 带 CREATE 标志时还**原子地**兼有 v3 的 CREATE 语义（"create with a share"）。
- OPEN 成功后，服务器返回 open stateid；该 stateid 的 access mode 由本次 OPEN 的 share_access 决定（§9.1.2）。

## 3.2 升级：同一 open-owner 重复 OPEN（§9.9）

当同一 ==🔴open-owner== 再次 OPEN 同一文件时，结果是**升级（upgrade）而非新建：

- 服务器把该文件对该 open-owner 的打开状态更新为"**已有 access/deny 位与新请求位的并集**"；
- 协议上视为**一个打开**，由**单个 stateid** 表示（"other" 值与原 open 相同，"seqid" 递增——即使 upgrade 实际没有改变模式也要递增）；
- 最终只需一次 CLOSE 即可释放全部效果。

注意：若服务器对同一文件对象导出多个 filehandle，两次 OPEN 落到不同 filehandle 时，**不得**合并 access/deny 位，必须分别维护独立的 OPEN 和 stateid、分别 CLOSE。

## 3.3 降级：OPEN_DOWNGRADE（§9.9、§18.18）

客户端本地关闭了合并打开的其中一个文件时，剩余打开所需的 access/deny 位可能缩小，此时必须用 **OPEN_DOWNGRADE** 通知服务器收缩：

- 新的 `share_access` / `share_deny` **应（SHOULD）等于**当前仍生效的若干 OPEN 的对应位并集的某个子集；不满足时服务器**应**返回 `NFS4ERR_INVAL`（§18.18）；
- 由于新位是已授权位的子集，OPEN_DOWNGRADE **不可能因共享保留冲突而被拒绝**；
- 降级后 stateid 的 "other" 不变、"seqid" **必须递增**；
- 副作用：OPEN_DOWNGRADE 可能使原来不可授予的 `OPEN_DELEGATE_READ` 委托变为可授予，服务器可立即响应挂起的委托请求（§18.18.4）。

## 3.4 释放：CLOSE

- **CLOSE 移除该 open-owner 在此文件上的全部共享保留**（§9.8）；
- 若还持有字节范围锁，客户端应先释放全部锁再 CLOSE；若 CLOSE 后仍有锁存在，服务器必须返回 `NFS4ERR_LOCKS_HELD`（部分服务器不支持带锁 CLOSE）。

---

# 4. I/O 阶段的强制执行（§9.1.2）

共享保留与咨询式字节锁的本质区别在于：**它是强制的（mandatory）**。

> 共享保留由 OPEN 建立，其性质是强制的：当 OPEN 拒绝了 READ 或 WRITE 操作时，这些操作将被以 NFS4ERR_LOCKED 错误拒绝。字节范围锁则可由服务器实现为强制的或咨询的……咨询锁只阻止冲突的加锁请求，不影响 READ/WRITE。

具体分两层：

## 4.1 OPEN 时刻：准入检查（见第 2 节）

冲突 → `NFS4ERR_SHARE_DENIED`，根本拿不到打开状态。

## 4.2 I/O 时刻：stateid 关联的 access mode 检查与 deny 强制

每个有效 stateid（除特殊 stateid 外）都定义一个文件的 access mode（`READ` / `WRITE` / `BOTH`，可被后续 OPEN、OPEN_DOWNGRADE 修改，§9.1.2）：

- **WRITE 类操作**（WRITE、SETATTR 改 size——SETATTR 改大小被视同写被截断/新增的区域）必须验证 access mode 允许写，否则必须返回 `NFS4ERR_OPENMODE`（如向只读打开的文件写入）；
- **READ** 时服务器**可以**做对应检查，**也可以**放宽——允许在 WRITE-only 打开的 open 上读（照顾那些因缓冲区缓存约束写路径上不得不读的客户端实现）；**但即便放宽，仍必须检查与本次 READ 冲突的 share reservation**（例如别的 OPEN 声明了 `OPEN4_SHARE_DENY_READ` / `DENY_BOTH`）；反过来，严格做 access mode 检查的服务器无需再单独检查 share reservation——因为存在 READ 打开本身就保证了不存在 DENY_READ；
- **与现存 deny 模式冲突的 I/O 一律拒绝**：无论请求带什么 stateid，只要文件上存在与这次 I/O 冲突的 share reservation（或强制字节锁），服务器**必须**拒绝该 READ/WRITE，错误为 **`NFS4ERR_LOCKED`**（§9.1.2、§15.1.8.4）。这就是"OPEN 的 deny 声明在 I/O 路径上仍然有效"的落点。

## 4.3 特殊 stateid 下的行为

| stateid | 语义 | 与共享保留的关系 |
|---|---|---|
| 服务器返回的 open/lock/delegation stateid | 正常路径，携带 I/O | 受 access mode 检查与 deny 强制约束 |
| 匿名 stateid（全 0） | 客户端未在服务器建立任何锁状态（无打开、无锁、无委托）时使用 | 服务器**可以**服务 READ/WRITE，**但必须受强制字节锁和当前 share deny 模式约束**——存在冲突 deny 时必须拒绝（§9.1.2、§18.22、§18.32） |
| READ bypass（全 1） | 请求绕过锁检查 | READ 时服务器**可以**允许绕过；**WRITE 用全 1 stateid 不得绕过**任何锁检查，按匿名 stateid 同等处理（§9.1.2、§18.32） |

经 LOOKUP（而非 OPEN）拿到 filehandle 的客户端没有有效 stateid，服务器按"最小访问"对待：例如别人以 `DENY_BOTH` 打开时，它只能用 READ bypass stateid 读，**完全无法写**（§9.8）。

---

# 5. 与委托（Delegation）的交互（§16.11、§9.8）

共享保留的检查与委托的冲突检测交织在一起：

- 若被打开的文件上存在与本次 OPEN（有时取决于 share_access/share_deny 取值）冲突的其他客户端委托，**必须先召回该委托**，操作要委托归还或被吊销后才能继续；召回未完成的期间对客户端返回 `NFS4ERR_DELAY`（§16.11）；
- 冲突规则：`OPEN_DELEGATE_WRITE` 委托与**任何**其他客户端的 OPEN 冲突；`OPEN_DELEGATE_READ` 委托只与特定特征的 OPEN 冲突；
- 持有委托的客户端发 I/O 时应携带委托 stateid（或其派生的 open/lock stateid），避免服务器因无法识别委托而**不必要地召回**委托（§9.1.2）；
- 委托存在时服务器不掌握该客户端真实的 OPEN 状态，因此 REMOVE 等操作需要"是否存在阻止删除的打开"的判定时，往往要先把委托召回才能确定（§18.22 附近 REMOVE 语义）。

---

# 6. 服务器故障与恢复期（grace period）中的处理（§8.2 附近）

共享保留是有状态的，服务器重启后进入恢复期（grace period）。RFC 允许一种优化：

> 如果服务器在稳定存储上维护了"是否存在强制锁"的摘要信息——包括强制字节锁和**声明了 deny 模式的共享保留**——许多请求在恢复期就可以被放行。已知不存在这类共享保留时，**不声明 deny 模式的 OPEN 可以安全授予**；进一步已知不存在强制字节锁时，READ/WRITE 也可安全处理。

即：**带 deny 模式的打开是服务器恢复期的障碍**——服务器必须确保恢复期间新授予的打开不会与尚待恢复的 deny 语义冲突，除非它把相关信息持久化了。

---

# 7. 与字节范围锁、ACL 的关系与边界

| 维度 | 共享保留 | 字节范围锁 |
|---|---|---|
| 作用对象 | 整个文件的"打开方式" | 文件内字节区间 |
| 建立方式 | 隐含于每次 OPEN（share_access/share_deny 参数） | LOCK 操作显式建立 |
| 冲突时机 | OPEN 时检查 + I/O 时强制 | LOCK 时检查；I/O 时仅在强制模式下检查 |
| 强制力 | **强制性**：冲突 I/O 被拒绝（NFS4ERR_LOCKED） | 咨询式（服务器可选强制，如 UNIX mandatory lock bit） |
| 状态归属 | open-owner（OPEN stateid） | lock-owner + 所属 open-owner（LOCK stateid） |

另外两点边界：

- **access mode 与 ACL 无关但互相补充**：OPEN 时服务器按 share_access 检查请求者对文件的访问权（一般是 ACL 判定结果），无权则 `NFS4ERR_ACCESS`；但客户端不应越俎代庖地预判服务器决定（权限可能变化、可能有 ACL 框架之外的管理控制）（§16.11）；
- **凭据一致性**（§16.11）：若 EXCHANGE_ID 未协商 `EXCHGID4_FLAG_BIND_PRINC_STATEID`，服务器不得要求 I/O 与 OPEN 使用相同凭据，且必须对 READ/WRITE 本身做访问检查；若协商了该能力，则 I/O 与 OPEN 的凭据及 stateid 必须一致（SP4_SSV/SP4_MACH_CRED 保护下有例外）。

---

# 8. 各实现的行为差异

## 8.1 Windows vs UNIX 客户端

- **Windows/CIFS 客户端**：有天然的 share mode 概念（deny read/deny write/deny both），经 NFSv4 打开文件时直接映射到 share_deny；
- **UNIX/POSIX 客户端**：open(2) 无 deny 参数，按 RFC 建议固定使用 `OPEN4_SHARE_DENY_NONE`（§16.11："For clients that do not directly support SHAREs (i.e., UNIX), the expected deny value is OPEN4_SHARE_DENY_NONE"）。Linux NFSv4 客户端即属此类，从不发起带 deny 模式的 OPEN——这意味着纯 Linux 环境中共享保留机制实际退化为" access mode 检查 + 委托冲突管理"。

## 8.2 pNFS 文件布局 vs 灵活文件布局

- **文件布局（files layout，[RFC 5661] §13.9.2.2）**：DS 参与 open/deny 语义——"Open and deny mode validation **必须**对照 DS 持有的 open/deny 模式执行。当访问收缩或 deny 变严（CLOSE 或 OPEN_DOWNGRADE 引起）时，DS 必须阻止任何在 MDS 上会被拒绝的 I/O；当访问扩展时，DS 必须确保不再因已失效的 open/deny 问题拒绝请求。"这要求 DS 与 MDS 之间存在控制协议同步打开状态（紧耦合）；
- **灵活文件布局（flexible files，[RFC 8435] §2.3.1）**：松耦合下所有 I/O 使用匿名 stateid，DS 无法分辨 openowner，"**共享保留的执行责任在客户端**"——服务器端不做 I/O 路径的 deny 强制。这是灵活文件布局松耦合模型能力折损的一部分（与不支持强制字节锁同理）。

---

# 9. 相关错误码速查（[RFC 5661] §15.1）

| 错误码 | 场景 |
|---|---|
| `NFS4ERR_SHARE_DENIED` (10015) | OPEN 的共享保留冲突（§15.1.8.9） |
| `NFS4ERR_LOCKED` (10012) | READ/WRITE 与现存锁冲突：包括与 share reservation 不一致、命中强制字节锁（§15.1.8.4） |
| `NFS4ERR_OPENMODE` (10038) | I/O 操作不被 stateid 的 access mode 许可，如写只读打开的文件（§15.1.8.8） |
| `NFS4ERR_INVAL` | share_access/share_deny 取值非法，或 OPEN_DOWNGRADE 未遵守子集约束（§16.11、§18.18） |
| `NFS4ERR_ACCESS` | 请求者对文件的访问权限不足（ACL 判定），与共享保留相互独立（§16.11） |
| `NFS4ERR_DELAY` | OPEN 等待冲突委托被召回（§16.11） |
| `NFS4ERR_LOCKS_HELD` | CLOSE 时仍存在字节范围锁（§9.8） |

---

# 10. 小结

共享保留是 NFSv4 为复刻 Win32 share mode 而设计的**强制性打开互斥机制**，核心要点：

1. **一次 OPEN 即一份共享保留**：share_access 声明我要什么，share_deny 声明我拒绝别人什么；
2. **两级检查**：OPEN 时刻按 `(request.access & file_state.deny) || (request.deny & file_state.access)` 判定，冲突即 `NFS4ERR_SHARE_DENIED`；I/O 时刻对现存 deny 模式强制执行，冲突 I/O 被拒绝并返回 `NFS4ERR_LOCKED`——这是它区别于咨询式字节锁的根本特征；
3. **生命周期由三个操作闭环**：OPEN 建立（同 open-owner 重复 OPEN 自动按位并集升级）、OPEN_DOWNGRADE 收缩、CLOSE 一次性释放；
4. **stateid 是强制执行的载体**：open stateid 隐含 access mode，WRITE 类操作受 `NFS4ERR_OPENMODE` 约束；匿名/READ bypass 特殊 stateid 只能绕开部分检查，绕不过 deny 模式；
5. **使用与否是客户端的文化属性**：Windows 客户端真正使用 deny 模式，UNIX/Linux 客户端恒为 `DENY_NONE`，此时机制退化为访问模式与委托冲突管理；
6. **pNFS 下能否在 DS 强制执行取决于耦合模式**：文件布局由 DS 校验 open/deny（需控制协议），灵活文件布局松耦合下责任回到客户端。

---

# 参考资料

- [RFC 5661](https://www.rfc-editor.org/rfc/rfc5661.txt)：Network File System (NFS) Version 4.1（2010-01）。第 9 节（File Locking and Share Reservations，含 §9.7 Share Reservations、§9.8 OPEN/CLOSE Operations、§9.9 Open Upgrade and Downgrade、§9.1.2 Use of the Stateid and Locking）；§16.11 OPEN；§18.18 OPEN_DOWNGRADE；§15.1.8 错误码。
- [RFC 8881](https://www.rfc-editor.org/rfc/rfc8881.txt)：Network File System (NFS) Version 4.1（2020-08，5661 的修订版），共享保留语义相同。
- [RFC 7530](https://www.rfc-editor.org/rfc/rfc7530.txt)：NFSv4.0（共享保留机制的最初形态，§9.7 等）。
- [RFC 8435](https://www.rfc-editor.org/rfc/rfc8435.txt)：pNFS 灵活文件布局，§2.3.1 松耦合锁模型中 share reservation 由客户端强制。
- [RFC 7862](https://www.rfc-editor.org/rfc/rfc7862.txt)：NFSv4.2，`OPEN4_SHARE_ACCESS_WANT_*` 委托提示位。
