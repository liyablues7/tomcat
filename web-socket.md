# WebSocket 

## 目录   
- 概述  
- 应用开发  
- 生产用途  
- Tomcat WebSocket 特定配置   

## 简介  

Tomcat 支持由 [RFC 6455](http://tools.ietf.org/html/rfc6455) 所定义的 WebSocket。

## 应用开发    

Tomcat 实现由 [JSR-356](https://www.jcp.org/en/jsr/detail?id=356) 定义的 Java WebSocket 1.1 API。  

关于 WebSocket API 的使用方法，可查看相关范例，既需要查看[客户端 HTML 代码](http://svn.apache.org/viewvc/tomcat/trunk/webapps/examples/websocket/)，也需要查看[服务器代码](http://svn.apache.org/viewvc/tomcat/trunk/webapps/examples/WEB-INF/classes/websocket/)。  

## 生产用途    

虽然 WebSocket 实现可以和任何 HTTP 连接器一起使用，但并不建议和 BIO HTTP 连接器一起使用，因为 WebSocket 典型用途（大量连接很多时候都是空闲的）并不是很适合 HTTP BIO 连接器，因为该连接器需要不管连接是否空闲，每个连接都应该分配一个线程。  

目前，已有报告（[56304](http://bz.apache.org/bugzilla/show_bug.cgi?id=56304)）发现，Linux 会用大量时间来报告删除的连接。当利用 BIO HTTP 连接器使用 WebSocket 时，当在这段时间内写入时，就容易产生线程阻塞。这似乎不是一种理想的解决方案。使用内核网络参数 `/proc/sys/net/ipv4/tcp_retries2`，可以减少报道删除的连接所花费的时间。或者可以选择另一种 HTTP 连接器，因为它们使用的是非阻塞 IO，从而能让 Tomcat 实现自己的超时机制来解决这些问题。

## Tomcat WebSocket 特定配置   

Tomcat 为 WebSocket 提供了一些 Tomcat 专有配置选项。这些配置将来有望能进入 WebSocket 正式规范中。  

以阻塞模式发送 WebSocket 消息所用的写入超时默认值为 20000 毫秒（20 秒）。通过设定连接到 WebSocket 会话的用户属性集合中的 `org.apache.tomcat.websocket.BLOCKING_SEND_TIMEOUT` 属性，我们可以修改这个写入超时属性值。该属性值类型应该为 `Long`，以毫秒表示所用的超时时间，`-1` 表示超时无限。   

如果应用没有为传入的二进制消息定义 `MessageHandler.Partial`，那么必须先对任何传入的二进制消息进行缓存，继而可以通过调用一个已注册专用于处理二进制消息的 `MessageHandler.Whole` 来传递整个消息。默认用于二进制消息的缓存容量是 8192 字节。在应用中，将servlet 上下文初始参数 `org.apache.tomcat.websocket.binaryBufferSize` 设置为期望的字节值，就能修改这个缓存容量。  

如果应用没有为传入的文本消息定义 `MessageHandler.Partial`，那么必须先对任何传入的文本消息进行缓存，继而可以通过调用一个已注册专用于处理文本消息的 `MessageHandler.Whole` 来传递整个消息。默认用于文本消息的缓存容量是 8192 字节。在应用中，将servlet 上下文初始参数 `org.apache.tomcat.websocket.textBufferSize` 设置为期望的字节值，就能修改这个缓存容量。   

Java WebSocket 规范 1.0 并不允许第一个服务端点开始 WebSocket 握手之后进行程序性部署。默认情况下，Tomcat 继续允许额外的程序性部署。这一行为是通过 servlet 上下文初始化参数 `org.apache.tomcat.websocket.noAddAfterHandshake` 来控制的。将系统属性 `org.apache.tomcat.websocket.STRICT_SPEC_COMPLIANCE` 变为 `true`，可以改变默认值，但是 servlet 上下文中的显式设置却总是优先的。  

Java WebSocket 规范 1.0 要求，异步写操作的回调所在的线程应不同于初始化写操作的线程。因为容器线程池并未通过 Servlet API 暴露出来，所以 WebSocket 实现必须提供自己的线程池。这种线程池通过下列 Servlet 上下文初始化参数来控制：  

- `org.apache.tomcat.websocket.executorCoreSize`：executor 线程池的核心容量大小。如果不设置，则采用默认值 0。注意，executor 线程池最大允许容量被硬编码至 `Integer.MAX_VALUE`，实际上可以认为该值是无限的。  
- `org.apache.tomcat.websocket.executorKeepAliveTimeSeconds`：在终止前，空闲线程在 executor 线程池中留存的最长时间。如未指定，则采用默认的 60 秒。

在使用 WebSocket 客户端来连接服务端点时，建立该连接的 IO 超时是通过提供的 `javax.websocket.ClientEndpointConfig` 的 `userProperties` 来控制的。该属性是 `org.apache.tomcat.websocket.IO_TIMEOUT_MS`，是以字符串形式表示的超时时间（以毫秒计），默认为 5000（5 秒）。  

在使用 WebSocket 客户端来连接安全的服务端点时，客户端 SSL 配置是通过提供的 `javax.websocket.ClientEndpointConfig` 的 `userProperties` 来控制的。提供以下用户属性：  

- `org.apache.tomcat.websocket.SSL_CONTEXT`
- `org.apache.tomcat.websocket.SSL_PROTOCOLS`
- `org.apache.tomcat.websocket.SSL_TRUSTSTORE`
- `org.apache.tomcat.websocket.SSL_TRUSTSTORE_PWD`  

默认的信任存储密码（truststore password）为：`changeit`。  

如果设置了 `org.apache.tomcat.websocket.SSL_CONTEXT` 属性，则将忽略这两个属性：`org.apache.tomcat.websocket.SSL_TRUSTSTORE` 和 `org.apache.tomcat.websocket.SSL_TRUSTSTORE_PWD`。  
















