<properties
	pageTitle="Azure SNAT"
	description="Azure SNAT 详解及实际应用分析"
	service=""
	resource="virtualnetworks"
	authors=""
	displayOrder=""
	selfHelpType=""
	supportTopicIds=""
	productPesIds=""
	resourceTags="Virtual Networks, SNAT, DIP, VIP, PIP"​
	cloudEnvironments="MoonCake" />
<tags 
	ms.service="virtual-networks-aog"
	ms.date=""
	wacn.date="02/07/2017" />
# Azure SNAT

这篇文章由 Pedro Perez 提供。

Azure 的网络基础设施和通常的本地网络是完全不同的，因为在后台运行的是不同的软件抽象层。我今天想要讲述的是这些软件层中的某一层，以及为什么在对您的应用程序进行网络故障排除时需要考虑到这点。

在云端对于我们及我们的客户最大的挑战可能是构建和扩展。我们的工程师已经将 Azure 设计成适用于超大规模并同时满足技术要求最为苛刻的用户。正如您所想象，这增加了系统的复杂性。例如，有多个 IP 地址与每个虚拟机相关联。在非 Azure 资源管理(非 ARM)的场景下，当您在云服务中部署虚拟机时，虚拟机获取一个 IP 地址分配，这个 IP 被称为动态 IP （DIP），该 IP 在 Azure 外不可路由。云服务获取一个被称为虚拟 IP（VIP）的 IP 地址分配，这是一个可路由的公共 IP 地址。当发送出站流量时在云服务中的每一个虚拟机将会被隐藏在 VIP 之后，并且只能通过在 VIP 上创建映射到该指定虚拟机的终结点来进行访问。

类似于传统的本地网络，VIP 可以被定义为 NAT IP 地址。VIP 的最大特殊性是在相同的云服务中所有虚拟机共享该 VIP。您可以通过利用云服务中的终结点很容易地控制端口的数据流量重定向到特定的虚拟机，但是对于出站流量该如何转换？

## 源 NAT

这就是源 NAT（以下用 SNAT 代替）发挥作用之处。任何云服务的出站流量（从云服务中的一台虚拟机传出并且不传入到相同云服务的另一台虚拟机的该部分流量）将会通过 NAT 层，它将会得到SNAT应用。顾名思义，SNAT 只更改源信息：源 IP 地址和源端口。

## 源 IP 地址转换

源 IP 地址从原始的 DIP 转变为云服务的 VIP，所以流量能够很容易被路由。这是一个多对一的映射，所有云服务中的虚拟机将被转变为该云服务的 VIP。在这一点上我们已经面临一个挑战。如果在相同云服务中的两个虚拟机创建了一个到相同目标的出站连接并且也使用相同的源端口将会发生什么状况？请记住，系统将通过在源 IP、源端口、目标 IP和目标端口的4元组中寻找以区分不同的 TCP（或 UDP）连接。

更改任何目标信息将有效的断开连接，我们无法改变源 IP，因为我们只有一个（ VIP）。因此我们需要改变源端口。

## 源端口转换

Azure 在 VIP 中为虚拟机的连接预分配了 160 个源端口。这种预分配是为了加快建立新的通信，并且限制了 160 个端口以节省资源。初始端口是从该大范围的端口中随机选择的， 然后对其以及随后的 159 个端口进行预分配。我们发现这些设置都会起作用只要我们在开发应用程序时在网络设计部分遵循一些最佳做法。


Azure 将把一个出站连接的源端口转换为 160 个预分配端口中的第一个可用端口。

第一个您可能想要问的是如果我们同时使用 160 个端口将会发生什么？那么，如果任何一个端口都没有被释放则系统会尽最大努力分配更多的端口，只要有一个端口被释放了，它将再次可供使用。

## SNAT 表

所有这些转换必须进行存储，以便跟踪流入和流出我们虚拟机的数据包。这些用于存储的地方被称为 SNAT 表，它和那些您在任何其他网络产品如防火墙或路由器中发现的 NAT 表具有相同的概念。

