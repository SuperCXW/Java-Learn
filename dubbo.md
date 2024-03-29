### dubbo
    > java高性能开源RPC分布式服务框架
- netty高性能
- zk高可用
- dubbo分层
- 参与节点
    - priovider
    - consumer
    - contianer
    - registry
    - monitor
- 核心配置
    - dubbo:service
    - dubbo:reference
    - dubbo:protocol
    - dubbo:application
    - dubbo:module
    - dubbo:registry
    - dubbo:monitor
    - dubbo:provider
    - dubbo:consumer
    - dubbo:method
    - dubbo:argument
- provider配置
    - timeout
    - retries
    - loadbalance
    - actives 最大并发限制
- 启动会检查provider是否正常
- 序列化，默认hessian
- 默认netty 还有mina grizzly
- 高可用
    - 负载均衡
    - 集群容错
    - 服务路由
- 集群容错方案
    - failover Cluster失败自动切换 默认
    - failfast Cluster快速失败 只调用一次
    - failSafe Cluster失败安全 出现异常直接忽略
    - failback Cluster 失败自动恢复 记录失败请求，定时重发
    - Forking Cluster 并行调用多个服务提供者，几个成功就返回
    - Broadcast Cluster 广播逐个调用提供者
    - AvailableCluster 循环每个provider，调用第一个可用的
    - MergeableCluster 按组合并返回结果 ，比如菜单服务，接口一样，但有多种实现，用group区分，现在消费者需从每种group中调用一次返回结果，合并结果返回，这样就可以实现聚合菜单项。
    - MockClusterWrapper 判断是否需要开启 Mock 机制，由 invoke 方法完成。根据 MockInvokersSelector 过滤出对应的 Mock Invoker，由 selectMockInvoker 完成，实际是委托给 MockInvokersSelector 完成路由。执行 MockInvoker，由 doMockInvoke方法完成，实际是委托给 MockInvoker。
    - ZoneAwareCluster 2.7.5 加入的cluster 多注册中心的场景（多地机房）
    > 1.制定优先级  
      2.根据invoker的url上的zonekey做匹配
      3.权重轮训 
      4.选择第一个可用的

- 粘滞连接
    - sticky 执行粘滞连接，适用于有状态的服务

- 默认负载均衡策略
    - 随机 按权重设置随机概率  默认
    - 轮询，按公约后的权重 设置 轮询比例
    - 最少活跃调用数，相同活跃数随机调用 这个最小活跃数，其实就是同一时间，正在处理的请求数
    - 一致性hash 相同的请求，总是发给同一提供者
        - 一致性hash
            > 普通hash算法，在分布式场景下，hash值，映射规则可能会发生变化
            > 满足了平衡性，单调性，分散性
            > 通过一个环来实现，顺时针查找距离这个对象hash值最近的机器， 新机器的加减 自动hash到最近的路径
            > 但是这样的话，还是没解决负载不均衡的问题，通过在环上找一组的虚拟节点，达到对称来解决这个问题
- SPI
    - 未使用java的SPI，重新实现了一下 ExtensionLoader来加载SPI
    - 路径 META-INF/dubbo
- 自适应扩展
    - 在 Dubbo 中，很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance 等。有时，有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。这听起来有些矛盾。拓展未被加载，那么拓展方法就无法被调用（静态方法除外）。拓展方法未被调用，拓展就无法被加载。对于这个矛盾的问题，Dubbo 通过自适应拓展机制很好的解决了。自适应拓展机制的实现逻辑比较复杂，首先 Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类。最后再通过反射创建代理类，整个过程比较复杂。
    - Dubbo 自适应扩展机制解决了在调用扩展接口方法时，扩展类未被加载方法不能调用的问题。
    - 在生成代理动态类的时候，直接生成对应的代理代码，并编译出来代理类
    - @Adaptive 自适应扩展注解
        - 如果在类上上，会直接选择这个类作为实现类
        - 如果在方法上，会为该接口生成一个子类，没标注的方法抛出异常UnsupportedOperationException
            -   比如dubbo注入机制的时候，需要寻找一个自适应的工厂ExtensionFactory，来决定注入的方式，比如dubboSPI加载，和springbean加载就不一样
        -  Dubbo 就是通过 URL 来选择作为扩展的点。例如说，在 URL 定义一个属性，可以通过 URL 的属性来选择在 Dubbo 究竟用那个实现类来处理。
        - 流程
            -   加载@Adaptive的接口， 不存在则不支持
            - 为目标接口按照模板生成代码，然后编译class，然后反射生成实例
            - 结合实例，通过传入的URL，指定接口的实现，然后委托给该实例
