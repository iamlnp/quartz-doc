# 1 rfc8435
## 1.1 灵活文件布局的合成uid/gid - by kimi
### 1.1.1 它要解决什么问题
pNFS 把元数据和数据分离：MDS（metadata server）管元数据和布局，DS（storage device）存数据文件。MDS 和 DS 之间有两种耦合模式：

• 紧耦合（tight coupling）：有专门的控制协议，可以传递全局 stateid，fencing（吊销客户端对数据文件的访问权）由控制协议负责，DS 上文件的属主/权限与 MDS 一致。
• 松耦合（loose coupling）：MDS 和 DS 之间唯一的控制协议就是 NFS 本身。这时有两个致命限制：
   1. 没有全局 stateid 模型——DS 不知道 MDS 发的 stateid 是什么，客户端只能拿匿名 stateid 访问 DS；
   2. MDS 无法让 DS"拒绝对某个客户端的 I/O"——协议里没有"撤销 credential"的语义。

既然不能让 DS 拒绝凭据，RFC 8435 就反其道而行：让凭据本身失效。这就是合成 uid/gid 方案——一套建立在 DS 本地文件权限位（mode bits）之上的、纯 NFS 就能实现的 fencing机制。

**核心设定**（第 2.2 节）：
• MDS 以 root（超级用户）身份在 DS 上拥有一切权限，即"MDS 对 DS 上的数据文件系统有完全访问权，然后用合成 id 来控制客户端对单个文件的访问"。
• 每个数据文件在 DS 上被设上一个与真实用户无关的、临时分配的"合成" owner uid 和 gid：
   • 属主（合成 uid）→ 读写权限
   • 属组（合成 gid）→ 只读权限
   • 其他用户 → 无任何权限
### 1.1.2 工作流程
一次授权访问大致是这样的（2.2 节 + 2.2.1 节）：
1. MDS 授权：客户端对某文件请求布局（LAYOUTGET）时，MDS 先通过 SETATTR 在 DS 上把该数据文件的 owner/group 改成选定的合成 uid/gid（可以从预留的 id 池里 round-robin选取），权限位设为 -rw-r-----。
2. 下发凭据：MDS 把这两个值通过布局中的 ffds_user 和 ffds_group 字段（见 ff_data_server4，XDR 类型分别是 fattr4_owner / fattr4_owner_group，即 utf8str_mixed 字符串）发给客户端。
3. 客户端访问 DS：客户端把这两个值作为 RPC credential（AUTH_SYS 的 uid/gid）发给 DS。DS 按普通 POSIX 权限检查：
   • credential 里的 uid == 合成 uid → 读写；
   • uid 不匹配但 gid == 合成 gid → 只读；
   • 都不匹配 → NFS4ERR_ACCESS。
4. 收回：客户端用完、且 MDS 确认无其他客户端在访问该文件时，MDS 把 owner/group 重置回一个"受限 id"（restricted id，即永远不会被授权访问的 id），文件回到禁止访问状态。

注意整个过程中合成 uid/gid 与发起访问的真实用户（如 loghyr、garbo）毫无关系。真实用户的权限检查只在 LAYOUTGET 那一刻由 MDS 完成（MDS 验证该用户对元数据文件的OPEN/ACCESS 权限，不满足直接拒绝发布局）；布局一旦到手，客户端对所有本地用户一律使用同一组合成凭据访问 DS。也就是说，布局和其中的合成 uid/gid 构成了一种"能力"（capability）：filehandle + 匿名 stateid + 合成 uid/gid。

只读布局（iomode = LAYOUTIOMODE4_READ）有个特殊处理：MDS 应返回一个在 DS 上根本不存在的合成 uid，只有 gid 有效，从而即使 credential 到了客户端手里也只能走 group 读
权限（2.2.2 节）。
### 1.1.3 Fencing：靠"改属主"来吊销凭据
fencing 时 MDS 不需要联系客户端，只需要在 DS 上对该数据文件做一次 SETATTR，把合成 uid/gid 改成另一个值（2.2 节）：
   ```
 -rw-r-----    1 19453   28419    1697 Dec  4 11:31 data_ompha.c   ← fencing 后
   ```
 旧凭据立即失效：还拿着旧 uid/gid 发 I/O 的客户端会收到 NFS4ERR_ACCESS，随后它应当归还布局、重新 LAYOUTGET，MDS 此时可以选择发新布局（含新凭据）、不发布局（强制 I/O走 MDS 转发）、或拒绝进一步访问。客户端不需要知道 MDS/DS 之间是松耦合还是紧耦合，只需统一处理这类错误。
 
