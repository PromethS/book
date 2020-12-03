# 一 重要概念

## 1.1 什么是 Dubbo?

Apache Dubbo (incubating) |ˈdʌbəʊ| 是一款高性能、轻量级的开源Java RPC 框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。简单来说 Dubbo 是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。

Dubbo 目前已经有接近 23k 的 Star ，Dubbo的Github 地址：https://github.com/apache/incubator-dubbo 。 另外，在开源中国举行的2018年度最受欢迎中国开源软件这个活动的评选中，Dubbo 更是凭借其超高人气仅次于 vue.js 和 ECharts 获得第三名的好成绩。

Dubbo 是由阿里开源，后来加入了 Apache 。正式由于 Dubbo 的出现，才使得越来越多的公司开始使用以及接受分布式架构。

## 1.2 什么是 RPC?RPC原理是什么?

**什么是 RPC？**

RPC（Remote Procedure Call）—远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。比如两个不同的服务 A、B 部署在两台不同的机器上，那么服务 A 如果想要调用服务 B 中的某个方法该怎么办呢？使用 HTTP请求当然可以，但是可能会比较麻烦。 RPC 的出现就是为了让你调用远程方法像调用本地方法一样简单。

**RPC原理是什么？**

![image-20200608224155056](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200608224156.png)

1. 服务消费方（client）调用以本地调用方式调用服务；
2. client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体；
3. client stub找到服务地址，并将消息发送到服务端；
4. server stub收到消息后进行解码；
5. server stub根据解码结果调用本地的服务；
6. 本地服务执行并将结果返回给server stub；
7. server stub将返回结果打包成消息并发送至消费方；
8. client stub接收到消息，并进行解码；
9. 服务消费方得到最终结果。

![image-20200608224412986](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200608224414.png)

## 1.3 为什么要用 Dubbo?

Dubbo 的诞生和 SOA 分布式架构的流行有着莫大的关系。SOA 面向服务的架构（Service Oriented Architecture），也就是把工程按照业务逻辑拆分成服务层、表现层两个工程。服务层中包含业务逻辑，只需要对外提供服务即可。表现层只需要处理和页面的交互，业务逻辑都是调用服务层的服务来实现。SOA架构中有两个主要角色：**服务提供者（Provider）和服务使用者（Consumer）**。

![image-20200608224513101](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200608224514.png)

**如果你要开发分布式程序，你也可以直接基于 HTTP 接口进行通信，但是为什么要用 Dubbo呢？**

我觉得主要可以从 Dubbo 提供的下面四点特性来说为什么要用 Dubbo：

1. **负载均衡**——同一个服务部署在不同的机器时该调用那一台机器上的服务。
2. **服务调用链路生成**——随着系统的发展，服务越来越多，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在哪个应用之前启动，架构师都不能完整的描述应用的架构关系。Dubbo 可以为我们解决服务之间互相是如何调用的。
3. **服务访问压力以及时长统计、资源调度和治理**——基于访问压力实时管理集群容量，提高集群利用率。
4. **服务降级**——某个服务挂掉之后调用备用服务。

另外，Dubbo 除了能够应用在分布式系统中，也可以应用在现在比较火的微服务系统中。不过，由于 Spring Cloud 在微服务中应用更加广泛，所以，我觉得一般我们提 Dubbo 的话，大部分是分布式系统的情况。

## 1.4 什么是分布式?

分布式或者说 SOA 分布式重要的就是面向服务，说简单的分布式就是我们把整个系统拆分成不同的服务然后将这些服务放在不同的服务器上减轻单体服务的压力提高并发量和性能。比如电商系统可以简单地拆分成订单系统、商品系统、登录系统等等，拆分之后的每个服务可以部署在不同的机器上，如果某一个服务的访问量比较大的话也可以将这个服务同时部署在多台机器上。

## 1.5 为什么要分布式?

从开发角度来讲单体应用的代码都集中在一起，而分布式系统的代码根据业务被拆分。所以，每个团队可以负责一个服务的开发，这样提升了开发效率。另外，代码根据业务拆分之后更加便于维护和扩展。

