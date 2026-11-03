# 并行 NFS（pNFS）灵活文件布局（Flexible File Layout）

> **RFC 8435**（中文翻译）
>
> | 项目 | 内容 |
> | --- | --- |
> | 发布组织 | Internet Engineering Task Force（IETF） |
> | 文档类别 | 标准跟踪（Standards Track） |
> | 作者 | B. Halevy、T. Haynes（Hammerspace） |
> | ISSN | 2070-1721 |
> | 发布日期 | 2018 年 8 月 |
> | 原文链接 | <https://www.rfc-editor.org/rfc/rfc8435.txt> |

## 摘要

并行 NFS（pNFS）允许将文件的元数据（放在元数据服务器上）与数据（放在存储设备上）分离。本文档将灵活文件布局类型（flexible file layout type）定义为 pNFS 的一个扩展：它允许使用只需与元数据服务器进行有限程度交互、并使用现有协议的存储设备。本文档还加入了客户端侧镜像（client-side mirroring），以提供文件副本复制能力。

## 本备忘录的状态

这是一份互联网标准跟踪（Internet Standards Track）文档。

本文档是互联网工程任务组（IETF）的成果，代表了 IETF 社区的共识。它已经过公开评审，并由互联网工程指导组（IESG）批准发布。关于互联网标准的更多信息可参见 RFC 7841 的第 2 节。

关于本文档当前状态、勘误表以及如何提供反馈的信息，可在 <https://www.rfc-editor.org/info/rfc8435> 获取。

## 版权声明

版权所有（c）2018 IETF Trust 及本文档中注明的作者。保留所有权利。

本文档受 BCP 78 以及本文档发布之日生效的 IETF Trust 关于 IETF 文档的法律条款（<https://trustee.ietf.org/license-info>）约束。请仔细阅读这些文档，因为它们描述了您对本文档所享有的权利和限制。从本文档中提取的代码组件必须包含如 Trust 法律条款第 4.e 节所述的简化 BSD 许可证文本，并按简化 BSD 许可证的规定不附带任何担保提供。

## 目录

