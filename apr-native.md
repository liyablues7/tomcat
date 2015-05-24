

## 目录   

- 简介   
- 安装   
	1. Windows  
	2. Linux   
- APR 组件  
- 配置 APR 生命周期侦听器（APR Lifecycle Listener）  
	1. AprLifecycleListener   
- 配置 APR 连接器  
	1. HTTP/HTTPS  
	2. AJP   
	

## 简介   

Tomcat 可以使用 [Apache Portable Runtime](http://apr.apache.org)（APR） 来提供优秀的可扩展性与性能，并能更好地与原生服务器技术相集成。APR 是一种具有高度可移植性的类库，是 Apache HTTP Server 2.x 的核心。APR 具有许多用途，包括访问高级 IO 功能（比如 sendfile、epoll 和 OpenSSL）、系统级功能（随机数生成、系统状态，等等）以及》原生进程处理（共享内存、NT 管道、UNIX 套接字）。  

这些特性能让 Tomcat 成为一种通用的 Web 服务器，更使其更好地与原生的 Web 技术相集成。从整体上来说，这使得 Java 越来越有望成为一个成熟的 Web 服务器平台，而不单纯是一种仅仅着重研究后端的技术。  

## 安装  

APR 支持需要安装三个关键的原生组件：   

- APR 库  
- Tomcat 所用的 》JNI 包装器    
- OpenSSL 库

### Windows   


### Linux   

多数 Linux 分发版都会自带 APR 与 OpenSSL 包。JNI 包装器（litcnative）然后被编译。它依赖 APR、OpenSSL 与 Java 头。   

需要：   

- APR 1.2+ 开发头（libarp-1 dev package）  
- OpenSSL 》》》
- Java compatible JDK 1.4+》》   
- GNU 开发环境（gcc，make）  


## APR 组件  

当所有的库都正确安装好并且（如果加载失败，就会显示相关的库路径），Tomcat 连接器就会自动使用 APR。这里，连接器的配置跟通常的配置没什么不同，但会用一些特别的属性来配置 APR 组件。对于大多数用例来说，这些属性的默认值都已经非常适用了，根本不需要再加以微调。   

当启用 APR 时，Tomcat 同样也启用了下面这些功能：   

- 默认在所有平台安全会话 ID 生成（Linux 之外的平台需要随机数生成使用配置好的熵）。  
- 关于Tomcat 进程的内存使用和 CPU 使用情况的 OS 级统计，由status servlet所显示。  

## 配置 APR 生命周期侦听器（APR Lifecycle Listener）   

### AprLifecycleListener  

|属性|描述|
|----|---|  
|`SSLEngine`|所要使用的 `SSLEngine` 名称。off：不使用 SSL。on：使用 SSL，但没有特定引擎。默认值为 `on`。这将初始化原生的 SSL 引擎，然后使用 `SSLEnabled` 属性在连接器中》》》。范例：<br/>`<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />`<br/>请访问 [OpenSSL 官方网站](http://www.openssl.org)以详细了解 SSL 硬件引擎与制造商的相关信息。|



## 配置 APR 连接器   

### HTTP/HTTPS   

关于 HTTP 配置的相关信息，可查阅 [HTTP 连接器配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)。  
关于 HTTPS 配置的相关信息，可查阅 [HTTPS 连接器配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html#SSL_Support)。   

下面这个范例介绍了 SSL 连接器的声明：  

```   
<Connector port="443" maxHttpHeaderSize="8192"
                 maxThreads="150"
                 enableLookups="false" disableUploadTimeout="true"
                 acceptCount="100" scheme="https" secure="true"
                 SSLEnabled="true"
                 SSLCertificateFile="${catalina.base}/conf/localhost.crt"
                 SSLCertificateKeyFile="${catalina.base}/conf/localhost.key" />

```

### AJP

关于 AJP 配置的相关信息，可查阅 [AJP 连接器配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/ajp.html)。