另外，我觉得将系统拆分成分布式之后不光便于系统扩展和维护，更能提高整个系统的性能。你想一想嘛？把整个系统拆分成不同的服务/系统，然后每个服务/系统 单独部署在一台服务器上，是不是很大程度上提高了系统性能呢？

# 二 Dubbo 的架构

## 2.1 Dubbo 的架构图解

![image-20200608224746368](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200608224747.png)

**上述节点简单说明：**

- **Provider：** 暴露服务的服务提供方
- **Consumer：** 调用远程服务的服务消费方
- **Registry：** 服务注册与发现的注册中心
- **Monitor：** 统计服务的调用次数和调用时间的监控中心
- **Container：** 服务运行容器

**调用关系说明：**

1. 服务容器负责启动，加载，运行服务提供者。
2. 服务提供者在启动时，向注册中心注册自己提供的服务。
3. 服务消费者在启动时，向注册中心订阅自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

**重要知识点总结：**

- **注册中心负责服务地址的注册与查找**，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
- **监控中心负责统计各服务调用次数，调用时间等**，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示
- **注册中心，服务提供者，服务消费者三者之间均为长连接**，监控中心除外
- **注册中心通过长连接感知服务提供者的存在**，服务提供者宕机，注册中心将立即推送事件通知消费者
- **注册中心和监控中心全部宕机**，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
- **注册中心和监控中心都是可选的**，服务消费者可以直连服务提供者
- **服务提供者无状态**，任意一台宕掉后，不影响使用
- **服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复**

## 2.2 Dubbo 工作原理

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613161931.jpg)

图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。

**各层说明**：

- 第一层：**service层**，接口层，给服务提供者和消费者来实现的
- 第二层：**config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
- 第三层：**proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`
- 第四层：**registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`
- 第五层：**cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`
- 第六层：**monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`
- 第七层：**protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`
- 第八层：**exchange 信息交换层**：封装请求响应模式，同步转异步，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`
- 第九层：**transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`
- 第十层：**serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`

# 三 常见问题

## 3.1 负载均衡策略