**三个重要推论：**
1. **粒度是全有或全无**：松耦合模型下 MDS 无法只 fence 一个客户端——改属主会把所有持有该文件布局的客户端一起 fence 掉。这是设计上的接受项：其他（合作的）客户端被 fence后会自然地归还布局并申请新布局，MDS 顺手把新的 uid/gid 发给它们，恢复正常。
2. 新 id 必须不可预测：fencing 后的新值既不能等于该文件用过的旧值，也不能是可预测的变化（比如简单 +1），否则客户端可以猜出下一个有效凭据（2.2.2 节）。
3. 权限/ACL 变更流程：当 MDS 收到修改某文件权限或 ACL 的请求时，它应先 recall 该文件的所有布局，然后必须 fence 掉仍持有布局的客户端（使旧凭据失效），之后才能提交新权限。这样客户端再来申请布局时会针对新的访问控制元数据重新授权（第 15 节）。
### 1.1.4 RFC 2.2.2 的完整例子
用户 loghyr 在 MDS 上创建 ompha.c：
```
 -rw-r--r--    1 loghyr  staff    1697 Dec  4 11:31 ompha.c          ← MDS 上的元数据文件
 -rw-r-----    1 19452   28418    1697 Dec  4 11:31 data_ompha.c     ← DS 上的数据文件（初始：不可预测的合成 id，拒绝访问）
```
文件被打开后：
• 布局不关心是谁打开的（loghyr 还是 garbo 都一样），客户端必须出示 uid=19452 才能得到写权限；出示任何其他 uid 时，必须同时给出 gid=28418 才能得到读权限。
• 假设 fencing 后合成 id 变为 uid=1066 / gid=1067（例子换了一组数）：给 loghyr 提供写访问时，客户端发 credential 1066:1067（uid 命中属主 → 读写）；给 garbo 提供读访问时，发 1067:1067（uid 不命中属主，gid 命中属组 → 只读）。读场景下 uid 的具体值无所谓，只要不等于当前合成 uid 即可。
• 数据文件的权限位只允许合成 uid 读写、合成 gid 只读，因此知道一个文件的合成 id，推不出任何其他数据文件的凭据；加上客户端只拿到文件对象的 filehandle、拿不到目录filehandle，也无法向上猜测目录结构（第 15 节）。

### 1.1.5 实现注意事项（2.2.1 节）
• id 选取纯属实现问题。常见做法：管理员预留一段 uid 范围（比如让 LDAP 的 uidNumber 字段避开这段），从中 round-robin 选取；同时选定一个"永不授权"的受限 id，用于空闲时重置属主和 fencing 后的兜底。
• 避免使用 0。在 AUTH_SYS 模型下，uid/gid 为 0 通常有两种结局，都有害：一是被 DS 当作超级用户——文件被 fence（属主改掉）后客户端凭 uid 0 依然能访问文件，fencing 形同虚设；二是被映射为匿名 id（nobody），多个不同 id 塌缩成同一个匿名 id，破坏凭据的唯一性。
• 合成 gid 应在 DS 使用的所有名字服务中都没有映射（组里没有任何成员），保证只有 credential 中显式携带才有效，避免某个真实用户因为组成员关系意外获得访问权。
• DS 侧加固（RECOMMENDED）：DS 文件系统只允许 MDS root 访问；所有存放数据文件的目录属主设为 root；MDS↔DS 之间的通信要防窃听和中间人篡改（物理安全、加密信道等均可）

### 1.1.6 安全模型与局限
这套机制的安全等级等价于 AUTH_SYS，必须明确两点（第 2.2 节和第 15 节）：
1. 只防"合作客户端"，防不了恶意客户端。凭据是 MDS 明文发给客户端的，恶意客户端本来就可以拿它做它被授权做的事（读写该文件），或者干脆伪造 AUTH_SYS 凭据。它约束的是"拿到了布局的诚实客户端在布局被吊销后不再继续 I/O"这一行为。
2. 与 RPCSEC_GSS 不兼容。合成 uid/gid 依赖客户端在 AUTH_SYS 凭据里出示任意的 uid/gid 数值；而 Kerberos/RPCSEC_GSS 用的是 principal 身份，无法表达"以 uid 19452 的名义"这种委托，IETF 讨论中已证明松耦合模型在 RPCSEC_GSS 下不成立。所以合成 uid/gid 本质上是 AUTH_SYS 世界的解决方案，这也是为什么 ffds_user/ffds_group 虽是字符串类型（NFSv4 的 owner 属性形式），但约定"无前导零的十进制数字符串可按 NFSv3 的数值 uid/gid 解释"——用 Kerberos 时才会是 name@domain 形式（第 5.1 节）。

另外容易误解的一点：这里的合成 uid/gid 与客户端侧的 idmapping 完全是两回事。它不需要、也不应该被解析成本地用户——这些数值只在 DS 的权限检查里有意义，客户端只是原样把它塞进 RPC 凭据。

### 1.1.7 总结
合成 uid/gid 是 RFC 8435 为"MDS 和 DS 之间只有 NFS、没有控制协议"的松耦合场景设计的 fencing 机制：MDS 把数据文件的属主/属组动态改成临时分配的合成 id，通过 DS 的普通POSIX 权限位把凭据变成"一次一密"——授权就是改属主，吊销就是再改一次属主。它实现简单（纯 NFS SETATTR 即可）、无需 DS 配合任何 pNFS 语义，代价是粒度粗（一次 fence 全部客户端）、安全上限就是 AUTH_SYS。

## 1.2 [RFC8435状态与锁模型-by kimi](内部链接/RFC8435状态与锁模型-by%20kimi.md)

# 2 [NFS共享保留机制-by kimi](内部链接/NFS共享保留机制-by%20kimi.md)