- 服务导出
    - 和spring结合 响应spring的ContextRefreshedEvent时间，来注册所有的接口
    - 配置检查
        - export 是否到处
        - delay 延迟时间
        - ====属性
    - URL装配（采用URL作为配置信息的统一格式，所有扩展点都通过传递URL携带配置信息）
    - 加载注册中心 
    - 用proxyFactory加载自适应扩展，反射创建一个invoker，放入到exporter里面
        - invoker是实体域，dubbo的核心模型
        - 放入前调用注册中心的导出接口，在zk创建节点，完成导出
    - export导出，创建一个netty服务，写入自己的节点数据到注册中心
    - 本地暴露 远程暴露 
      > 在dubbo中我们一个服务可能既是Provider,又是Consumer,因此就存在他自己调用自己服务的情况,如果再通过网络去访问,那自然是舍近求远,因此他是有本地暴露服务的这个设计.从这里我们就知道这个两者的区别
- 服务引入
    - spring装配的时候，就生成对应的代理类装配 ReferBean
    - ReferenceConfig根据XML生成bean
    - 然后根据这个ban的信息获取zk上provider信息，组装invokerURL
    - 使用invokerUrl，来生成代理对象
- 服务调用过程
    - 动态代理生成的bean invoke
    - 在Directory 服务字典中获取所有的invoker
    - Router 做route，选出调用的服务
    - LoadBalance 根据负载均衡选择对应的机器
    - 当前invoker自带集群容错方案

- 踢出失效的服务
    - 基于zk ,服务失效,自动释放临时节点

- dubbo分层
    - 接口服务层 Service 业务逻辑API
    - 配置层 对外配置接口 ServiceConfig ReferenceConfig为中心
    - 服务代理层 服务接口透明代理 生成服务的客户端Stub和服务端的Skeleton，以ServiceProxy为中心，
    - 服务注册层 封装服务地址的注册和发现
    - 路由层 封装多个提供者的路由和负载均衡
    - 监控层 RPC调用次数和调用时间监控
    - 远程调用层 封装RPC调用 
    - 信息交换层
    - 网络传输层
    - 数据序列化层

- 注册中心
    - zk 基于watch来做
    - multiCast不需要中心，直接广播地址
    - Redis
    - simple注册中心

- 通信
    - 基于Netty

- 支持协议
    - dubbo 长连接+NIO
    - RMI JDK的RMI协议
    - WebService
    - Http
    - Hessian
    - Memcache
    - Redis

- directory 服务字典
    - RegistryDirectory 动态的，根据zk推送的节点变化而变化
    - StaticDirectory 固定的多注册中心

- 动态代理，实现动态调用远程服务
    - jdk动态代理
    - cglib动态代理
        > 依赖ASM
    - javassist动态代理
    - asm字节码
    - javassist操作字节码来生成代理实例
    - 生成代理 jdk proxy最快 执行起来 asm字节码最快

- 默认协议是什么 原因
    - dubbo协议 采用NIO单一长链接,使用线程池处理并发请求,减少握手频率,加大并发效率,性能较好 但是大数据量传输会有性能问题
    - RMI JDK标准实现 和rmi接口结合 基于tcp协议完成 支持原生RMI接口  会有一些🔗失败的情况 短链接
    - hessian 可以支持原生hessian 基于http协议 每次请求开销大,重新握手链接
    - http sping的httpinvoker实现
    - webservice 基于CXF实现
    - thrift

- 默认序列化是什么,原因
   - hessian
   -  jdk序列化
   - json
   - soap文本序列化
- dubbo响应返回怎么找到对应的请求方
  - https://juejin.cn/post/6962430318267596813#heading-13

- 启动时,spring解析bean, 遇到dubbo的命名空间,调用dubbo的解析器,然后返回一个BeanDefinition,注册到beanDefinition,实际上是一个factory bean