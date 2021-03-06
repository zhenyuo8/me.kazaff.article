今天开始打算了解一下Dubbo框架，主要关注它的“服务动态寻址与路由，依赖分析”等SOA治理解决方案，这也和我所在的公司目前正在攻克的问题方向一致，刚好可以好好借鉴一下，希望能得到启发！

先来看一下Dubbo使用的场景：

	当网站变大后，不可避免的需要拆分应用进行服务化，以提高开发效率，调优性能，节省关键竞争资源等。
	当服务越来越多时，服务的URL地址信息就会爆炸式增长，配置管理变得非常困难，F5硬件负载均衡器的单点压力也越来越大。
	当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在哪个应用之前启动，架构师都不能完整的描述应用的架构关系。
	接着，服务的调用量越来越大，服务的容量问题就暴露出来，这个服务需要多少机器支撑？什么时候该加机器？等等…… 
	
	在遇到这些问题时，都可以用Dubbo来解决。 

那Dobbo的性能又如何呢：

	Dubbo通过长连接减少握手，通过NIO及线程池在单连接上并发拼包处理消息，通过二进制流压缩数据，比常规HTTP等短连接协议更快。
	在阿里巴巴内部，每天支撑2000多个服务，30多亿访问量，最大单机支撑每天近1亿访问量。 

