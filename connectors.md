# 连接器  

## 目录  

- 简介  
- HTTP  
- AJP  

## 简介  

选择适用于 Tomcat 的连接器是非常困难的。本文列出了目前版本的 Tomcat 所支持的连接器，可根据具体需要来选择使用。  

## HTTP  

HTTP 连接器是 Tomcat 默认配置好的，可立即使用。该连接器能实现最低的延时以及最佳的整体性能。  

对于集群化来说，必须安装**支持 Web 会话粘性**的 HTTP 负载均衡器，以便将流量导引至多个 Tomcat 服务器上。Tomcat 支持将 mod_proxy模块（可加载到 Apache HTTP server 2.0 中，到了 Apache HTTP server 2.2 时，成为默认包含的模块。）用作负载均衡器。不过要注意的是，HTTP 代理的性能往往要低于 AJP，所以 AJP 集群化才是首选方式。  

## AJP   

在仅使用一个服务器的情况下，使用位于 Tomcat 实例之前的原生 Web 服务器，往往要比使用带有默认 HTTP 连接器的 Tomcat 要低效得多，即使当大部分 Web 应用都只是由静态文件构成时，情况依然是这样。但假如基于某种原因，必须要使用原生的 Web 服务器时，那么使用 AJP 连接器，就会比使用 HTTP 代理在性能上更加优越。从 Tomcat 的角度来看，AJP 集群无疑是最高效的。除了这一点之外，AJP 集群与 HTTP 集群在功能上是等同的。  

这一版本的 Tomcat 所支持的原生连接器有：  

- JK 1.2.x + 任何支持的服务器；
- Apache HTTP Server 2.x 上的 启用了 AJP 的 mod_proxy 模块（在 Apache HTTP Server 2.2 上已成为默认配置模块）。