在集群负载均衡时，Dubbo 提供了多种均衡策略，默认为 `random` 随机调用。可以自行扩展负载均衡策略，参见：[负载均衡扩展](https://dubbo.gitbooks.io/dubbo-dev-book/content/impls/load-balance.html)。

### Random LoadBalance

- **随机**，按权重设置随机概率。
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

### RoundRobin LoadBalance

- **轮询**，按公约后的权重设置轮询比率。
- 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

### LeastActive LoadBalance

- **最少活跃调用数**，相同活跃数的随机，活跃数指**调用前后计数差**。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

### ConsistentHash LoadBalance

- **一致性 Hash**，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 缺省**只对第一个参数 Hash**，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
- 缺省**用 160 份虚拟节点**，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`

## 3.2 服务注册发现流程

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609011815.jpg)

-  Provider(提供者)绑定指定端口并启动服务
-  指供者连接注册中心，并发本机IP、端口、应用信息和提供服务信息发送至注册中心存储
-  Consumer(消费者），连接注册中心 ，并发送应用信息、所求服务信息至注册中心
-  注册中心根据消费者所请求的服务信息匹配对应的提供者列表发送至Consumer 应用缓存。
-  Consumer 在发起远程调用时基于缓存的消费者列表择其一发起调用。
-  Provider 状态变更会实时通知注册中心、在由注册中心实时推送至Consumer

## 3.3 服务调用流程

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609011715.jpg)

默认是**同步等待**结果阻塞的，**⽀持异步调⽤**。

Dubbo 是基于 **NIO** 的⾮阻塞实现并⾏调⽤，客户端不需要启动多线程即可完成并⾏调⽤多个远程服务，相对多线程开销较⼩，异步调⽤会返回⼀个 **Future** 对象  

![image-20200613163032864](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613163034.png)

## 3.4 Dubbo的多协议

-  **dubbo：** **单一长连接和NIO异步通讯**，适合**大并发小数据量**的服务调用，以及消费者远大于提供者。传输协议TCP，异步，Hessian序列化；
-  **rmi：** **采用JDK标准的rmi协议实现**，传输参数和返回参数对象需要实现Serializable接口，使用java标准序列化机制，**使用阻塞式短连接**，传输数据包大小混合，消费者和提供者个数差不多，**可传文件，传输协议TCP**。多个短连接，TCP协议传输，同步传输，适用常规的远程服务调用和rmi互操作。在依赖低版本的Common-Collections包，java序列化存在安全漏洞；**适合大数据量场景**。
-  webservice： 基于WebService的远程调用协议，集成CXF实现，提供和原生WebService的互操作。多个短连接，基于HTTP传输，同步传输，适用系统集成和跨语言调用；
-  **http：** **基于Http表单提交**的远程调用协议，**使用Spring的HttpInvoke实现**。**多个短连接，传输协议HTTP**，传入参数大小混合，提供者个数多于消费者，需要给应用程序和浏览器JS调用；**适合大数据量场景**。
-  **hessian：** **集成Hessian服务，基于HTTP通讯**，采用Servlet暴露服务，Dubbo内嵌Jetty作为服务器时默认实现，提供与Hession服务互操作。多个短连接，同步HTTP传输，Hessian序列化，传入参数较大，提供者大于消费者，提供者压力较大，可传文件；
-  memcache： 基于memcached实现的RPC协议
-  redis： 基于redis实现的RPC协议

## 3.5 Dubbo 的注册中心

- Multicast注册中心：Multicast注册中心不需要任何中心节点，只要广播地址，就能进行服务注册和发现。基于网络中组播传输实现；组播受网络结构限制，只适合小规模应用或开发阶段使用。组播地址段: 224.0.0.0 - 239.255.255.255
- **Zookeeper注册中心：** 基于分布式协调系统Zookeeper实现，采用Zookeeper的watch机制实现数据变更；**推荐**
- **Nacos注册中心**：Nacos 是 Dubbo 生态系统中重要的注册中心实现，其中 [`dubbo-registry-nacos`](https://github.com/apache/incubator-dubbo/tree/master/dubbo-registry/dubbo-registry-nacos) 则是 Dubbo 融合 Nacos 注册中心的实现。**推荐**
- redis注册中心： 基于redis实现，采用key/Map存储，住key存储服务名和类型，Map中key存储服务URL，value服务过期时间。基于redis的发布/订阅模式通知数据变更；
- Simple注册中心

## 3.6 集群容错机制

**Failover Cluster**-默认

失败自动切换，当出现失败，重试其它服务器 。通常用于读操作，但重试会带来更长延迟。可通过 `retries="2"` 来设置重试次数(不含第一次)。

**Failfast Cluster**

快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

**Failsafe Cluster**

失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

**Failback Cluster**

失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

**Forking Cluster**

并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 `forks="2"` 来设置最大并行数。

**Broadcast Cluster**

广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。

# 四 SPI

## Dubbo SPI 简介

SPI(Service Provider Interface)是**服务发现机制**，Dubbo没有使用jdk SPI而对其增强和扩展：

1. **jdk SPI仅通过接口类名获取所有实现，但是Duboo SPI可以根据接口类名和key值获取具体一个实现**
2. **可以对扩展类实例的属性进行依赖注入，即IOC**
3. **可以采用装饰器模式实现AOP功能**

你可以发现Dubbo的源码中有很多地方都用到了**@SPI注解**，例如：Protocol（通信协议），LoadBalance（负载均衡）等。基于Dubbo SPI，我们可以非常容易的进行拓展。**ExtensionLoader是扩展点核心类**，用于载入Dubbo中各种可配置的组件，比如刚刚说的Protocol和LoadBalance等。那么接下来我们看一下Dubbo SPI的示例

## Dubbo SPI 示例

比如现在我们要拓展Protocol这个组件，新建一个DefineProtocol类并修改默认端口为8888：

```java

public class DefineProtocol implements Protocol {
    @Override
    public int getDefaultPort() {
        return 8888;
    }

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        return null;
    }

    @Override
    public <T> Invoker<T> refer(Class<T> aClass, URL url) throws RpcException {
        return null;
    }

    @Override
    public void destroy() {

    }
}
```

配置文件的文件名字是接口的全限定名，那么在这个例子中就是：com.alibaba.dubbo.rpc.Protocol

Dubbo SPI所需的配置文件要放在以下3个目录任意一个中：

- **META-INF/services/**
- **META-INF/dubbo/**
- **META-INF/dubbo/internal/**

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609001103.png)

同时需要将服务提供者配置文件设计成KV键值对的形式，Key是拓展类的name，Value是扩展的全限定名实现类。比如：

```properties
myProtocol=com.grimmjx.edu.DefineProtocol
```

然后测试一下：

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("dubbo-client.xml");

Protocol myProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("myProtocol");

System.out.println(myProtocol.getDefaultPort());
```