更多的介绍，可以查看[这里](http://www.iteye.com/magazines/103)。

非常钦佩的是Dubbo团队的开源精神，甚至是文档，都非常的详细，值得点赞！！

需求演化
---

![](http://pic.yupoo.com/kazaff_v/DTd8g4qX/CWACu.jpg)

* 单一应用架构
	* 当网站流量很小时，只需要一个应用，将所有功能都部署在一起，以减少部署节点和成本
	* 此时，数据访问框架（ORM）是关键
* 垂直应用架构
	* 当访问量逐渐增大，单一应用增加机器带来的加速越来越小，需要将应用拆成互不相干的几个应用，以提升效率
	* 此时，用于加速前端页面开发的web框架（MVC）是关键
* 分布式服务架构
	* 当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取独立作为服务，逐渐形成服务中心以快速响应多变的市场需求
	* 此时，用于提高业务复用及整合的分布式服务框架（RPC）是关键
* 流动计算框架
	* 当服务越来越多，容量估计，小服务资源的浪费问题逐渐显现，需要增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率
	* 此时，用于提高机器利用率的资源调度和治理中心（SOA）是关键



在大规模服务化之前，应用可能只是通过RMI简单的暴漏和引用远程服务，通过配置服务的URL地址进行调用，通过F5等硬件进行负载均衡。

当服务越来越多时，服务URL配置管理变得困难，F5硬件负载均衡器的单点压力越来越大，此时就需要一个服务注册中心，动态的注册和发现服务，使服务的位置透明，并通过在消费方获取服务提供方地址列表，实现软负载均衡和Failover。

当进一步发展，服务间依赖关系变得错综复杂，甚至分不清哪个应用要在哪个应用之前启动，这就需要自动画出应用间的依赖关系图，以帮助架构师理清关系。

接着，服务的调用量越来越大，服务的容量问题就暴露出来了，这个服务需要多少机器支撑，什么时候该加机器，为了解决这些问题，就需要将服务每条的调用量，响应时间等数据都统计出来，作为容量规划的参考指标。

![](http://pic.yupoo.com/kazaff_v/DTcNwYoQ/1MQpe.jpg)

架构
---

![](http://pic.yupoo.com/kazaff_v/DTcRH71h/ehNrJ.jpg)

节点角色：

* Provider：服务提供方
* Consumer：服务消费方
* Registry：注册中心
* Monitor：监控中心
* Container：服务运行容器

调用关系：

* 0：服务容器负责启动，加载，运行服务提供者
* 1：服务提供者在启动时，向注册中心注册自己提供的服务
* 2：服务消费者在启动时，向注册中心注册自己所需的服务
* 3：注册中心返回服务提供者地址列表给消费者，如有变更，注册中心将基于长连接推送变更数据给消费者
* 4：服务消费者从提供者地址列表中，给予软负载均衡算法，选一台提供者进行调用，如果调用失败则选另一台调用
* 5：服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

连通性：

* 注册中心负责服务地址的注册和查找，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
* 监控中心负责统计各服务调用次数和执行时间等
* 服务提供者向注册中心注册提供的服务，并汇报调用时间给监控中心（不包含网络开销）
* 服务消费者想注册中心获取服务提供者地址列表，并汇报调用时间到监控中心（包含网络开销）
* 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
* 注册中心通过长连接感知服务提供者的状态，若服务提供者宕机，注册中心将会立即推送事件通知消费者
* 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了服务提供者地址列表
* 注册中心和监控中心都是可选的，服务消费者可直连服务提供者

健壮性：

* 监控中心宕机不影响使用，只丢失部分采样数据
* 数据库宕机后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
* 注册中心对等集群，任意一台死掉，将自动切换
* 注册中心全部宕机，服务提供者和消费者仍可以通过本地缓存通讯
* 服务提供者无状态，任意一台死掉，不影响使用
* 服务提供者全宕机，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

伸缩性：

* 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
* 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者

![](http://pic.yupoo.com/kazaff_v/DTd78Wq9/629Py.jpg)

Xml配置
---

* <dubbo:service />：服务配置，用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心
* <dubbo:reference />：引用配置，用于创建一个远程服务代理，一个引用可以指向多个注册中心
* <dubbo:protocol />：协议配置，用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受
* <dubbo:application />：应用配置，用于配置当前应用信息，不管该应用是提供者还是消费者
* <dubbo:module />：模块配置，用于配置当前模块信息，可选
* <dubbo:registry />：注册中心配置，用于配置连接注册中心相关信息
* <dubbo:monitor />：监控中心配置，用于配置连接监控中心相关信息，可选
* <dubbo:provider />：提供方的缺省值，当ProtocolConfig和ServiceConfig某属性没配置时，采用此缺省值，可选
* <dubbo:consumer />：消费方缺省配置，当ReferenceConfig某属性没有配置时，采用此缺省值，可选
* <dubbo:method />：方法配置，用于ServiceConfig和ReferenceConfig指定方法级的配置信息
* <dubbo:argument />：用于指定方法参数配置

![](http://pic.yupoo.com/kazaff_v/DTeSQq1A/PU7ET.jpg)

上图中以timeout为例，显示了配置的查找顺序，规则如下：

* 方法级优先，接口级次之，全局配置再次之
* 如果级别一样，则消费方优先，提供方次之

其中，服务提供方的配置，通过URL经由注册中心传递给消费方。建议由服务器提供方设置超时，因为一个方法需要执行多长时间，服务提供方更清楚，如果一个消费方同时引用多个服务，就不需要关系每个服务的超时设置了。

启动时检查
---
Dubbo缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止Spring初始化完成，以便上线时能及早发现问题：默认check=true。

但如果你的Spring容器是懒加载的，或者通过API编程延迟引用服务，最好关闭check，否则服务临时不可用时会导致服务消费者抛出异常，拿到null引用。如果check=false，总是会返回引用，当服务恢复时能自动连上。另外也可以打破出现循环依赖的尴尬局面（必须有一方先启动）。

关闭某个服务的启动时检查：
	
	<dubbo:reference interface="com.foo.BarService" check="false" />

关闭所有服务的启动时检查：

	<dubbo:consumer check="false" />

关闭注册中心启动时检查：
	
	<dubbo:registry check="false" />

引用本身缺省是延迟初始化的，只有引用被注入到其它Bean，或被getBean()获取，才会初始化。如果需要饥饿加载，即没有人引用也立即生成动态代理，可以配置：

	<dubbo:reference interface="com.foo.BarService" init="true" />

集群容错
---

![](http://pic.yupoo.com/kazaff_v/DTfTiTDT/pH5uo.jpg)

在集群调用失败时，Dubbo提供了多种容错方案，缺省为failover重试。

上图中各节点的关系：

* Invoker：Provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息
* Directory：代表多个Invoker，可以把它看成List<Invoker>，但不同的是，它的值可能动态变化，比如注册中心推送变更
* Cluster：它将Directory中的多个Invoker伪装成一个Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后重试另一个
* Router：负责从多个Invoker中按路由规则选择出子集，比如读写分离，应用隔离等
* LoadBalance：负责从多个Invoker中选出具体的一个Invoker用于本次调用，选择过程包含负载均衡算法，失败调用后需要重选


....................................................


异步调用
---

基于NIO的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销小。

![](http://pic.yupoo.com/kazaff_v/DUz95xv1/TixfD.jpg)