系统会保存原始的5元组（源 IP、源端口、目标 IP、目标端口、协议 – TCP/UDP）和转换后的 5 元组，其中源 IP 、源端口已被转换为 VIP 和一个预分配的端口。

## 删除 SNAT 表中记录

在任何其他 NAT 表中，您无法永远存储这些记录，应该有删除记录的规则，其中最明显的是：

-	如果连接在 FIN、ACK 状态下被关闭，则我们会等待几分钟（2 x MSL（Maximum Segment Lifetime）- [http://www.rfc-editor.org/rfc/rfc793.txt](http://www.rfc-editor.org/rfc/rfc793.txt)）再删除该记录。
-	如果连接在 RST 状态下被关闭，则我们会立刻删除该记录。

此时，我相信您已经发现了一个问题。我们如何决定删除一个 配对消失（例如，程序崩溃或停止响应）的 UDP 连接或 TCP 连接。

在这种情况下，我们已经硬编码了一个的超时值。每当一个特定的连接数据包经过 SNAT 的过程中，我们在该连接上启动一个 4 分钟的倒计时。如果在另一个数据包到达之前计时器归零了，则我们从 SNAT 表中删除该记录，因为我们认为这个连接已经完成。这是非常重要的一点：如果您的应用程序保持一个空闲连接超过 4 分钟，那么这条记录将会从表中删除。大多数应用程序不会处理丢失连接，他们认为仍处于活跃状态，所以明智的管理您连接的生命周期以及不让连接进入空闲状态是需要非常谨慎的。

## 长期闲置的连接被认为是有害的

有时给个例子更容易说明一个复杂的状况，所以此处举例说明会发生什么错误以及为什么您不应该保持闲置的 TCP 连接。以下展示一个在客户端（云服务中的虚拟机）活跃的 HTTP 连接、SNAT 以及服务端的连接表：

### 客户端

| 源 IP	 	| 源端口	| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:----:	|:---------:|:-------:	|:-------:	|
| 客户端 DIP	| 12345	| 服务端 VIP	| 80		| 连接成功 	|

源端口是由客户端操作系统随机选择的。

### SNAT 表

| 源 IP	 	| 源端口				| 目标 IP 	| 目标端口	|
|:--------:	|:---------------:	|:---------:|:--------:	|
| DIP-> VIP	| 12345 -> 54321	| 服务端 IP	| 80		|

DIP 被转换为 VIP 并且源端口被转换为160 个预分配端口中的第一个可用的端口。

### 服务端

| 源 IP	 	| 源端口	| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:----:	|:---------:|:--------:	|:--------：	|
| 客户端 VIP	| 54321	| 服务端 IP	| 80		| 连接成功 	|

服务端不知道客户端 DIP 或者源端口，因为这些都被 SNAT 隐藏在 VIP 之后。

目前为止，一切都很好。

现在设想连接已经闲置超过了 4 分钟，结果在表中将是什么状态？

### 客户端

| 源 IP	 	| 源端口	| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:----:	|:---------:|:-------:	|:-------:	|
| 客户端 DIP	| 12345	| 服务端 VIP	| 80		| 连接成功 	|

此处没有改变。客户端为更多需要的数据已建立好连接，但是没有数据从服务端传来。

### SNAT 表

| 源 IP	 	| 源端口		| 目标 IP	| 目标端口	|
|:--------:	|:--------:	|:---------:|:--------:	|
| **已删除**	| **已删除**	| **已删除**	| **已删除**	|

此处发生了什么？

SNAT 表中记录已过期因为该连接已闲置超过了 4 分钟，所以 SNAT 表中记录被删除。

### 服务端

| 源 IP	 	| 源端口	| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:----:	|:---------:|:--------:	|:--------：	|
| 客户端 VIP	| 54321	| 服务端 IP	| 80		| 连接成功 	|

不出所料，在服务器上没有任何改变。 它返回了客户端请求的所有数据并且 4 分钟以后在该 TCP 连接上等待新的请求。

这样问题就出现了。假设客户端恢复其操作并决定使用相同的连接从服务端请求更多数据。不幸的是，这将不会起作用因为这些数据包不符合以下任何标准， Azure 将会在 SNAT 层丢弃这部分流量。

-	属于现有连接（不满足该条件，因为它已经过期，所以会被删除！）
-	SYN 数据包（对于新连接而言）（不满足该条件，因为它不是一个 SYN 数据包！）

这意味着在这个多元组上尝试的连接都将失败。好吧，这是一个问题，但是其实并不是无法解决的，对吧？客户端将建立一个新的 TCP 连接（即 SYN 数据包将通过 SNAT）并在其中发送 HTTP 请求。这是正确的，但是有些情况下我们可能会面临的另一个后果，SNAT 记录到期。

如果客户端与同一个服务端建立新的连接并且端口（服务器 IP : 80）足够快的在 160 个分配好的端口中循环（或者更快），但并没有显式地关闭他们， 端口 `54321` 就可以再次使用（记住：端口 `12345` --> `54321` 的转换将过期），我们将可以轻易的在原始的 160 个端口中随意使用。迟早端口 `54321` 将会与 `12345` 以外的源端口进行转换从而再次被使用，但对于相同的源和目标 IP 地址（以及相同的目标端口！），它将会是如下状态：


### 客户端

| 源 IP	 	| 源端口		| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:--------:	|:---------:|:-------:	|:-------:	|
| 客户端 DIP	| 1234**6**	| 服务端 VIP	| 80		| 连接成功 	|

客户端决定建立新的连接，所以它会发送一个 SYN 数据包从而在服务端 IP : `80` 建立一个新的连接。

### SNAT 表

| 源 IP	 	| 源端口					| 目标 IP 	| 目标端口	|
|:--------:	|:-------------------:	|:---------:|:--------:	|
| DIP-> VIP	| 1234**6** -> 54321	| 服务端 IP	| 80		|

Azure 发现这个数据包在表中没有匹配记录，会将其作为 SYN 数据包接收（即新连接）。将端口 `12346` 转换为 `54321` 因为这在 160 个分配好的端口中又是第一个可用端口。

### 服务端

| 源 IP	 	| 源端口	| 目标 IP 	| 目标端口	| TCP 状态	|
|:--------:	|:----:	|:---------:|:--------:	|:--------：	|
| 客户端 VIP	| 54321	| 服务端 IP	| 80		| 连接成功 	|

服务端已经建立好了连接，所以当它从客户端 VIP : `54321` 接收到一个 SYN 数据包时将忽略并且丢弃它。此时，我们已经终结了两个断开的连接：被闲置 4 分钟以上的原连接和新连接。

避免这个问题的最好方式实际上是在不同平台上让很多问题在应用层面有一个合理的生存周期（[SO_KEEPALIVE 套接字选项](https://msdn.microsoft.com/en-us/library/windows/desktop/ee470551(v=vs.85).aspx)） 。应该考虑每 30 秒或 1 分钟通过空闲连接发送一个数据包，因为它会在 Azure 和本地防火墙中重置每一个用于统计空闲时间的计时器。

## 需要快速的解决方法？

在 Azure 中您可以使用一个快速的解决方法。对于出站流量（同样对于入站流量）您可以避免使用 VIP，而是指定一个实例级 IP ，称为 PIP。PIP 只分配给一个实例， 因此不需要使用 SNAT 以适应不同虚拟机的请求。它仍然通过 软件负载平衡（SNAT），但并没有 SNAT 应用，也没有 SNAT 表，您可以随意的保持您的空闲连接直到 SLB 终结它们（[Azure 负载均衡器的可配置空闲超时](http://azure.microsoft.com/blog/2014/08/14/new-configurable-idle-timeout-for-azure-load-balancer/)），但这将是另一种情况。

在结束这篇文章之前，我们也应该承认这种设计的另一个长期存在的问题。由于 Azure 在这一批的 160 个端口中分配出站端口，有可能创建一批新的 160 个端口不够快，从而导致出站连接的尝试将会失败。我们通常只在非常高的负荷情况下才会看到（几乎总是在负载测试阶段），但如果您真的遇到了这种情况，请尝试使用 PIP 这个解决方案。