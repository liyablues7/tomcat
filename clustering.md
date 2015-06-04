# 集群化与会话复制

## 重要说明  

相关内容详情可以查看[集群配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster.html)

## 目录  

- 快速入门  
- 集群基本知识  
- 概述  
- 集群信息  
- 当发生崩溃时，将会话绑定到故障转移节点
- 配置范例  
- 集群架构  
- 工作原理  
- 利用 JMX 监控集群  
- FAQ




## 快速入门

只需将下列信息放入 `<Engine>` 或 `<Host>` 元素即可实现集群：   

`<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>`

上述配置启用了全局（all-to-all）会话复制功能，全局会话复制是指利用 `DeltaManager` 来只复制会话中的**变更**（*Session Delta*，也译作“会话增量”）。这里说的“全局”是指：会话变更会被复制到集群中的所有其他节点（指 Tomcat 实例）中。全局复制非常适于小集群，但不建议在大集群（包含很多 Tomcat 节点）上采用这种方法。另外，值得注意的是，当使用 `delta manager` 时，它会将变更复制到所有的节点上，甚至包括那些根本没有部署该应用的节点。  

为了解决这个问题，你就得使用 **BackupManager**。它会把会话数据复制给一个指定的备份节点（这种复制也被称为“配对复制”），而且该备份节点也一定要部署了相关应用。BackupManager 的缺点在于：不像 DeltaManager 那样久经实践考验。   


下面是一些重要的默认值。  

1. IP 组播地址为：228.0.0.4 
2. IP 组播端口为：45564（端口和地址一起确定了集群成员）。  
3. 广播的 IP 是 `java.net.InetAddress.getLocalHost().getHostAddress()`（你一定不能广播 127.0.0.1，这是一个常见错误。）  
4. 侦听复制信息的 TCP 端口是在 4000 - 4100 之间遇到的第一个能用的服务器套接字。
5. 两个侦听器都配置有 `ClusterSessionListener`。  
6. 两个拦截器都配置有 `TcpFailureDetector` 和 `MessageDispatch15Interceptor`。   

下面是默认的集群配置：   

```     

        <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">

          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>

          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="auto"
                      port="4000"
                      autoBind="100"
                      selectorTimeout="5000"
                      maxThreads="6"/>

            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
          </Channel>

          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=""/>
          <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>

          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>

          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>  
        
```     


稍后，本文档将更详细地阐述这部分的内容。  

## 集群基本知识  

要想在 Tomcat 8 上运行会话复制，需要执行以下步骤：  

- 所有的会话属性必须实现 `java.io.Serializable`。  
- 在 server.xml 中取消注释 `Cluster` 元素。   
- 如果你已经定义了自定义集群值，确保在 server.xml 中的 Cluster 元素下面也定义了 `ReplicationValve`。   
- 如果你的多个 Tomcat 实例都运行在同一台机器上，则要确保每个实例都具有唯一的 `tcpListenPort`。通常 Tomcat 会自行解决这个问题，会在 4000 - 4100 上自动侦测可用的端口。  
- 确保 `web.xml` 含有 `<distributable/>` 属性。   
- 如果使用 mod_jk，则要确保在 `<Engine name="Catalina" jvmRoute="node01" >` 上设定 `jvmRoute` 属性。`jvmRoute` 属性值必须匹配 workers.properties 中的 worker 名称。  
- 所有的节点必须具有相同的时间，并且与 NTP 服务器同步。  
- Make sure that your loadbalancer is configured for sticky session mode.   

负载均衡可以通过多种技术来实现，参看[负载均衡](》》)部分。  

注意：会话状态是通过 cookie 来记录的，所以你的 URL 必须保持一致，否则就会创建一个新会话。   

注意：当前如要支持集群，需要 JDK 1.5 或更新版本。  

集群模块使用 Tomcat 的JULI 日志框架，所以可以通过 logging.properties 文件来配置日志。为了跟踪消息，你可以启用 `org.apache.catalina.tribes.MESSAGES` 键上的日志。   

## 概述  