结果如下：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609001145.png)

 ## 源码解析

那我们就从上面的方法看起，重要方法红色标注：

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("myProtocol");
```

1.getExtensionLoader方法，入参是一个可拓展的借口，返回ExtensionLoader实体类，然后可以通过name（key）来获取具体的扩展：

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    // 扩展点必须是接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    // 必须有@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }
    // 每个扩展只会被加载一次
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        // 初始化扩展
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

2.getExtension方法，首先检查缓存，如果没有则用双检锁方式创建实例：

```java
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        // 默认拓展实现类
        return getDefaultExtension();
    }
    // 获取持有目标对象
    Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    // 双检锁
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建实例
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

3.**createExtension方法**，**这个方法比较核心**。做了有4件事情，第3件和第4件分别为上面介绍Dubbo SPI中对jdk SPI扩展的第二和第三点（红字已标注）。请看代码注释：

```java
private T createExtension(String name) {
    // 1.加载配置文件所有拓展类，得到配置名-拓展类的map，从map中获取到拓展类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 2.通过反射创建实例
            // EXTENSION_INSTANCES这个map是配置名-拓展类实例的map
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 3.注入依赖，即IOC
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            // 4.循环创建Wrapper实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 通过反射创建Wrapper实例
                // 向Wrapper实例注入依赖，最后赋值给instance
                // 自动包装实现类似aop功能
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

4.getExtensionClasses方法，这里就是找出所有拓展类，返回一个配置名-拓展类的map。

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双检锁
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 缓存无则加载
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

5.loadExtensionClasses方法，主要就是解析SPI注解，然后加载指定目录的配置文件，也不是很难

```java
private Map<String, Class<?>> loadExtensionClasses() {
    // 获取SPI注解，检查合法等
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if(defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if(value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if(names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if(names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // META-INF/dubbo/internal/目录
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    // META-INF/dubbo/目录
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    // META-INF/services/目录
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

这里返回的extensionClasses的map就肯定包含了"myProtocol"->"com.grimmjx.edu.DefineProtocol"。同时也可以看到，dubbo支持有很多协议：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609001702.png)

接下来不用多说了吧，再从map里get出"myProtocol"，得到的就是我们自定义的协议类。

上面3.createExtension方法，这个方法里注释的3和4很关键，里面实现了依赖注入和AOP的功能，那么接下来我们主要看看Dubbo的IOC和AOP

## Dubbo 的 IOC

Dubbo IOC是**通过setter方法注入依赖的**，首先遍历方法是否有setter方法特征，如果有则通过**objectFactory获取依赖对象**进行注入。Dubbo注入的可能是Dubbo的扩展，也有可能是一个Spring bean！

上面的3方法中有这一行代码，实现了Dubbo SPI的IOC

```java
injectExtension(instance);
```

1.injectExtension方法。我们主要看这个方法，自动装配的功能都在这个方法中：

```java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                // 如果方法以set开头 && 只有一个参数 && 方法是public级别的
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    // 获取setter方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";

                        // 关键，从objectFactory里拿出依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            // 利用反射进行注入
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

2.objectFactory，究竟这里的objectFactory是什么呢？**它是ExtensionFactory类型的**，自身也是一个扩展点。我先告诉你这里的objectFactory是**AdaptiveExtensionFactory**。后面会有解释。

```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());		
}
```

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609002049.png)

3.getExtension方法，这里比较简单，为了直观，用debug模式看一下，factories有两个，分别是SpiExtensionFactory和SpringExtensionFactory

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609002152.png)

对于SpringExtensionFactory就是从工厂里根据beanName拿到Spring bean来注入，对于SpiExtensionFactory就是根据传入Class获取自适应拓展类，那么我们写一段代码，来试试获取一个Spring Bean玩玩，先定义一个bean：

```java

public class Worker {

    private int age = 24;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

　然后再配置文件里配置这个Spring Bean：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609002300.png)

　　最后简单写个Main方法，可以看出SpringExtensionFactory可以加载Spring Bean：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609002307.png)

## Dubbo 的 AOP

**Dubbo中也支持Spring AOP类似功能，通过装饰者模式，使用包装类包装原始的扩展点实例。**在扩展点实现前后插入其他逻辑，实现AOP功能。说这很绕口啊，那什么是包装类呢？举个例子你就知道了：

```java
class A{
    private A a;
    public A(A a){
        this.a = a;
    }

    public void do(){
        // 插入扩展逻辑
        a.do();
    }
}
```

这里的插入扩展逻辑，是不是就是实现了AOP功能呢？比如说Protocol类，有2个Wrapper，分别是ProtocolFilterWrapper和ProtocolListenerWrapper，我们可以在dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol看到：

```properties
filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
```

源码的话createExtension方法里的注释已经写的很清楚了，这里可以自行研究。所以我们在最开始的Dubbo SPI的例子中，我们打个断点就很明显了，得到的myProtocol对象其实是这样的：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200609020000.png)