- [1. 引言](#1-引言)
  - [1.1. 定义](#11-定义)
  - [1.2. 需求用语](#12-需求用语)
- [2. 存储设备的耦合](#2-存储设备的耦合)
  - [2.1. LAYOUTCOMMIT](#21-layoutcommit)
  - [2.2. 将客户端与存储设备隔离](#22-将客户端与存储设备隔离)
  - [2.3. 状态与锁模型](#23-状态与锁模型)
- [3. 灵活文件布局类型的 XDR 描述](#3-灵活文件布局类型的-xdr-描述)
- [4. 设备寻址与发现](#4-设备寻址与发现)
- [5. 灵活文件布局类型](#5-灵活文件布局类型)
- [6. 通过稀疏映射进行条带化](#6-通过稀疏映射进行条带化)
- [7. 从客户端 I/O 错误中恢复](#7-从客户端-io-错误中恢复)
- [8. 镜像](#8-镜像)
- [9. 灵活文件布局类型返回](#9-灵活文件布局类型返回)
- [10. 灵活文件布局类型 LAYOUTERROR](#10-灵活文件布局类型-layouterror)
- [11. 灵活文件布局类型 LAYOUTSTATS](#11-灵活文件布局类型-layoutstats)
- [12. 灵活文件布局类型创建提示](#12-灵活文件布局类型创建提示)
- [13. 召回布局](#13-召回布局)
- [14. 客户端隔离](#14-客户端隔离)
- [15. 安全考虑](#15-安全考虑)
- [16. IANA 考虑](#16-iana-考虑)
- [17. 参考文献](#17-参考文献)
- [致谢](#致谢)
- [作者地址](#作者地址)

## 1. 引言

在并行 NFS（pNFS）中，元数据服务器返回布局类型（layout type）结构，用于描述文件数据位于何处。针对不同的存储系统以及不同的数据在存储设备上的排布方式，存在不同的布局类型。本文档定义了灵活文件布局类型（flexible file layout type），它用于基于文件的数据服务器，这些数据服务器通过 NFS 协议访问：NFSv3 [RFC1813]、NFSv4.0 [RFC7530]、NFSv4.1 [RFC5661] 以及 NFSv4.2 [RFC7862]。

为了提供与 files 布局类型等效的全局状态模型，可以在元数据服务器与 NFSv4.1+ 存储设备之间实现一种后端控制协议。实现既可以定义自己的专有机制，也可以在标准跟踪文档中定义一种控制协议。控制协议的需求在 [RFC5661] 中规定，并在 [RFC8434] 中进一步澄清。

本文档所描述的控制协议基于 NFS。它不在元数据服务器与存储设备之间传递对 stateid 的了解。取而代之的是，存储设备被配置为：元数据服务器对数据文件系统拥有完全访问权限，然后元数据服务器使用合成 id（synthetic ids）来控制客户端对单个文件的访问。

在传统的数据镜像方式中，服务器负责复制、校验和修复数据文件的副本。而在客户端侧镜像中，元数据服务器提供一种布局，向客户端呈现可用的镜像。客户端随后选择一个镜像进行读取，并确保所有写入都发往全部镜像。只有当所有镜像都成功更新时，客户端才认为写事务成功。出现错误时，客户端可以使用 LAYOUTERROR 操作通知元数据服务器，由元数据服务器负责修复该文件的镜像副本。

### 1.1. 定义

**控制通信需求（control communication requirements）**：关于布局（layout）、stateid、文件元数据和文件数据的信息，这些信息必须在元数据服务器与存储设备之间通信。每种布局类型有一组单独的需求。

**控制协议（control protocol）**：某种布局类型的实现用来满足该布局类型的控制通信需求的特定机制。它不必是通常意义上的一种协议。在某些情况下，同一种协议既可用作控制协议，也可用作存储协议。

**客户端侧镜像（client-side mirroring）**：一种由客户端（而非服务器）负责更新布局段的所有镜像副本的特性。

**（文件）数据（(file) data）**：文件系统对象中包含待读或待写数据的那一部分。它是对象的内容，而不是对象的属性。

**数据服务器（data server，DS）**：当通过基于文件的协议访问文件系统对象时，提供该文件数据的 pNFS 服务器。

**隔离（fencing）**：元数据服务器阻止存储设备处理来自特定客户端、针对特定文件的 I/O 的过程。

**文件布局类型（file layout type）**：存储设备通过 NFS 协议访问的一种布局类型（参见 [RFC5661] 第 13 节）。

**gid**：组 id，用于标识文件属于哪个组的数值。

**布局（layout）**：客户端用于访问存储设备上文件数据的信息。该信息包括对协议（布局类型）的说明以及所要使用的存储设备的标识。

**布局 iomode（layout iomode）**：授予客户端只读或读/写 I/O 的许可。

**布局段（layout segment）**：布局的一个细分。该细分可以按布局 iomode（参见 [RFC5661] 第 3.3.20 和 12.2.9 节）、条带模式（参见 [RFC5661] 第 13.3 节）或请求的字节范围来划分。

**布局 stateid（layout stateid）**：由服务器返回的 128 位量，它唯一地定义了服务器为某个特定布局所提供的状态，该布局描述了一种布局类型和一个文件（参见 [RFC5661] 第 12.5.2 节）。此外，[RFC5661] 第 12.5.3 节描述了布局 stateid 与其他类型 stateid 在处理上的差异。

**布局类型（layout type）**：既指明用于访问数据的存储协议，又指明用于在底层存储设备上排布文件数据的聚合方案的规范。

**松耦合（loose coupling）**：当控制协议就是一种存储协议时，称为松耦合。

**（文件）元数据（(file) metadata）**：文件系统对象中包含与文件对象相关的各种描述性数据的那一部分，与文件数据本身相对。它可以包括最后修改时间、访问时间、EOF 位置等。

**元数据服务器（metadata server，MDS）**：为文件系统对象提供元数据信息的 pNFS 服务器。它还负责为文件系统对象生成、召回和撤销布局，执行目录操作，以及在客户端将 I/O 操作指向元数据服务器本身时，对普通文件执行 I/O 操作。

**镜像（mirror）**：一个布局段的副本。注意，如果镜像中的一个副本被更新，则所有副本都必须被更新。

**召回布局（recalling a layout）**：元数据服务器通过回调（callback）优雅地召回某个特定布局。这里的“优雅”是指客户端在将布局返回给元数据服务器之前，有机会刷新任何 WRITE 等操作。

**撤销布局（revoking a layout）**：元数据服务器使某个特定布局失效。一旦发生撤销，元数据服务器将不再把对已撤销布局的任何引用视为有效，存储设备也将不接受基于该布局的任何客户端访问。

**重新同步重建（resilvering）**：从布局段的一个已知完好副本出发，重建该布局段镜像副本的行为。注意，这也可用于创建该布局段的一个新的镜像副本。

**rsize**：用于 READ 操作的数据传输缓冲区大小。

**stateid**：由服务器返回的 128 位量，唯一地定义服务器提供的一组与锁相关的状态。stateid 可以指与打开文件、字节范围锁、委托或布局相关的状态。

**存储设备（storage device）**：当客户端持有适当布局时，可以将 I/O 请求定向到的目标。关于数据服务器与存储设备之间差异的进一步讨论，请参见 [RFC8434] 第 2.1 节。

**存储协议（storage protocol）**：客户端用于对存储设备执行 I/O 操作的协议。每种布局类型都规定了一组存储协议。

**紧耦合（tight coupling）**：一种控制协议被专门设计用于控制通信的架构安排。它可以是专门针对某个特定元数据服务器改造的专有协议，也可以是基于标准跟踪文档的协议。

**uid**：用户 id，用于标识哪个用户拥有某文件的数值。

**wsize**：用于 WRITE 操作的数据传输缓冲区大小。

### 1.2. 需求用语

本文档中的关键词“MUST”（必须）、“MUST NOT”（禁止）、“REQUIRED”（必需的）、“SHALL”（应当）、“SHALL NOT”（不应当）、“SHOULD”（应该）、“SHOULD NOT”（不应）、“RECOMMENDED”（建议）、“NOT RECOMMENDED”（不建议）、“MAY”（可以）和“OPTIONAL”（可选），仅当它们以全部大写字母出现时，才按照 BCP 14 [RFC2119] [RFC8174] 中的描述进行解释。

## 2. 存储设备的耦合

服务器实现可以在元数据服务器与存储设备之间选择松耦合模型或紧耦合模型。[RFC8434] 描述了 pNFS 实现面临的一般性问题。本文档详细说明了新的灵活文件布局类型如何处理这些问题。要实现紧耦合模型，必须定义一种控制协议。由于灵活文件布局对客户端不施加任何特殊要求，该控制协议需要提供：

1. 对安全性以及 LAYOUTCOMMIT 的管理；以及
2. 一种全局 stateid 模型以及对这些 stateid 的管理。

在实现松耦合模型时，唯一的控制协议是某种版本的 NFS，它无法提供全局 stateid 模型，也无法阻止客户端不当使用布局。为使客户端能在该环境中工作，本文档将规定如何管理安全性、状态和锁。

### 2.1. LAYOUTCOMMIT

无论采用哪种耦合模型，元数据服务器在收到 LAYOUTCOMMIT（参见 [RFC5661] 第 18.42 节）时都有责任确保遵守 pNFS 的语义（参见 [RFC8434] 第 3.1 节）。这些语义确实包含一项要求：==写入数据存储设备的数据必须在 LAYOUTCOMMIT 发生之前保持稳定==。

客户端有责任确保在元数据服务器开始查询存储设备有关文件的变更之前，数据文件已经稳定。如果对某个存储设备的任何 WRITE 操作所返回的 stable_how 不等于 FILE_SYNC，那么在向元数据服务器发送 LAYOUTCOMMIT 之前，客户端必须先向被写入的存储设备发送 COMMIT。注意，如果客户端尚未向存储设备执行 COMMIT，则 LAYOUTCOMMIT 可能无法与对该存储设备的最后一次 WRITE 操作保持同步。

### 2.2. 将客户端与存储设备隔离

==对于松耦合的存储设备，元数据服务器对数据文件使用合成的 uid（用户 id）和 gid（组 id）==：数据文件的 uid 所有者被授予读/写访问权限，gid 所有者被授予只读访问权限。作为布局的一部分（参见第 5.1 节中的 ffds_user 和 ffds_group），客户端会获知在访问数据文件所需的远程过程调用（RPC）[RFC5531] 凭据中应使用的用户和组。对客户端的隔离是这样实现的：元数据服务器更改存储设备上数据文件的合成 uid 和/或 gid 所有者，从而隐式地使尚未完成的 RPC 凭据失效。客户端若针对所期望的访问出示错误的凭据，将收到 NFS4ERR_ACCESS 错误。

在这种松耦合模型下，元数据服务器无法隔离单个客户端，而只能隔离所有客户端。不过，随着其他客户端对隔离作出反应——返回它们的布局并尝试获取新布局——元数据服务器可以下发新的 uid 和 gid 来允许访问。

建议在存储设备的文件系统上实现通用的访问控制方法，仅允许元数据服务器 root（超级用户）访问存储设备，并将存放数据文件的所有目录的所有者设置为 root 用户。这种方法提供了一种切实可行的模型来强制执行访问控制并隔离合作的客户端，但它无法抵御恶意客户端；因此，它提供的是与 AUTH_SYS 等同的安全级别。建议元数据服务器与存储设备之间的通信能够抵御窃听者以及中间人协议篡改。安全措施可以是物理安全（例如服务器共同位于物理安全的区域内）、加密通信或其他某种技术。

对于紧耦合的存储设备，元数据服务器将数据文件的用户和组所有者、模式位（mode bits）以及访问控制列表（ACL）设置为与元数据文件相同。客户端必须与存储设备进行身份认证，并经历与通过元数据服务器时所经历的相同的授权过程。在紧耦合情况下，隔离是控制协议的责任，本文档不做详细描述。但是，紧耦合锁模型（参见第 2.3 节）的实现需要一种方式，通过使存储设备上相应的 stateid 失效，来阻止特定客户端访问特定文件。在这种情况下，客户端将收到 NFS4ERR_BAD_STATEID 错误。

客户端不需要知道元数据服务器与存储设备之间使用哪种模型。它只需对在与存储设备交互中出现的任何错误作出一致的反应即可。它既应该将布局和错误返回给元数据服务器，又应该请求一个新布局。此时，元数据服务器可以选择下发一个新布局、不下发任何布局（迫使 I/O 经由它进行），或者拒绝客户端对该文件的进一步访问。

#### 2.2.1. 合成 uid/gid 的实现说明

用于在松耦合存储设备中实现隔离的合成 uid 和 gid 的选择方法，纯粹是实现问题。也就是说，管理员可以限制 Lightweight Directory Access Protocol（LDAP）中 'uid' 字段 [RFC4519] 可用的这类 id 的范围。管理员还可能能够选择一个永远不会被用来授予访问权限的 id。这样，当元数据服务器收到对某个文件的访问请求时，它会向存储设备发送一个 SETATTR，以设置数据文件的所有者和组。用户和组可以按轮询（round-robin）方式从可用 id 范围中选取。

这些 id 会作为 ffds_user 和 ffds_group 返回给客户端，客户端会将它们作为 RPC 凭据出示给存储设备。当客户端完成对文件的访问、且元数据服务器知道没有其他客户端正在访问该文件时，它可以将所有者和组重置为受限 id，以限制对数据文件的访问。

当元数据服务器想要隔离某个客户端时，它将合成 uid 和/或 gid 更改为受限 id。注意，使用受限 id 可以确保所有者一定会发生变化，并且至少有一个 id 是永远不会被允许访问的。

在 AUTH_SYS 安全模型下，应避免使用值为 0 的合成 uid 和 gid。它们通常要么授予对存储设备上文件的超级用户访问权限，要么被映射到匿名 id。在第一种情况下，即使数据文件已被隔离，客户端仍可能访问该文件。在第二种情况下，多个 id 可能被映射到匿名 id。

#### 2.2.2. 使用合成 uid/gid 的示例

用户 loghyr 在元数据服务器上创建文件 "ompha.c"，元数据服务器随后在存储设备上创建相应的数据文件。

元数据服务器上的条目可能如下所示：

```
-rw-r--r--    1 loghyr  staff    1697 Dec  4 11:31 ompha.c
```

在存储设备上，该文件可能被分配某个不可预测的合成 uid/gid，以拒绝访问：

```
-rw-r-----    1 19452   28418    1697 Dec  4 11:31 data_ompha.c
```

当文件在客户端上被打开和访问时，用户将尝试获取该数据文件的布局。由于布局对用户一无所知（也不关心用户是谁），因此无论用户是 loghyr 还是 garbo 打开该文件都没有关系。客户端必须出示 uid 19452 才能获得写权限。如果它出示任何其他 uid 值，则必须给出 gid 28418 才能获得读访问权。

此外，如果元数据服务器决定隔离该文件，它应更改 uid 和/或 gid，使这些值既不与该文件的先前值匹配，也不与基于先前隔离的可预测变化相匹配。

```
-rw-r-----    1 19453   28419    1697 Dec  4 11:31 data_ompha.c
```

存储设备上的合成 gid 集合的选择应满足：在存储设备使用的任何名称服务中都不存在映射，即每个组都不应有任何成员。

如果布局段的 iomode 为 LAYOUTIOMODE4_READ，则元数据服务器应返回一个在存储设备上未设置的合成 uid。此时只有合成 gid 是有效的。

因此，在松耦合模型中，强制执行文件权限完全是客户端的责任。若要允许 loghyr 进行写访问，它将向存储设备发送一个 RPC，凭据为 1066:1067。若要允许 garbo 进行读访问，它将向存储设备发送一个 RPC，凭据为 1067:1067。只要 uid 的值不是获取布局时授予的合成 uid，该值是什么都无所谓。

将权限检查的执行职责推给客户端看似削弱了安全性，但客户端在向服务器发送修改之前，可能本来就已经负责执行权限检查了。对于缓存写入（cached writes），客户端始终负责跟踪谁在修改文件，并确保不会将来自多个用户的请求合并为一个请求。

### 2.3. 状态与锁模型

一种实现总是可以作为松耦合模型来部署。然而，无法通过 NFS 协议让存储设备表明自己一定能够参与紧耦合模型：

- 实现了 NFSv3 和 NFSv4.0 协议的存储设备总是被当作松耦合处理。
- 未在 EXCHANGE_ID 中返回设置 EXCHGID4_FLAG_USE_PNFS_DS 标志的 NFSv4.1+ 存储设备，表明它们应被当作松耦合处理。从锁的角度来看，它们与 NFSv4.0 存储设备的处理方式相同。
- 在 EXCHANGE_ID 中返回并设置 EXCHGID4_FLAG_USE_PNFS_DS 标志来标识自身的 NFSv4.1+ 存储设备，则有可能被紧耦合。它们将使用一种后端控制协议来实现 [RFC5661] 所述的全局 stateid 模型。

存储设备必须通过控制协议被发现或通告出来，才能启用紧耦合模型。

#### 2.3.1. 松耦合锁模型

当请求与锁相关的操作时，这些操作主要由元数据服务器处理，由它生成相应的 stateid。当使用某个 NFSv4 版本作为数据访问协议时，元数据服务器可以向存储设备发出与 stateid 相关的请求。但它并非必须这样做，而且由此产生的 stateid 只有元数据服务器和存储设备知道。

基于这一基本结构，与锁相关的操作按如下方式处理：

- OPEN 操作由元数据服务器处理。stateid 由元数据服务器选择，并与描述客户端到元数据服务器连接的客户端 ID（client ID）相关联。元数据服务器可能需要与存储设备交互来定位要打开的文件，但在存储设备上无需使用任何与锁相关的功能。OPEN_DOWNGRADE 和 CLOSE 只需要在元数据服务器上本地执行。
- 建议性字节范围锁可以在元数据服务器上本地实现。与 OPEN 的情况一样，与字节范围锁相关联的 stateid 由元数据服务器分配，且仅在元数据服务器上使用。
- 委托（delegation）由元数据服务器分配；当处理相互冲突的 OPEN 时，由元数据服务器发起召回。无需存储设备参与。
- TEST_STATEID 和 FREE_STATEID 在元数据服务器上本地处理，无需存储设备参与。

发往存储设备的所有 I/O 操作都使用匿名 stateid 来执行。因此，存储设备无法获知负责发出特定 I/O 操作的 openowner 和 lockowner。由此：

- 无法支持强制性字节范围锁，因为存储设备无法区分代表锁所有者执行的 I/O 与其他人执行的 I/O。
- 共享保留（share reservation）的执行是客户端的责任。尽管 I/O 使用匿名 stateid 执行，但客户端必须确保它拥有与 openowner 关联的有效 stateid。

一旦发生 stateid 被撤销的情况，元数据服务器负责阻止客户端访问，因为它无法确保客户端知道相关 stateid 已被撤销。

由于客户端从不接收存储设备生成的 stateid，因此存储设备上不存在客户端租约（lease），也就不存在租约到期的可能，即使访问经由 NFSv4 协议也是如此。客户端在元数据服务器上拥有租约。在处理租约到期时，元数据服务器可能需要使用隔离手段，来防止已被撤销的 stateid 被不知情的客户端所依赖。

#### 2.3.2. 紧耦合锁模型

当请求与锁相关的操作时，这些操作主要由元数据服务器处理，由它生成相应的 stateid。这些 stateid 必须通过控制协议机制告知存储设备，其细节不在本文档中讨论。

基于这一基本结构，与锁相关的操作按如下方式处理：

- OPEN 操作主要在元数据服务器上处理。stateid 由元数据服务器选择，并与描述客户端到元数据服务器连接的客户端 ID 相关联。元数据服务器需要与存储设备交互来定位要打开的文件，并让存储设备知晓元数据服务器所选 stateid 与它所代表的客户端及 openowner 之间的关联。OPEN_DOWNGRADE 和 CLOSE 最初在元数据服务器上执行，但所作的状态变更必须传播到存储设备。
- 建议性字节范围锁可以在元数据服务器上本地实现。与 OPEN 的情况一样，与字节范围锁相关联的 stateid 由元数据服务器分配，并可在元数据服务器上使用。由于 I/O 操作被允许出示锁 stateid，元数据服务器需要能够让存储设备知晓元数据服务器所选 stateid 与其对应的 open stateid 之间的关联。
- 当元数据服务器和存储设备都具有适当支持时，可以支持强制性字节范围锁。与建议性字节范围锁的情况一样，这些锁由元数据服务器分配，并可在元数据服务器上使用。为使存储设备能够执行强制性锁，元数据服务器需要能够让存储设备知晓元数据服务器所选 stateid 与它所代表的客户端、openowner 以及锁（即 lockowner、字节范围和锁类型）之间的关联。由于 I/O 操作被允许出示锁 stateid，这些信息需要传播到所有可能被定向 I/O 的存储设备，而不仅仅是包含被锁区域的存储设备。
- 委托由元数据服务器分配；当处理相互冲突的 OPEN 时，由元数据服务器发起召回。由于 I/O 操作被允许出示委托 stateid，元数据服务器需要具备以下能力：（1）让存储设备知晓元数据服务器所选 stateid 与它所代表的文件句柄和委托类型之间的关联；（2）解除这种关联。
- TEST_STATEID 在元数据服务器上本地处理，无需存储设备参与。
- FREE_STATEID 在元数据服务器上处理，但元数据服务器需要具备将请求传播到相应存储设备的能力。

由于客户端将拥有并使用在存储设备上有效的 stateid，因此存储设备上将存在客户端租约，也就确实存在租约到期的可能。对存储设备而言，最佳做法是出于礼节而保留这些锁。但是，如果它不这样做，控制协议机制需要提供在元数据服务器与存储设备之间同步锁状态的手段。

客户端在元数据服务器上也将拥有租约，这些租约同样可能到期。在处理租约到期时，元数据服务器应使用控制协议机制，使其能够在存储设备上使已撤销的 stateid 失效。如果客户端没有响应，元数据服务器可能需要使用隔离手段，来防止存储设备对已撤销的 stateid 进行操作。

## 3. 灵活文件布局类型的 XDR 描述

本文档包含灵活文件布局类型的外部数据表示（XDR）[RFC4506] 描述。XDR 描述以这样一种方式嵌入本文档中，使读者可以轻松地将其提取为可直接编译的形式。读者可以将本文档输入以下 shell 脚本，以生成灵活文件布局类型的机器可读 XDR 描述：

```sh
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
```

也就是说，如果将上述脚本保存为名为 "extract.sh" 的文件，并将本文档保存为名为 "spec.txt" 的文件，则读者可以执行：

```sh
sh extract.sh < spec.txt > flex_files_prot.x
```

该脚本的作用是去掉每行前导空白字符，以及一个 "///" 哨兵序列。

内嵌的 XDR 文件头如下所示。后续带有哨兵序列的 XDR 描述贯穿全文嵌入。

注意，本文档中包含的 XDR 代码依赖于 NFSv4.1 的 nfs4_prot.x 文件 [RFC5662] 中的类型。这包括以 4 结尾的 nfs 类型（如 offset4、length4 等），以及更通用的类型（如 uint32_t 和 uint64_t）。

### 3.1. 代码组件许可声明

XDR 描述以及用于提取 XDR 描述的脚本均为《Trust 法律条款（TLP）》[LEGAL] 第 4 节所述的代码组件（Code Components）。这些代码组件按照该文档的条款获得许可。

```c
/*
 * Copyright (c) 2018 IETF Trust and the persons identified
 * as authors of the code.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with
 * or without modification, are permitted provided that the
 * following conditions are met:
 *
 * - Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 * - Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer in the documentation and/or other
 *   materials provided with the distribution.
 *
 * - Neither the name of Internet Society, IETF or IETF
 *   Trust, nor the names of specific contributors, may be
 *   used to endorse or promote products derived from this
 *   software without specific prior written permission.
 *
 *   THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS
 *   AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED
 *   WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 *   IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
 *   A PARTICULAR PURPOSE ARE DISCLAIMED.  IN NO
 *   EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
 *   LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
 *   EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
 *   NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 *   SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 *   INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
 *   LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
 *   OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING
 *   IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF
 *   ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 *
 * This code was derived from RFC 8435.
 * Please reproduce this note if possible.
 */

/*
 * flex_files_prot.x
 */

/*
 * The following include statements are for example only.
 * The actual XDR definition files are generated separately
 * and independently and are likely to have a different name.
 * %#include <nfsv42.x>
 * %#include <rpc_prot.x>
 */
```

## 4. 设备寻址与发现

对存储设备的数据操作要求客户端知道该存储设备的网络地址。客户端使用 NFSv4.1+ 的 GETDEVICEINFO 操作（[RFC5661] 第 18.40 节）来检索该信息。

### 4.1. ff_device_addr4

ff_device_addr4 数据结构由服务器作为 layout-type 特定的 opaque 字段 da_addr_body（位于 device_addr4 结构中）返回，这是 GETDEVICEINFO 操作成功时返回的。

```xdr
struct ff_device_versions4 {
        uint32_t        ffdv_version;
        uint32_t        ffdv_minorversion;
        uint32_t        ffdv_rsize;
        uint32_t        ffdv_wsize;
        bool            ffdv_tightly_coupled;
};

struct ff_device_addr4 {
        multipath_list4     ffda_netaddrs;
        ff_device_versions4 ffda_versions<>;
};
```

ffda_netaddrs 字段用于定位存储设备。服务器必须将它设置为一个包含一个或多个设备网络地址的列表。

ffda_versions 数组允许元数据服务器向客户端提供关于 NFS 版本、次版本号以及耦合强度的选择。ffdv_version 和 ffdv_minorversion 表示用于访问存储设备的 NFS 协议。本布局规范定义了 ffdv_versions 3 和 4 的语义。如果 ffdv_version 等于 3，则服务器必须将 ffdv_minorversion 设置为 0，并将 ffdv_tightly_coupled 设置为 false。此时客户端必须使用 NFSv3 协议 [RFC1813] 访问存储设备。如果 ffdv_version 等于 4，则服务器必须将 ffdv_minorversion 设置为 NFSv4 的某个次版本号之一，客户端必须使用具有相应次版本的 NFSv4 访问存储设备。

注意，虽然客户端在从元数据服务器获取设备列表时，可能会断定自己无法使用 ffdv_version、ffdv_minorversion 和 ffdv_tightly_coupled 的任何已配置组合，但它无法向元数据服务器指出具体是哪个设备与自己存在版本不兼容。不过，如果客户端等到从元数据服务器检索布局时再处理，它就可以在那时明确指认出有问题的存储设备（参见第 5.4 节）。

ffdv_rsize 和 ffdv_wsize 用于传达存储设备所支持的最大 rsize 和 wsize。由于存储设备的 rsize 或 wsize 可能与元数据服务器不同，ffdv_rsize 和 ffdv_wsize 允许元数据服务器代表存储设备传达该信息。

ffdv_tightly_coupled 告知客户端元数据服务器是否与存储设备紧耦合。注意，即使数据协议至少是 NFSv4.1，实际生效的仍可能是松耦合。如果未设置 ffdv_tightly_coupled，则客户端必须在对元数据服务器发送 LAYOUTCOMMIT 之前，将针对该文件的写入提交到存储设备。也就是说，写入必须由客户端通过发出 stable_how == FILE_SYNC 的 WRITE 操作来提交到稳定存储，或者在 stable_how != FILE_SYNC 的 WRITE 操作之后发出 COMMIT 来提交（参见 [RFC1813] 第 3.3.7 节）。

### 4.2. 存储设备多路径

灵活文件布局类型支持到多个存储设备地址的多路径。存储设备级多路径用于通过汇聚（trunking）实现带宽扩展，以及在存储设备故障时提供更高的可用性。多路径允许客户端切换到另一个存储设备地址——该地址可能是正在导出相同数据条带单元的另一台存储设备的地址——而无需联系元数据服务器获取新布局。

为支持存储设备多路径，ffda_netaddrs 包含一个由一个或多个存储设备网络地址组成的数组。该数组（数据类型 multipath_list4）表示一个存储设备列表（每个存储设备由一个网络地址标识），并且某个存储设备有可能在列表中出现多次。

客户端可以自由地使用其中任意网络地址作为发送存储设备请求的目的地。如果某些网络地址作为数据通路的可取程度低于其他地址，则元数据服务器不应将它们包含在 ffda_netaddrs 中。如果确实存在较不可取的网络地址用于提供故障转移，则建议的提供方式是：将它们放在一个替代的“设备 ID 到设备地址”映射中，或者放在一个替代的设备 ID 中。当客户端使用 ffda_netaddrs 中的所有地址都无法得到存储设备的响应时，它应发送 GETDEVICEINFO 来尝试替换现有的“设备 ID 到设备地址”映射。如果元数据服务器检测到由 ffda_netaddrs 表示的所有网络路径都不可用，则元数据服务器应发送 CB_NOTIFY_DEVICEID（前提是客户端已表明它希望对变更的设备 ID 获得设备 ID 通知），以将“设备 ID 到设备地址”映射更改为可用地址。如果设备 ID 本身将被替换，则元数据服务器应召回所有使用该设备 ID 的布局，从而强制客户端通过 LAYOUTGET 和 GETDEVICEINFO 获取新的布局和设备 ID 映射。

一般来说，如果两个网络地址出现在 ffda_netaddrs 中，它们将指向同一台存储设备。当通过 NFSv4.1 或更高的次版本访问存储设备时，这两个存储设备地址将支持实现客户端 ID 或会话汇聚（后者是建议的做法），其定义见 [RFC5661]。这两个存储设备地址将共享相同的服务器所有者（server owner）或服务器所有者的主 ID（major ID）。使用汇聚时，这两个存储设备地址并非总必须指向同一台存储设备。例如，数据可以是只读的，且数据由完全一致的副本组成。

## 5. 灵活文件布局类型

[RFC5662] 中引入的原始 layouttype4 被修改为：

```xdr
    enum layouttype4 {
        LAYOUT4_NFSV4_1_FILES   = 1,
        LAYOUT4_OSD2_OBJECTS    = 2,
        LAYOUT4_BLOCK_VOLUME    = 3,
        LAYOUT4_FLEX_FILES      = 4
    };

    struct layout_content4 {
        layouttype4             loc_type;
        opaque                  loc_body<>;
    };

    struct layout4 {
        offset4                 lo_offset;
        length4                 lo_length;
        layoutiomode4           lo_iomode;
        layout_content4         lo_content;
    };
```

本文档定义了与 layouttype4 值 LAYOUT4_FLEX_FILES 相关联的结构。[RFC5661] 将 loc_body 结构指定为 XDR 类型 "opaque"。通用的 pNFS 客户端层不解释该 opaque 布局，而是由灵活文件布局类型实现来解释它。本节定义了这个原本 opaque 的值的结构，即 ff_layout4。

### 5.1. ff_layout4

```xdr
const FF_FLAGS_NO_LAYOUTCOMMIT   = 0x00000001;
const FF_FLAGS_NO_IO_THRU_MDS    = 0x00000002;
const FF_FLAGS_NO_READ_IO        = 0x00000004;
const FF_FLAGS_WRITE_ONE_MIRROR  = 0x00000008;

typedef uint32_t            ff_flags4;

struct ff_data_server4 {
    deviceid4               ffds_deviceid;
    uint32_t                ffds_efficiency;
    stateid4                ffds_stateid;
    nfs_fh4                 ffds_fh_vers<>;
    fattr4_owner            ffds_user;
    fattr4_owner_group      ffds_group;
};

struct ff_mirror4 {
    ff_data_server4         ffm_data_servers<>;
};

struct ff_layout4 {
    length4                 ffl_stripe_unit;
    ff_mirror4              ffl_mirrors<>;
    ff_flags4               ffl_flags;
    uint32_t                ffl_stats_collect_hint;
};
```

ff_layout4 结构描述了当前布局段所描述的数据文件的那一部分的布局。它要么是数据文件该部分的单个实例，要么是该部分的一组镜像副本。当镜像生效时，它可以防止布局段中的数据丢失。

虽然上面的 XDR 中没有明确显示，但 LAYOUTGET4res 的 logr_layout 数组中返回的每个 layout4 元素（参见 [RFC5661] 第 18.43.2 节）都描述一个布局段。因此，每个 ff_layout4 也描述一个布局段。一个文件有可能由多个布局段拼接而成。每个布局段可以表示不同的条带化参数。

ffl_stripe_unit 字段是当前布局段所使用的条带单元大小。条带数量在每个镜像内部由 ffm_data_servers 中的元素个数给出。如果条带数量为 1，则 ffl_stripe_unit 的值必须默认为 0。唯一支持的映射方案是稀疏（sparse）映射，详见第 6 节。注意，这里有一个假设：所有镜像的条带单元大小和条带数量都相同。

ffl_mirrors 字段是为当前条带提供存储空间的镜像存储设备数组；参见图 1。

ffl_stats_collect_hint 字段向客户端提供一个提示，告知服务器希望客户端以多高的频率报告针对某文件的 LAYOUTSTATS。时间单位为秒。

```
                      +-----------+
                      |           |
                      |           |
                      |   File    |
                      |           |
                      |           |
                      +-----+-----+
                            |
               +------------+------------+
               |                         |
          +----+-----+             +-----+----+
          | Mirror 1 |             | Mirror 2 |
          +----+-----+             +-----+----+
               |                         |
          +-----------+            +-----------+
          |+-----------+           |+-----------+
          ||+-----------+          ||+-----------+
          +||  Storage  |          +||  Storage  |
           +|  Devices  |           +|  Devices  |
            +-----------+            +-----------+

                           图 1
```

ffs_mirrors 字段表示当前布局段每个镜像副本的状态信息数组。每个元素由 ff_mirror4 类型描述。

ffds_deviceid 提供保存数据文件的存储设备的 deviceid。

ffds_fh_vers 是数据文件的文件句柄数组，与给定存储设备上可用的 NFS 版本相匹配。ffds_fh_vers 中的元素个数必须与 ffda_versions 中的元素个数完全一致。数组中的每个元素对应于为该设备提供的某种特定的 ffdv_version、fffdv_minorversion 和 ffdv_tightly_coupled 组合。该数组允许这样的服务器实现：针对不同的版本、次版本和耦合强度组合使用不同的文件句柄。关于如何处理客户端与存储设备之间的版本问题，请参见第 5.4 节。

对于紧耦合，ffds_stateid 提供客户端用于访问该文件的 stateid。对于松耦合且使用 NFSv4 存储设备的情况，客户端将不得不使用匿名 stateid 对存储设备执行 I/O。由于没有控制协议，元数据服务器的 stateid 无法用于提供全局 stateid 模型。因此，服务器必须将 ffds_stateid 设置为匿名 stateid。

对 ffds_stateid 的这一规定限制了 NFSv4.x 存储协议下的两种模型：

- **松耦合模型**：该 stateid 必须是匿名 stateid。
- **紧耦合模型**：该 stateid 必须是全局 stateid。

有一个问题源于这样一个事实：ffds_stateid 被定义为单个元素，而 ffds_fh_vers 被定义为数组。存储设备上每个打开的文件都可能需要自己的 open stateid。由于本文档所述协议版本已有成熟的松耦合实现，这里的潜在问题没有被解决。如果今后为同一个底层文件提供多个 stateid 变得重要，未来可以定义解决这些问题的布局类型。

对于松耦合存储设备，ffds_user 和 ffds_group 提供合成用户和组，客户端在访问数据文件时，将它们用于出示给存储设备的 RPC 凭据中。对于紧耦合存储设备，存储设备上的用户和组将与元数据服务器上的相同；也就是说，如果设置了 ffdv_tightly_coupled（参见第 4.1 节），则客户端必须忽略 ffds_user 和 ffds_group。

ffds_user 和 ffds_group 的允许值在 [RFC5661] 第 5.9 节中分别被指定为 owner 和 owner_group。为兼容 NFSv3，由十进制数值组成、没有前导零的用户和组字符串，可以被选择提供此类支持的客户端和服务器给予特殊解释。接收方可以将这样的用户或组字符串视为代表与具有相应数值的 NFSv3 uid 或 gid 所代表的相同的用户。注意，如果使用 Kerberos 作为安全机制，则期望这些值是 name@domain 形式的字符串。

ffds_efficiency 描述元数据服务器对每个镜像有效性的评估。注意，这是按布局而不是按设备的度量，因为该度量可能因感知到的负载、对元数据服务器的可用性等因素而变化。数值越高，表示感知到的效用越高。客户端选择最佳访问镜像的方式在第 8.1 节中讨论。

ffl_flags 是一个位图，允许元数据服务器告知客户端因存储设备耦合程度不同而可能产生的特定状况。

**FF_FLAGS_NO_LAYOUTCOMMIT**：可以设置，表示客户端无需向元数据服务器发送 LAYOUTCOMMIT。

**FF_FLAGS_NO_IO_THRU_MDS**：可以设置，表示客户端不应向元数据服务器发送 I/O 操作。也就是说，即使客户端能够断定到某个存储设备的网络连接已断开，客户端也不应尝试通过元数据服务器来代理（proxy）该 I/O。

**FF_FLAGS_NO_READ_IO**：可以设置，表示客户端不应使用 iomode 为 LAYOUTIOMODE4_RW 的布局发送 READ 请求。取而代之的是，它应向元数据服务器请求一个 iomode 为 LAYOUTIOMODE4_READ 的布局。

**FF_FLAGS_WRITE_ONE_MIRROR**：可以设置，表示客户端只需要更新其中一个镜像（参见第 8.2 节）。

#### 5.1.1. LAYOUTGET 返回的错误码

[RFC5661] 对于客户端在收到 NFS4ERR_LAYOUTTRYLATER、NFS4ERR_LAYOUTUNAVAILABLE 和 NFS4ERR_DELAY 错误后应如何继续 LAYOUTGET 操作，提供的指导很少。在本文档的上下文中：

**NFS4ERR_LAYOUTUNAVAILABLE**：没有可用布局，I/O 应发往元数据服务器。注意，在召回之前可能持有布局，而在召回之后则可能没有。

**NFS4ERR_LAYOUTTRYLATER**：存在某些问题导致布局暂时无法被授予。如果客户端已经拥有适当的布局，则应继续对存储设备进行 I/O。

**NFS4ERR_DELAY**：存在某些问题导致布局暂时无法被授予。如果客户端已经拥有适当的布局，则不应继续对存储设备进行 I/O。

#### 5.1.2. 客户端与 FF_FLAGS_NO_IO_THRU_MDS 的交互

即使元数据服务器提供了 FF_FLAGS_NO_IO_THRU_MDS 标志，客户端仍然可以对元数据服务器执行 I/O。该标志起提示作用。该标志向客户端表明：元数据服务器倾向于将元数据 I/O 与数据 I/O 分离，最有可能的原因是与性能有关。

### 5.2. LAYOUTCOMMIT

灵活文件布局不在 LAYOUTCOMMIT 的 loca_layoutupdate 参数中使用 lou_body。如果 lou_type 是 LAYOUT4_FLEX_FILES，则 lou_body 字段必须具有零长度（参见 [RFC5661] 第 18.42.1 节）。

### 5.3. 设备与布局之间的交互

在 [RFC5661] 中，文件布局类型的定义使得多路径与文件句柄之间的关系可能导致 0 个、1 个或 N 个文件句柄（参见第 13.3 节）。这样设计的一些理由包括：共享相同文件句柄的集群服务器，或者允许同一存储设备上存在某文件的多个只读副本。在灵活文件布局类型中，虽然也有一个文件句柄数组，但它们与所使用的多路径无关。如果元数据服务器想在同一存储设备上提供同一文件的多个只读副本，那么它应该提供多个镜像实例，每个实例具有不同的 ff_device_addr4。这样客户端就能判定：由于每个 ffds_fh_vers 都不同，当前布局段存在该文件的多个可用副本。

### 5.4. 处理版本错误

当元数据服务器在 ff_device_addr4 中提供 ffda_versions 数组时（参见第 4.1 节），客户端能够判断自己是否可以使用所提供的某种 ffdv_version、ffdv_minorversion 和 ffdv_tightly_coupled 组合来访问存储设备。然而，由于 GETDEVICEINFO 在错误报告方面的局限（参见 [RFC5661] 第 18.40 节），客户端无法指明具体是哪台设备无法通过所提供的某种 ffdv_version 和 ffdv_minorversion 组合进行通信。通过在 LAYOUTRETURN（参见 [RFC5661] 第 18.44 节）或 LAYOUTERROR（参见 [RFC7862] 第 15.6 节以及本文档第 10 节）内部使用 ff_ioerr4（参见第 9.1.1 节），客户端可以定位出有问题的存储设备。

针对 LAYOUTRETURN 和/或 LAYOUTERROR 应返回的错误码是 NFS4ERR_MINOR_VERS_MISMATCH。无论不匹配是主版本（例如客户端可以使用 NFSv3 但不能使用 NFSv4）还是次版本（例如客户端可以使用 NFSv4.1 但不能使用 NFSv4.2），该错误都表明：对于所提供的 ffdv_version 和 ffdv_minorversion 的所有组合，客户端都无法与该存储设备通信。客户端可以重试 GETDEVICEINFO，看元数据服务器是否能提供不同的组合；也可以退回到经由元数据服务器执行 I/O。

## 6. 通过稀疏映射进行条带化

其他布局类型同时支持文件内逻辑偏移到物理偏移的稠密（dense）映射和稀疏（sparse）映射（例如参见 [RFC5661] 第 13.4 节），而灵活文件布局类型只支持稀疏映射。

在稀疏映射下，文件内的逻辑偏移（L）同时也是存储设备上的物理偏移。如 [RFC5661] 第 13.4.4 节所述，这会导致在每条不包含当前条带索引的存储设备上产生空洞（hole）。

- L：文件内的逻辑偏移
- W：条带宽度（stripe width），W = ffm_data_servers 中元素的个数
- S：一个条带的字节数，S = W * ffl_stripe_unit
- N：条带编号，N = L / S

## 7. 从客户端 I/O 错误中恢复

pNFS 客户端在直接访问存储设备时可能会遇到错误。但是，从 I/O 错误中恢复是元数据服务器的责任。当使用 LAYOUT4_FLEX_FILES 布局类型时，客户端必须在 LAYOUTRETURN 时向服务器报告 I/O 错误，报告时使用 ff_ioerr4 结构（参见第 9.1.1 节）。

元数据服务器分析错误并确定所需的恢复操作，例如恢复介质故障或重建丢失的数据文件。

元数据服务器必须召回所有尚未完成的布局，以便让自己对被恢复条带获得独占写访问权，并防止其他客户端遇到相同的错误状况。在这些情况下，服务器必须完成恢复，然后才能对受影响的字节范围下发任何新布局。

虽然客户端实现可以选择向发起该 I/O 操作的应用程序传播相应的错误并丢弃任何未写入的数据，但客户端应尝试通过请求新布局、或经由常规的 NFSv4.1+ READ 或 WRITE 操作将 I/O 发送给元数据服务器，来重试原始的 I/O 操作。客户端应首先尝试检索新布局并使用存储设备重试该 I/O 操作；只有当错误持续存在时，才应经由元数据服务器重试该 I/O 操作。

## 8. 镜像

灵活文件布局类型为受布局段约束的文件数据镜像提供了一个简单的模型。该模型并不假定镜像的每个副本都以相同的方式存储在存储设备上。例如，某个设备可能对数据采用压缩或去重。但是，文件内容在线路上的传输必须看起来完全一致。注意，这是所选 XDR 表示方式的一个约束：布局段的每个镜像副本具有相同的条带模式（参见图 1）。

元数据服务器负责确定镜像副本的数量以及每个镜像的位置。虽然客户端可以提供它想要多少个副本的提示（参见第 12 节），但元数据服务器可以忽略该提示；无论如何，客户端都无法指定存储设备（这也意味着无法指定用于访问布局段的耦合程度和/或协议级别）或该存储设备的位置。

镜像布局段的更新通过客户端侧镜像来完成。采用这种方式，客户端负责确保对它通过布局获知的所有布局段副本进行修改。如果某个布局段正在被重新同步重建（resilvering）到某个存储设备，该镜像副本将不会出现在布局中。因此，在客户端通过布局获得该副本之前，元数据服务器必须负责更新该副本。如果 ffl_flags 中设置了 FF_FLAGS_WRITE_ONE_MIRROR，则客户端只需要更新其中一个镜像（参见第 8.2 节）。如果客户端通过元数据服务器写入布局段，则元数据服务器必须更新该镜像的所有副本。如第 8.3 节所示，在重新同步重建期间，布局会被召回，客户端必须经由元数据服务器进行修改。

### 8.1. 选择镜像

当元数据服务器向客户端授予布局时，它可以通过 ffds_efficiency 成员，让客户端知道一旦请求到达存储设备后，它预期每个镜像的访问速度有多快。虽然计算该值的算法留待元数据服务器实现去决定，但可能对该计算有贡献的因素包括：存储设备的速度、设备可用的物理内存、操作系统版本、当前负载等。

然而，不应纳入该计算的是客户端与存储设备之间感知到的网络距离。客户端更适合根据自身过去通过两个节点之间不同可用网络接口与存储设备交互的经验来作出该判断；也就是说，元数据服务器可能不知道客户端与存储设备之间的短暂中断，因为它在该子网上没有存在点（presence）。

因此，由客户端决定对哪个镜像进行访问以读取文件。对镜像布局段的写入要求如下所述。

### 8.2. 写入镜像

#### 8.2.1. 单个存储设备更新镜像

如果 ffl_flags 中的 FF_FLAGS_WRITE_ONE_MIRROR 标志被设置，客户端只需要更新布局段的一个副本。在这种情况下，当任何一个镜像被更新时，存储设备必须确保该镜像的所有副本都被更新。如果存储设备在更新某个镜像时出错，那么它必须告知客户端原始的 WRITE 操作出错。随后客户端必须通知元数据服务器（参见第 8.2.3 节）。客户端关于 COMMIT 的责任在第 8.2.4 节中说明。客户端可以选择任意一个镜像，并且在做选择时可以使用第 8.1 节所述的 ffds_efficiency。

#### 8.2.2. 客户端更新所有镜像

如果 ffl_flags 中的 FF_FLAGS_WRITE_ONE_MIRROR 标志未被设置，客户端负责更新布局中提供给它的布局段的所有镜像副本。一次更新失败就足以使整个操作失败。如果除一个副本之外的所有副本都更新成功，而最后一个副本返回错误，则客户端需要将该错误通知元数据服务器。客户端可以使用 LAYOUTRETURN 或 LAYOUTERROR 来通知元数据服务器对该存储设备的更新失败。==如果客户端按串行方式更新镜像，那么它应在遇到的第一个错误处停止，并将该错误报告给元数据服务器。如果客户端按并行方式更新镜像，那么它应等到所有存储设备都响应，以便报告更新过程中遇到的所有错误==。

#### 8.2.3. 处理写入错误

当客户端向元数据服务器报告写入错误时，元数据服务器负责决定是否要从布局中移除出错的镜像、该镜像是否已从某种瞬时错误中恢复等。当客户端尝试获取新布局时，元数据服务器通过布局的内容将决定告知客户端。客户端不得假定先前布局的内容与新布局的内容一致。如果有尚未提交到所有镜像的更新，则客户端必须将这些更新重新发送到所有镜像。

协议中没有提供让元数据服务器直接确定客户端是否已从错误中恢复的机制。例如，假设某台存储设备与客户端发生网络分区，客户端向元数据服务器报告了错误；之后网络分区被修复，所有副本都成功更新了。客户端没有任何机制来报告这一事实，元数据服务器只能被迫在整个镜像范围内修复该文件。

如果客户端支持 NFSv4.2，它可以使用 LAYOUTERROR 和 LAYOUTRETURN 向元数据服务器提供关于恢复工作的提示。针对某文件的 LAYOUTERROR 表示非致命错误。随后不带 ff_ioerr4 的 LAYOUTRETURN 表明客户端已成功将 I/O 重放到所有镜像。任何带 ff_ioerr4 的 LAYOUTRETURN 都是元数据服务器需要修复的错误。如果元数据服务器判定需要开始修复该文件，客户端必须为 LAYOUTERROR 触发 CB_LAYOUTRECALL 做好准备。

#### 8.2.4. 处理写入 COMMIT

当对元数据服务器或单个副本执行稳定写入（stable writes）时（如果 FF_FLAGS_WRITE_ONE_MIRROR 的用法允许这样做），负责接收的节点有义务在回复客户端之前，将已写入的数据稳定地传播出去。

在对应的不稳定写入（unstable writes）情况下，接收节点没有此类义务，但它可以选择异步地传播更新。然而，一旦对 COMMIT 作出响应，所有副本都必须反映出已完成的写入，并且这些数据必须已在所有副本上提交到稳定存储。

为避免从尚未传播写入的副本中读到过期数据的情况：

- 已向单个节点（元数据服务器或存储设备）发出未完成的不稳定写入的客户端，必须始终从同一节点进行所有读取。
- 当写入被刷新到服务器时（例如为实现 close-to-open 语义），客户端必须执行 COMMIT，以确保无论读取的是哪个特定副本，都能获得最新的已写入数据。

### 8.3. 元数据服务器对文件的重新同步重建（Resilvering）

元数据服务器可以随时决定为布局段创建一个新镜像。这可能是为了在因维护而宕机的存储设备上重建副本，为了在具有不同存储性能特性的存储上提供布局段的副本等。由于客户端不会知晓这个新镜像，且元数据服务器也不会知晓客户端正在对布局段进行的更新，元数据服务器必须召回它正在重建的可写布局段。如果客户端对正在重建中的可写布局段发出 LAYOUTGET，元数据服务器可以用 NFS4ERR_LAYOUTUNAVAILABLE 拒绝该请求。随后客户端将不得不经由元数据服务器执行 I/O。

## 9. 灵活文件布局类型返回

layoutreturn_file4 用于在 LAYOUTRETURN 操作中向服务器传达布局类型特定的信息。它在 [RFC5661] 第 18.44.1 节中定义如下：

```xdr
   /* Constants used for LAYOUTRETURN and CB_LAYOUTRECALL */
   const LAYOUT4_RET_REC_FILE      = 1;
   const LAYOUT4_RET_REC_FSID      = 2;
   const LAYOUT4_RET_REC_ALL       = 3;

   enum layoutreturn_type4 {
           LAYOUTRETURN4_FILE = LAYOUT4_RET_REC_FILE,
           LAYOUTRETURN4_FSID = LAYOUT4_RET_REC_FSID,
           LAYOUTRETURN4_ALL  = LAYOUT4_RET_REC_ALL
   };

struct layoutreturn_file4 {
        offset4         lrf_offset;
        length4         lrf_length;
        stateid4        lrf_stateid;
        /* layouttype4 specific data */
        opaque          lrf_body<>;
};

union layoutreturn4 switch(layoutreturn_type4 lr_returntype) {
        case LAYOUTRETURN4_FILE:
                layoutreturn_file4      lr_layout;
        default:
                void;
};

struct LAYOUTRETURN4args {
        /* CURRENT_FH: file */
        bool                    lora_reclaim;
        layouttype4             lora_layout_type;
        layoutiomode4           lora_iomode;
        layoutreturn4           lora_layoutreturn;
};
```

如果 lora_layout_type 布局类型是 LAYOUT4_FLEX_FILES 且 lr_returntype 是 LAYOUTRETURN4_FILE，则 lrf_body opaque 值由 ff_layoutreturn4 定义（参见第 9.3 节）。这使客户端能够按如下定义向元数据服务器报告 I/O 错误信息或布局使用统计信息。注意，虽然这些数据结构建立在 NFSv4.2 引入的概念之上，但有效的可判别联合（lora_layout_type 与 ff_layoutreturn4 的组合）允许 NFSv4.1 元数据服务器使用这些数据。

### 9.1. I/O 错误报告

#### 9.1.1. ff_ioerr4

```xdr
struct ff_ioerr4 {
        offset4        ffie_offset;
        length4        ffie_length;
        stateid4       ffie_stateid;
        device_error4  ffie_errors<>;
};
```

回顾 [RFC7862] 中对 device_error4 的定义：

```xdr
struct device_error4 {
        deviceid4       de_deviceid;
        nfsstat4        de_status;
        nfs_opnum4      de_opnum;
};
```

ff_ioerr4 结构用于返回在数据传输期间产生错误的数据文件的错误指示。它们是给元数据服务器的提示，表明该文件存在问题。对于每个错误，ffie_errors.de_deviceid、ffie_offset 和 ffie_length 表示发生错误的存储设备以及文件内的字节范围；ffie_errors 表示操作类型和错误类型。device_error4 的用法在 [RFC7862] 第 15.6 节中描述。

即使存储设备可能通过 NFSv3 访问并向客户端返回 NFSv3 错误，客户端也要负责将这些错误映射为适当的 NFSv4 状态码作为 de_status。同样，NFSv3 操作也需要映射为等价的 NFSv4 操作。

### 9.2. 布局使用统计

#### 9.2.1. ff_io_latency4

```xdr
struct ff_io_latency4 {
        uint64_t       ffil_ops_requested;
        uint64_t       ffil_bytes_requested;
        uint64_t       ffil_ops_completed;
        uint64_t       ffil_bytes_completed;
        uint64_t       ffil_bytes_not_delivered;
        nfstime4       ffil_total_busy_time;
        nfstime4       ffil_aggregate_completion_time;
};
```

操作计数和传输字节数都记录在 ff_io_latency4 中。如 ff_layoutupdate4 所示（参见第 9.2.2 节），READ 和 WRITE 操作分别聚合。READ 操作用于 ff_io_latency4 的 ffl_read；WRITE 和 COMMIT 操作都用于 ff_io_latency4 的 ffl_write。"requested"（已请求）计数器跟踪客户端尝试执行的操作，"completed"（已完成）计数器跟踪实际完成的操作。客户端只报告与报告周期内已请求结果相匹配的已完成结果，这不是必须的。

ffil_bytes_not_delivered 用于跟踪因错误状况而请求但未完成的字节总数。ffil_total_busy_time 是有未完成的 RPC 调用的时间总和。ffil_aggregate_completion_time 是已完成 RPC 调用所有往返时间的总和。

在 [RFC5661] 第 3.3.1 节中，nfstime4 被定义为自 1970 年 1 月 1 日零时（UTC 协调世界时）午夜起经过的秒数和纳秒数。在 ff_io_latency4 中使用 nfstime4 是为了存储客户端收到布局后、从第一个 I/O 开始算起的时间。换句话说，这些值应被解码为时长（duration），而不是日期时间。

注意，LAYOUTSTATS 是累积的，即每次发送操作时不会被重置。如果针对同一文件、同一布局 stateid 的两个 LAYOUTSTATS 操作来自同一 NFS 客户端、且由元数据服务器同时处理，那么包含较大值的那个操作包含最新的时间序列数据。

#### 9.2.2. ff_layoutupdate4

```xdr
struct ff_layoutupdate4 {
        netaddr4       ffl_addr;
        nfs_fh4        ffl_fhandle;
        ff_io_latency4 ffl_read;
        ff_io_latency4 ffl_write;
        nfstime4       ffl_duration;
        bool           ffl_local;
};
```

ffl_addr 用于区分客户端连接到存储设备所使用的网络地址。在多路径情况下，ffl_fhandle 表明选择了哪个只读副本。ffl_read 和 ffl_write 分别传达 READ 和 WRITE 操作的延迟。ffl_duration 用于表明统计信息的收集时间段。如果 ffl_local 为真，表示该 I/O 由客户端的缓存提供服务。该标志允许客户端告知元数据服务器它对某个文件存在"热点"访问，而它平时不被允许报告该文件。

#### 9.2.3. ff_iostats4

```xdr
struct ff_iostats4 {
        offset4           ffis_offset;
        length4           ffis_length;
        stateid4          ffis_stateid;
        io_info4          ffis_read;
        io_info4          ffis_write;
        deviceid4         ffis_deviceid;
        ff_layoutupdate4  ffis_layoutupdate;
};
```

[RFC7862] 将 io_info4 定义为：

```xdr
struct io_info4 {
        uint64_t        ii_count;
        uint64_t        ii_bytes;
};
```

在 pNFS 中，数据传输在 pNFS 客户端与存储设备之间直接进行。因此，元数据服务器不直接知晓正在进行的 I/O 操作，因而无法自行创建关于客户端 I/O 的统计信息来优化数据存储位置。客户端可以使用 ff_iostats4 在返回布局时向元数据服务器报告 I/O 统计信息。

由于客户端报告使用布局的每一个 I/O 并不现实，客户端可以标识出需要报告 I/O 统计信息的"热点"字节范围。"热点"的定义和/或配置机制、以及所报告字节范围的大小都超出本文档的范围。对于客户端实现，建议提供合理的默认值，并提供一个可选的运行时管理接口来控制这些参数。例如，客户端可以定义默认字节范围分辨率为 1 MB，报告阈值为 1 MB/秒或每秒 10 个 I/O 操作。

对于每个字节范围，ffis_offset 和 ffis_length 分别表示该范围的起始偏移和以字节为单位的长度。ffis_read.ii_count、ffis_read.ii_bytes、ffis_write.ii_count 和 ffis_write.ii_bytes 分别表示在报告的字节范围内连续的 READ 和 WRITE I/O 数量以及相应的传输字节总数。

ffis_deviceid 与 ffl_addr 的组合唯一地标识了存储路径以及通往它的网络路由。最后，ffl_fhandle 允许元数据服务器区分同一存储设备上同一文件的多个只读副本。

### 9.3. ff_layoutreturn4

```xdr
struct ff_layoutreturn4 {
        ff_ioerr4     fflr_ioerr_report<>;
        ff_iostats4   fflr_iostats_report<>;
};
```

当数据文件的 I/O 操作失败时，fflr_ioerr_report<> 用于以 ff_ioerr4 类型的元素数组的形式将这些错误报告给元数据服务器。数组中的每个元素表示由 ffie_errors.de_deviceid 标识的数据文件上发生的一个错误。如果没有要报告的错误，则 fflr_ioerr_report<> 数组的大小设置为零。客户端还可以使用 fflr_iostats_report<> 以 ff_iostats4 类型的元素数组的形式报告一组 I/O 统计信息。数组中的每个元素表示某个特定字节范围的统计信息。字节范围不保证互不重叠，它们可能重复或相交。

## 10. 灵活文件布局类型 LAYOUTERROR

如果客户端使用 NFSv4.2 与元数据服务器通信，那么它不必等到 LAYOUTRETURN 时才向元数据服务器发送错误信息（参见第 9.1 节），而是可以使用 LAYOUTERROR（参见 [RFC7862] 第 15.6 节）来传达该信息。对于灵活文件布局类型，这意味着 LAYOUTERROR4args 的处理方式与 ff_ioerr4 相同。

## 11. 灵活文件布局类型 LAYOUTSTATS

如果客户端使用 NFSv4.2 与元数据服务器通信，那么它不必等到 LAYOUTRETURN 时才向元数据服务器发送 I/O 统计信息（参见第 9.2 节），而是可以使用 LAYOUTSTATS（参见 [RFC7862] 第 15.7 节）来传达该信息。对于灵活文件布局类型，这意味着 LAYOUTSTATS4args.lsa_layoutupdate 被重载为与 ffis_layoutupdate 相同的内容。

## 12. 灵活文件布局类型创建提示

layouthint4 类型在 [RFC5661] 中定义如下：

```xdr
struct layouthint4 {
    layouttype4        loh_type;
    opaque             loh_body<>;
};
```

客户端使用 layouthint4 结构来传递提示，表明它希望为某个特定文件创建何种类型的布局。如果 loh_type 布局类型是 LAYOUT4_FLEX_FILES，则 loh_body opaque 值由 ff_layouthint4 类型定义。

### 12.1. ff_layouthint4

```xdr
union ff_mirrors_hint switch (bool ffmc_valid) {
    case TRUE:
        uint32_t    ffmc_mirrors;
    case FALSE:
        void;
};

struct ff_layouthint4 {
    ff_mirrors_hint    fflh_mirrors_hint;
};
```

该类型传达对期望数据映射的提示。所有参数都是可选的，因此客户端可以只给出它关心的参数的值。

## 13. 召回布局

虽然 [RFC5661] 第 12.5.5 节讨论了独立于布局类型的召回布局的理由，但灵活文件布局类型的元数据服务器应在以下情况下召回尚未完成的布局：

- 当文件的安全策略发生变化时，即设置 ACL 或权限模式位时。
- 当文件的布局发生变化，导致尚未完成的布局失效时。
- 当现有布局与强制执行锁约束的需要不一致时。
- 当现有布局与第 8.3 节所述关于重新同步重建的需求不一致时。

### 13.1. CB_RECALL_ANY

元数据服务器可以使用 CB_RECALL_ANY 回调操作通知客户端返回其部分或全部布局。[RFC5661] 第 22.3 节定义了"NFSv4 可回收对象类型注册表"（NFSv4 Recallable Object Types Registry）允许的类别。

```xdr
const RCA4_TYPE_MASK_FF_LAYOUT_MIN     = 16;
const RCA4_TYPE_MASK_FF_LAYOUT_MAX     = 17;

struct  CB_RECALL_ANY4args      {
    uint32_t        craa_layouts_to_keep;
    bitmap4         craa_type_mask;
};
```

通常，CB_RECALL_ANY 用于在服务器需要回收资源时召回客户端状态。craa_type_mask 位图指定被召回的资源类型，craa_layouts_to_keep 值指定允许客户端保留多少个被召回的灵活文件布局。灵活文件布局类型的掩码标志定义如下：

```xdr
enum ff_cb_recall_any_mask {
    PNFS_FF_RCA4_TYPE_MASK_READ = 16,
    PNFS_FF_RCA4_TYPE_MASK_RW   = 17
};
```

这些标志表示被召回布局的 iomode。作为响应，客户端应返回其所持 iomode 为被召回类型的布局中自己最不需要的那些，保留的灵活文件布局数量不超过 craa_layouts_to_keep 个。

PNFS_FF_RCA4_TYPE_MASK_READ 标志通知客户端返回 iomode 为 LAYOUTIOMODE4_READ 的布局。类似地，PNFS_FF_RCA4_TYPE_MASK_RW 标志通知客户端返回 iomode 为 LAYOUTIOMODE4_RW 的布局。当两个掩码标志都被设置时，通知客户端返回任一种 iomode 的布局。

## 14. 客户端隔离

当客户端失去通信能力且其租约已到期，或客户端未能在租约期内返回被召回的布局时，服务器可以撤销客户端的布局，并将这些资源重新分配给其他客户端（参见 [RFC5661] 第 12.5.5 节）。为避免数据损坏，元数据服务器必须按第 2.2 节所述，将已被撤销的客户端与相应的数据文件隔离。

## 15. 安全考虑

pNFS 系统中各组件的组合必须保持 NFSv4.1+ 的安全属性，以应对经由客户端访问数据的实体。pNFS 特性将 NFSv4.1+ 文件系统协议划分为两部分：控制协议和数据协议。由于本文档中的控制协议就是 NFS，其安全属性与所使用的 NFS 版本等同。灵活文件布局进一步将数据协议划分为元数据通路和数据通路。元数据通路的安全属性与 NFSv4.1x 等同（参见 [RFC5661] 第 1.7.1 和 2.2.1 节）。数据通路的安全属性与用于访问存储设备的 NFS 版本等同，但有一项附加规定：元数据服务器负责对客户端访问数据文件进行身份认证。元数据服务器向客户端提供适当的凭据，用于访问存储设备上的数据文件。它还负责撤销客户端对存储设备的访问权限。

元数据服务器在 LAYOUTGET 时强制执行文件访问控制策略。客户端应使用 RPC 授权凭据来获取所请求 iomode（LAYOUTIOMODE4_READ 或 LAYOUTIOMODE4_RW）的布局，服务器验证这些凭据的权限和 ACL，如果客户端不被允许所请求的 iomode，则可能返回 NFS4ERR_ACCESS。如果 LAYOUTGET 操作成功，客户端将作为布局的一部分收到一组凭据，允许它对与所请求 iomode 相对应的指定数据文件进行 I/O 访问。当客户端代表其本地用户执行 I/O 操作时，它必须通过向元数据服务器发出相应的 OPEN 和 ACCESS 调用来对用户进行身份认证和授权，这类似于持有 NFSv4 数据委托的情况。

布局中文件句柄、合成 uid 和 gid 三者的组合，是元数据服务器对数据服务器强制执行访问控制的方式。客户端只能访问文件对象的文件句柄，而不能访问目录对象的。因此，给定布局中的一个文件句柄，无法猜出其父目录的文件句柄。此外，由于数据文件的权限只允许给定的合成 uid 读/写权限、给定的合成 gid 读权限，知道一个文件的合成 id 并不一定允许访问存储设备上的任何其他数据文件。

元数据服务器还可以随时通过隔离数据文件来拒绝访问，这意味着更改合成 id。反过来，这会迫使客户端返回其当前布局，并在想要继续对该数据文件进行 I/O 时获取一个新布局。

如果允许访问，客户端将使用相应的（只读或读/写）凭据，在数据文件的存储设备上执行 I/O 操作。当元数据服务器收到更改文件权限或 ACL 的请求时，它应召回该文件的所有布局，然后必须通过隐式地使先前下发的凭据在构成该文件的所有数据文件上失效，来隔离仍然持有相应文件未完成布局的任何客户端。规定必须在提交新权限和/或 ACL 之前完成此操作。通过请求新布局，客户端将根据修改后的访问控制元数据重新授权访问。在这种情况下召回布局，是为了防止客户端在被隔离后执行 I/O 时收到错误。

### 15.1. RPCSEC_GSS 与安全服务

由于在松耦合模型中对主体（principal）的特殊使用，相关问题因耦合模型而异。

#### 15.1.1. 松耦合

RPCSEC_GSS 版本 3（RPCSEC_GSSv3）[RFC7861] 包含一些机制，允许它代表元数据服务器对客户端到存储设备的访问进行授权。要做到这一点，要求元数据服务器、存储设备和客户端各自都按照 [RFC7862] 第 4.9.1 节所述的方式，使用由 RPC 应用程序定义的结构化特权断言来实现 RPCSEC_GSSv3。实现这一目标所需的具体细节不在本文档中描述。这主要是因为任何此类规范都需要在广泛的各类存储设备上进行大量实现工作，这在相当长的时间内都不太可能产生一个可广泛使用的规范。

因此，本文档所述的布局类型将不支持在松耦合模型下使用 RPCSEC_GSS。但是，未来可以定义允许这种支持的布局类型——既可以通过使用 RPCSEC_GSSv3，也可以通过其他方式。

#### 15.1.2. 紧耦合

在紧耦合情况下，用于访问元数据文件的主体与用于访问数据文件的主体完全相同。存储设备可以使用控制协议来验证任何 RPC 凭据。因此，在紧耦合系统中使用 RPCSEC_GSS 不存在安全问题。例如，如果使用 Kerberos V5 通用安全服务应用程序接口（GSS-API）[RFC4121] 作为安全机制，那么存储设备可以使用控制协议向元数据服务器验证 RPC 凭据。

## 16. IANA 考虑

[RFC5661] 引入了"pNFS 布局类型注册表"（pNFS Layout Types Registry）；该注册表中的新布局类型号需要由 IANA 分配。本文档定义了与一个现有布局类型号相关联的协议：LAYOUT4_FLEX_FILES。见表 1。

| 布局类型名称 | 值 | RFC | 方式 | 次版本 |
| --- | --- | --- | --- | --- |
| LAYOUT4_FLEX_FILES | 0x00000004 | RFC 8435 | L | 1 |

表 1：布局类型分配

[RFC5661] 还引入了"NFSv4 可回收对象类型注册表"。本文档为 RCA4_TYPE_MASK_FF_LAYOUT_MIN 和 RCA4_TYPE_MASK_FF_LAYOUT_MAX 定义了新的可回收对象（见表 2）。

| 可回收对象类型名称 | 值 | RFC | 方式 | 次版本 |
| --- | --- | --- | --- | --- |
| RCA4_TYPE_MASK_FF_LAYOUT_MIN | 16 | RFC 8435 | L | 1 |
| RCA4_TYPE_MASK_FF_LAYOUT_MAX | 17 | RFC 8435 | L | 1 |

表 2：可回收对象类型分配

## 17. 参考文献

### 17.1. 规范性参考文献

- [LEGAL] IETF Trust，《Trust 法律条款（TLP）》，<https://trustee.ietf.org/trust-legal-provisions.html>。
- [RFC1813] Callaghan, B., Pawlowski, B., and P. Staubach, "NFS Version 3 Protocol Specification", RFC 1813, DOI 10.17487/RFC1813, June 1995, <https://www.rfc-editor.org/info/rfc1813>.
- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997, <https://www.rfc-editor.org/info/rfc2119>.
- [RFC4121] Zhu, L., Jaganathan, K., and S. Hartman, "The Kerberos Version 5 Generic Security Service Application Program Interface (GSS-API) Mechanism: Version 2", RFC 4121, DOI 10.17487/RFC4121, July 2005, <https://www.rfc-editor.org/info/rfc4121>.
- [RFC4506] Eisler, M., Ed., "XDR: External Data Representation Standard", STD 67, RFC 4506, DOI 10.17487/RFC4506, May 2006, <https://www.rfc-editor.org/info/rfc4506>.
- [RFC5531] Thurlow, R., "RPC: Remote Procedure Call Protocol Specification Version 2", RFC 5531, DOI 10.17487/RFC5531, May 2009, <https://www.rfc-editor.org/info/rfc5531>.
- [RFC5661] Shepler, S., Ed., Eisler, M., Ed., and D. Noveck, Ed., "Network File System (NFS) Version 4 Minor Version 1 Protocol", RFC 5661, DOI 10.17487/RFC5661, January 2010, <https://www.rfc-editor.org/info/rfc5661>.
- [RFC5662] Shepler, S., Ed., Eisler, M., Ed., and D. Noveck, Ed., "Network File System (NFS) Version 4 Minor Version 1 External Data Representation Standard (XDR) Description", RFC 5662, DOI 10.17487/RFC5662, January 2010, <https://www.rfc-editor.org/info/rfc5662>.
- [RFC7530] Haynes, T., Ed. and D. Noveck, Ed., "Network File System (NFS) Version 4 Protocol", RFC 7530, DOI 10.17487/RFC7530, March 2015, <https://www.rfc-editor.org/info/rfc7530>.
- [RFC7861] Adamson, A. and N. Williams, "Remote Procedure Call (RPC) Security Version 3", RFC 7861, DOI 10.17487/RFC7861, November 2016, <https://www.rfc-editor.org/info/rfc7861>.
- [RFC7862] Haynes, T., "Network File System (NFS) Version 4 Minor Version 2 Protocol", RFC 7862, DOI 10.17487/RFC7862, November 2016, <https://www.rfc-editor.org/info/rfc7862>.
- [RFC8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174, May 2017, <https://www.rfc-editor.org/info/rfc8174>.
- [RFC8434] Haynes, T., "Requirements for Parallel NFS (pNFS) Layout Types", RFC 8434, DOI 10.17487/RFC8434, August 2018, <https://www.rfc-editor.org/info/rfc8434>.

### 17.2. 资料性参考文献

- [RFC4519] Sciberras, A., Ed., "Lightweight Directory Access Protocol (LDAP): Schema for User Applications", RFC 4519, DOI 10.17487/RFC4519, June 2006, <https://www.rfc-editor.org/info/rfc4519>.

## 致谢

以下个人为本文档的早期草案版本提供了各种意见：Matt W. Benjamin、Adam Emerson、J. Bruce Fields 和 Lev Solomonov。

以下个人为本文档的最终草案版本提供了各种意见：Anand Ganesh、Robert Wipfel、Gobikrishnan Sundharraj、Trond Myklebust、Rick Macklem 和 Jim Sermersheim。

Idan Kedar 发现了客户端侧镜像与设备次版本化交互中的一个严重缺陷。

Dave Noveck 在工作组最后征求意见阶段对本文档进行了全面评审，并重写了第 2.3 节。

Olga Kornievskaia 对隔离方法中使用凭据（credential）而非主体（principal）的做法提出了有说服力的反对意见。Andy Adamson 和 Benjamin Kaduk 帮助厘清了关注点。

Benjamin Kaduk 和 Olga Kornievskaia 还为松耦合安全机制提供了具体的场景。最终，Olga 证明：按其定义，松耦合模型将无法与 RPCSEC_GSS 协同工作。

Tigran Mkrtchyan 提供了不允许客户端经由数据服务器代理 I/O 的用例。

Rick Macklem 提供了只写入单个镜像的用例。

## 作者地址

**Benny Halevy**

邮箱：bhalevy@gmail.com

**Thomas Haynes**
Hammerspace
4300 El Camino Real Ste 105
Los Altos, CA 94022
美利坚合众国

邮箱：loghyr@gmail.com

---

*译者注：本文为 RFC 8435 的非官方中文翻译，仅供参考。如有歧义，请以 IETF 发布的英文原文为准。*