在 Tomcat 中，可以使用以下方法中的一种启用会话复制：

1. 使用会话持久性，将会话保存到共享文件系统中（PersistenceManager + FileStore）。  
2. 使用会话持久性，将会话保存到共享数据库中（PersistenceManager + JDBCStore）。  
3. 使用内存复制，使用 Tomcat 自带的 SimpleTcpCluster（lib/catalina-tribes.jar + lib/catalina-ha.jar）。  

在这一版本的 Tomcat 中，可以使用 `DeltaManager` 执行全局式会话状态复制，或者使用 `BackupManager` 执行备份复制，将会话复制到一个节点上。全局式会话复制这种算法只有在集群较小时才比较有效。对于大型集群，更多使用主从会话复制，将会话存储到一台配置了 BackupManager 的备份服务器上。  

当前可以使用域名 worker 属性（mod_jk 版本 > 1.2.8）来构建集群分区，从而有可能利用 DeltaManager 实现更具有可扩展性的集群方案（需要为此配置域的拦截器）。为了在全局性环境中降低网络流量，可以将集群分成几个较小的分组。为不同的分组使用不同的组播地址即能实现这种方案。下图展示的是一种简单的配置方案。   



DNS 轮询
|
负 载 均 衡 器
/              \
集群 1              集群 2
/     \             /     \
Tomcat 1 Tomcat 2  Tomcat 3 Tomcat 4  


值得注意的是，使用会话复制仅仅是集群化的一个基础方案。关于集群的实现，另一个常用的概念是**耕种**（farming），比如：只需将应用部署到一个服务器上，集群就会将部署分发到整个集群的各个节点中。这都是 FarmWarDeployer 所具有的功能（参看 `server.xml` 中的集群范例）。  

下一节将深入介绍会话复制的工作原理以及配置方式。   


## 集群信息  

通过组播心跳包（heartbeat）建立起成员（Membership）关系，因此，如果希望细分集群，可以改变 `<Membership>` 元素中的组播 IP 地址或端口。  

心跳包中含有 Tomcat 节点的 IP 地址，以及 Tomcat 用来侦听会话复制流量的 TCP 端口。所有的数据通信都使用了 TCP 协议。    

`ReplicationValve` 用于查找请求结束的时间，如果存在会话复制，就对该复制进行初始化。只复制会话变更的数据（通过在会话上调用 `setAttribute` 或 `removeAttribute` 来完成）。   