**如果你调用export方法的话，会先经历ProtocolFilterWrapper的export方法，再经历ProtocolListenerWrapper的export方法**，这样是不是就实现了Spring AOP的功能呢？

## Dubbo SPI 自适应

这里getAdaptiveExtension到底获取的是什么呢，这里涉及到SPI 自适应扩展，十分重要，涉及到@Adaptive注解。

**如果注解加在类上**，比如说com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory（自行验证）：**直接加载当前的适配器**

**如果注解加载方法上**，比如说com.alibaba.dubbo.rpc.Protocol：动态创建一个自适应的适配器，就像是执行如下代码，**返回的是一个动态生成的代理类**

```java
Protocol adaptiveExtension = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

```java
public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

为什么getDefaultPort和destroy方法都是直接抛出异常呢？**因为Protocol接口只有export和refer方法使用了@Adaptive注解**，Dubbo会自动生成自适应实例，其他方法都是抛异常。

为什么还要要动态生成呢？有时候拓展不想在框架启动的时候被加载，而是希望在扩展方法被调用的时候，**根据运行时参数进行加载**。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```

```java
public T getAdaptiveExtension() {
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {    // 缓存未命中
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 创建自适应拓展
                        instance = createAdaptiveExtension();
                        // 设置自适应拓展到缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: ...");
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance:  ...");
        }
    }

    return (T) instance;
}
```

```java
private T createAdaptiveExtension() {
    try {
        // 获取自适应拓展类，并通过反射实例化
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension ...");
    }
}
```

```java
private Class<?> getAdaptiveExtensionClass() {
    // 通过 SPI 获取所有的拓展类
    getExtensionClasses();
    // 检查缓存，若缓存不为空，则直接返回缓存
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

```java
private Class<?> createAdaptiveExtensionClass() {
    // 构建自适应拓展代码
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    // 获取编译器实现类
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译代码，生成 Class
    return compiler.compile(code, classLoader);
}
```

```java
// 通过反射获取所有的方法
Method[] methods = type.getMethods();
boolean hasAdaptiveAnnotation = false;
// 遍历方法列表
for (Method m : methods) {
    // 检测方法上是否有 Adaptive 注解
    if (m.isAnnotationPresent(Adaptive.class)) {
        hasAdaptiveAnnotation = true;
        break;
    }
}

if (!hasAdaptiveAnnotation)
    // 若所有的方法上均无 Adaptive 注解，则抛出异常
    throw new IllegalStateException("No adaptive method on extension ...");
```

 # 五 服务导出



 

 

 