复制的异步与同步模式应该是最值得我们注意的一个特点了。在同步复制模式下，复制的会话通过线缆传送，重新在所有集群节点上实例化，这样才会返回请求。同步和异步是通过 `channelSendOptions` 标志（整型值）来配置的。`SimpleTcpCluster/DeltaManager` 组合的默认值是 8，从而是异步。详情可以参考一下 [send flag(overview)](http://tomcat.apache.org/tomcat-8.0-doc/tribes/introduction.html) 或  [send flag(javadoc)](http://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/tribes/Channel.html)。在异步复制过程中，请求不必等到数据被复制完毕即可返回。异步复制缩短了请求时间，而同步复制则保证了能在请求返回之前复制完会话。  


## 当发生崩溃时，将会话绑定到故障转移节点

如果你使用了 mod_jk 而没有使用粘性会话（sticky session），或者粘性会话由于某种原因而不起作用，或者仅是故障转移，会话 id 需要修改，因为它之前含有之前 Tomcat 的 worker id（通过 Engine 元素中的 jvmRoute 定义）。为了解决这个问题，就要用到 JvmRouteBinderValve。   

JvmRouteBinderValve 将重写会话 id，以便确保下一个请求在故障转移后依然能保持粘性（不会因为 worker 不再可用而回滚到某个随机的节点中）。利用同样的名字，该值重写了 cookie 中的 JSESSIONID 值。假如没有正确地设置 valve，将使 mod_jk 模块在失败后很难保持会话的粘性。

记住，如果在 server.xml 中自定义值，那么默认值将不再有效，所以一定要确保添加了默认所定义的值。  

> **提示**：  

> 利用属性 **sessionIdAttribute** 可以改变包含旧会话 id 的请求属性名。默认的请求属性名是：*org.apache.catalina.ha.session.JvmRouteOrignalSessionID*。    
> 
> 
---   

> **技巧**：  
> 
> 可以启用 mod_jk 翻转模式在删除一个节点，  然后启用了 mod_jk Worker 禁用 JvmRouteBinderValves 。这种用例意味着只有请求的会话才能得到迁移。
> 
---     



## 配置范例  

```
        <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="6">

          <Manager className="org.apache.catalina.ha.session.BackupManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"
                   mapSendOptions="6"/>
          <!--
          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>
          -->
          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="auto"
                      port="5000"
                      selectorTimeout="100"
                      maxThreads="6"/>

            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/>
          </Channel>

          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=".*\.gif|.*\.js|.*\.jpeg|.*\.jpg|.*\.png|.*\.htm|.*\.html|.*\.css|.*\.txt"/>

          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>

          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>

```

下面来仔细剖析一下这段代码。  


```
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
channelSendOptions="6">

```  

Cluster 是主要元素，可在该元素内配置所有的集群相关细节。 对于 `SimpleTcpCluster` 类或者调用 `SimpleTcpCluster.send` 方法的对象，它们所发出的每一个消息上都附加着一个 `channelSendOptions` 标志。关于发送标志的描述可参见我们的 [javadoc 文档](http://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/tribes/Channel.html)。`DeltaManager` 使用 `SimpleTcpCluster.send` 方法发送信息，而备份管理器则直接通过 channel 来发送自身。  

更多详细信息请参见[集群配置参考文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster.html)。  

```  
          <Manager className="org.apache.catalina.ha.session.BackupManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"
                   mapSendOptions="6"/>
          <!--
          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>
          -->

```  


如果在 <Context> 元素中没有定义 manager，则以上可当做 manager 的配置模板。在 Tomcat 5.x 时期，每个标识为**可分发**（distributable）的 Web 应用都必须使用同样的 manager，而如今不同了，我们可以为每个应用定义一个 manager 类，从而在集群中混合多个 manager。显然，A 节点上的某个应用的所有 manager 必须与 B 节点上的同样应用的 manager 相同。如果没有为应用指定 manager，而且该应用被标识为 `<distributable/>`，Tomcat 就会采取这种 manager 配置，创建一个克隆该配置的 manager 实例。    

更多详细信息请参见[集群管理器文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-manager.html)。  

`          <Channel className="org.apache.catalina.tribes.group.GroupChannel">`  

Channel 元素是 [Tribes](http://tomcat.apache.org/tomcat-8.0-doc/tribes/introduction.html) 架构的一个重要组成部分，Tribes 是 Tomcat 内部所使用的分组通信架构。Channel 元素封装了所有通信相关事项以及成员逻辑。

详情参见[集群 Channel 文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-channel.html)。  

```  
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>

```

成员关系（Membership）是通过组播来实现的。注意，如果你想将成员扩展到组播范围之外的某个点时，Tribes 现在已经能够支持使用 `StaticMembershipInterceptor` 的静态成员。`address` 属性是所用的组播地址，`port` 是所用的组播端口号。这两项组合起来将集群隔离开。如果你希望一个 QA 集群和一个生产集群，最简单	的方法就是将 QA 集群的组播地址和端口号不同于生产集群的组播地址和端口号组合。   

成员组件将其自身的 TCP 地址和端口广播到其他节点处，从而使节点间的通信都可以通过 TCP 协议来完成。请注意被广播的 TCP 地址正是 `Receiver.address` 属性值。

详情参见[集群成员文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-membership.html)。  

```  
<Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
address="auto"
port="5000"
selectorTimeout="100"
maxThreads="6"/>

```

在 Tribes 架构中，数据的发送与接收以及被拆分为两种功能性组件了。正如其名所示，Receiver 负责接收信息。由于 Tribes 与线程无关（其他架构也开始采用这一种常见改进了），该组件内部包含一个线程池，设定有 `maxThreads` 和 `minThreads` 两种参数。  

`address` 参数值是主机地址，由成员组件广播到其他节点中。  

关于更多详情，可参看[Receiver 文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-receiver.html)。  

```
<Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
<Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
</Sender>

```

Sender 组件负责将消息发送给其他节点。Sender 含有一个 shell 组件 `ReplicationTransmitter`，但真正所要完成的任务则是通过子组件 `Transport` 来完成的。由于 Tribes 支持一个 Sender 池，所以消息可以做到同步；如果使用的是 NIO Sender，你也可以并发地发送消息。  

并发（Concurrently）意味着将同时有多个发送者对应着一条消息，并行（Parallel）则意味着同时有多个消息对应着多个发送者。详情请参考[这篇文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-sender.html)。

```  
<Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
<Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
<Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/>
</Channel>

```  

Tribes 利用了一个堆栈传送消息。每个堆栈内的元素都被称为拦截器，跟 Tomcat 容器中的 valve 的作用差不多。使用拦截器，逻辑可被分成更容易管理的代码段。上面配置中的拦截器：  

- `TcpFailureDetector` 通过 TCP 核实崩溃的节点。如果组播包丢失，该拦截器就会防止误报的情况出现，比如，某个正在运行的节点虽然活跃，但也被标示为已崩溃。  
- `MessageDispatch15Interceptor` 分派消息到线程（线程池），异步发送消息。    
- `ThroughputInterceptor` 输出对信息流量的简单统计。

请注意，拦截器的顺序很重要。在 server.xml 中定义的顺序正是它们出现在 channel 堆栈中的顺序。这种机制就像是链表，最前面的是第一个拦截器，末尾的是最后一个拦截器。更多详细资料，可参看[这篇文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-interceptor.html)。   

```  
<Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
filter=".*\.gif|.*\.js|.*\.jpeg|.*\.jpg|.*\.png|.*\.htm|.*\.html|.*\.css|.*\.txt"/>

```

集群使用 valve 来跟踪针对 Web 应用的请求。我们之前已经提到过 ReplicationValve 和 JvmRouteBinderValve。`<Cluster>` 元素本身并不是 Tomcat 管道的一部分，集群将 valve 添加到了它的父容器上，比如说 `<Cluster>` 元素被配置到 `<Engine>` 元素中，那么 valve 就会被加到 `<Engine>` 元素中。更多详情，请参考[集群 valve 配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-valve.html)。  

```  
<Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
tempDir="/tmp/war-temp/"
deployDir="/tmp/war-deploy/"
watchDir="/tmp/war-listen/"
watchEnabled="false"/>

```

默认的 Tomcat 集群支持**耕种部署**（farmed deployment），比如说集群可以在其他的节点上部署和取消部署应用。该组件的状态目前还不稳定，但我们很快就会解决这个问题。Tomcat 5.0 和 5.5 版本相比，在部署算法上有一点变化。组件的逻辑改变到部署目录必须与应用目录相匹配。  

更多详情，请参考[集群部署器文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-deployer.html)。   

```  
<ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
</Cluster>

```


因为 `SimpleTcpCluster` 本身既是 Channel 对象的发送者，又是接受者，所以组件可以将它们自身注册成SimpleTcpCluster 的侦听器。 上面这个侦听器 `ClusterSessionListener` 将侦听 `DeltaManager` 复制的消息，并将会话变更应用到 manager 上，反过来应用到会话上。

更多详情，参看 [集群侦听器文档](http://tomcat.apache.org/tomcat-8.0-doc/config/cluster-listener.html)。   



## 集群架构    

组件层级：

```
Server
|
Service
|
Engine
|  \
|  --- Cluster --*
|
Host
|
------
/      \
Cluster    Context(1-N)
|             \
|             -- Manager
|                   \
|                   -- DeltaManager
|                   -- BackupManager
|
---------------------------
|                       \
Channel                    \
----------------------------- \
|                          \
Interceptor_1 ..               \
|                            \
Interceptor_N                    \
-----------------------------      \
|          |         |             \
Receiver    Sender   Membership       \
-- Valve
|      \
|       -- ReplicationValve
|       -- JvmRouteBinderValve
|
-- LifecycleListener
|
-- ClusterListener
|      \
|       -- ClusterSessionListener
|
-- Deployer
\
-- FarmWarDeployer

```

## 工作原理  

为了便于理解集群的工作机制，下面将通过一些实际情境来加深一下你的理解，我们只打算采用 2 个 Tomcat 实例：`Tomcat A` 和 `Tomcat B`。具体发生的事件流程为：  

1. `Tomcat A` 启动。  
2. `Tomcat A` 启动完毕后，`Tomcat B` 才启动。  
3. `Tomcat A` 接收一个请求，创建了一个会话 `S1`。  
4. `Tomcat A` 崩溃。     
5. `Tomcat B` 接收到对会话 `S1` 的请求。    
6. `Tomcat A` 启动。  
7. `Tomcat A` 接收到一个请求，调用会话 `S1` 上的 `invalidate` 方法。  
8. `Tomcat B` 接收到一个对新会话 `S2` 的请求。   
9. `Tomcat A` 会话 `S2` 由于不活跃而超时。      


介绍完了事件序列，下面详细剖析一下在会话复制代码中到底发生了什么。  

1. **`Tomcat A` 启动**

Tomcat 使用标准启动顺序来启动。Host 对象创建好之后，会关联一个 Cluster 对象。在解析上下文时，如果 web.xml 中包含 distributable 元素，Tomcat 就会让 Cluster 类（在该例中是 `SimpleTcpCluster`）创建复制的上下文的管理器。启用了集群并在 web.xml 中设置了 distributable 元素后，Tomcat 会为该上下文创建一个 `DeltaManager`（而不是 `StandardManager`）。Cluster 类会启动一个成员服务（组播）和一个复制服务（TCP 单播）。下文将会介绍更多的架构细节。

2. **`Tomcat B` 启动**    

Tomcat B 启动时，采取的顺序与 Tomcat A 基本一样。集群启动，建立成员（Tomcat A 与 Tomcat B）。Tomcat B 会请求集群中已有服务器（本例中是 Tomcat A）的会话状态。如果 Tomcat A 响应该请求，那么在 Tomcat B 开始侦听 HTTP 请求之前，Tomcat A 会将会话状态传到 Tomcat B那里；如果 Tomcat A 没有响应该请求，Tomcat 会等待 60 秒，超过这个时间之后，发出一个日志项。该会话状态会发送到每一个在 web.xml 中设置了 distributable 元素的应用。注意：为了有效地使用会话复制，所有的 Tomcat 实例都必须拥有相同的配置。  

3. **`Tomcat A` 接收一个请求，创建了一个会话 `S1`**   

Tomcat A 对发送给它的请求的处理方式，与没有会话复制时的处理方式完全相同。请求完成时会触发相应行为，`ReplicationValve` 会在响应返回用户之前拦截请求。如发现会话已经更改，则使用 TCP 将会话复制到 Tomcat B 上。一旦序列化的数据被转交给操作系统的 TCP 逻辑，请求就会重新通过 valve 管道返回给用户。对于每一个请求，都将复制所有的会话，这样做就有利于复制那些在会话中修改属性的代码，使其即使不必调用 `setAttribute` 或 `removeAttribute`，也能被复制。另外，使用 `useDirtyFlag` 配置参数也可以优化会话的复制次数。

4. **`Tomcat A` 崩溃**    

当 Tomcat A 崩溃时，Tomcat B 会接到通知，得知 Tomcat A 已被移出集群，随即 Tomcat B 就在其成员列表中也将 Tomcat A 移除，Tomcat B 从而不再收到关于 Tomcat A 的任何通知。负载均衡器会把从 Tomcat A 发送给 Tomcat B 的请求重新定向，所有的会话都将保持现有的状态。  

5. **`Tomcat B` 接收到对会话 `S1` 的请求**  

毫无悬念，Tomcat B 会照处理其他请求的方式那样来处理该请求。  

6. **`Tomcat A` 启动**   

在 Tomcat A 开始接收新的请求之前，将会根据上面（1）（2）两条所所说明的启动序列来启动。Tomcat A 会加入集群，联系 Tomcat B 并获取所有的会话状态。一旦接收到会话状态，就会完成加载，并打开 HTTP/mod_jk 端口。所以，除非 Tomcat A 从 Tomcat B 那里接收到了会话变更，否则没有发给 Tomcat A 的请求。   

7. **`Tomcat A` 接收到一个请求，调用会话 `S1` 上的 `invalidate` 方法**  

会拦截对 `invalidate` 的调用, 并且 `session` 会被加入失效会话队列。 在请求完成时，不会发送会话改变消息，而是发送一个 “到期” 消息给 Tomcat B，Tomcat B 也会让此会话失效。

8. **`Tomcat B` 接收到一个对新会话 `S2` 的请求**    

同步骤 3。

9. **`Tomcat A` 会话 `S2` 由于不活跃而超时**  

invalidate 调用会被拦截，当一个会话被用户标记失效时，该会话就会加入到无效会话队列。此时，失效的会话不会被复制，直到另一个请求通过系统并检查无效会话队列。  


**Membership** 集群成员是通过非常简单的组播 ping 命令来实现的。每个 Tomcat 实例都会定期发送一个组播 ping，ping 消息中包含 Tomcat 实例自身的 IP 和配置的 TCP 监听端口。如果实例在一个给定的时间内没有收到这样的 ping 信息,就会认为那个成员已经崩溃了。非常简洁高效！当然，您需要在系统上启用广播。  


**TCP 复制** 一旦收到一个多播 ping 包，在下一个复制请求时成员被添加到集群，发送实例将使用的主机和端口信息，以及建立TCP套接字。使用该套接字发送序列化的数据。之选择TCP套接字，是因为它内建有流量控制和保证发送的功能。所以发送的数据肯定会到达那里。   

**分布式的锁定与使用架构的页面s** Tomcat 在跨集群同步不保持会话实例。这种逻辑的实现将是多开销和导致各种各样的问题。如果你的客户用同一个会话同时发送多个请求，那么最后的请求将会覆盖集群中的其他会话。

## 利用 JMX 监控集群  

使用集群时，如何监控是一个重要课题。有些集群对象是 JMX MBean。

添加下列属性到启动脚本上。

```  
set CATALINA_OPTS=\
-Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=%my.jmx.port% \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false


```  


下面是 Cluster 的 MBean 列表：  

|名称|描述|MBean 对象名-引擎|MBean 对象名-主机|
|---|---|---|---|
| `Cluster` |完整的 cluster 元素|`type=Cluster`|`type=Cluster,host=${HOST}`|
| `DeltaManager` |该管理器控制会话，并处理会话复制|`type=Manager,context=${APP.CONTEXT.PATH}, host=${HOST}`|`type=Manager,context=${APP.CONTEXT.PATH}, host=${HOST}`|
| `FarmWarDeployer` |将一个应用部署到该集群的所有节点上。|目前不支持|`type=Cluster, host=${HOST}, component=deployer`|
| `Member` |代表集群中的一个节点|`type=Cluster, component=member, name=${NODE_NAME}`|`type=Cluster, host=${HOST}, component=memdber, name=${NODE_NAME}`|
| `ReplicationValve` |该 valve 控制到备份节点的会话复制|`type=Valve,name=ReplicationValve`|`type=Valve,name=ReplicationValve,host=${HOST}`|
| `JvmRouteBinderValve` |将 Session ID 变为 tomcat 当前的 jvmroute 的集群回滚值|`type=Valve,name=JvmRouteBinderValve, context=${APP.CONTEXT.PATH}`|`type=Valve,name=JvmRouteBinderValve,host=${HOST}, context=${APP.CONTEXT.PATH}`|



## 常见问题解答  

请参看 [FAQ：集群](http://wiki.apache.org/tomcat/FAQ/Clustering)文档。  





