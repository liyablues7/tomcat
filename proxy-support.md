## 目录  

## 简介  

使用 Tomcat 的标准配置，Web 应用可以请求服务器名称和端口号》。当 Tomcat 单独和 [HTTP/1.1 连接器](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)运行时，通常会报告指定在请求中的服务器名称，以及连接器正在侦听的端口号。servlet API 

- `ServletRequest.getServerName()` 返回接收请求的服务器主机名。  
- `ServletRequest.getServerPort()` 返回接收请求的服务器端口号。   
- `ServletRequest.getLocalName()` 返回接收请求的 IP 接口的主机名。  
- `ServletRequest.getLocalPort()` 返回接收请求的 IP 接口的端口号。  

当你在代理服务器后（或者配置成具有代理服务器特征行为的 Web 服务器）运行时，可能有时会更愿意管理通过这些调用产生的值。特别是，你一般会希望端口号反应指定在原始请求中的值，而非连接器所正在侦听的那个值。可以使用 `<Connector>` 元素中的 `proxyName` 和 `proxyPort` 属性来配置这些值。  

代理支持可以采取的形式有很多种。下面来讨论适用于一些通常情况的代理配置。  

## Apache 1.3 代理支持  

Apache 1.3 支持一种可选模式（`mod_proxy`），可以将 Web 服务器配置成代理服务器，从而将对于特定 Web 应用的请求转发给 Tomcat 实例，不需要配置 Web 连接器（比如说 `mod_jk`）。为了达成这一目标，需要执行下列任务：   

1. 配置 Apache，使其包含 `mod_proxy` 模块。如果是从源码开始构建，最简单的方式是在 `./configure` 命令行中包括 `--enable-module=proxy` 指令。  
2. 如果没有添加 `mod_proxy` 模块，则检查一下是否在 Apache 启动时加载了该模块，在 `httpd.conf` 文件中使用下指令：  

	```
	
	```
3. 在 `httpd.conf` 文件中包括两个指令。  分别为两个要转交给 Tomcat 的 Web 应用。例如，转交上下文路径 `/myapp` 处的应用，则需要如下指令：  

	```   
	
	```  
	
	上述指令告诉 Apache 将 `http://localhost/myapp/*` 形式的 URL 转交给在端口 8081 侦听的 Tomcat 连接器。  

4. 配置 Tomcat，使其包含一个特殊的 `<Connector>` 元素，并配置好相应的代理设置。范例如下所示：  

	```
	
	```   
	
	这将导致该 Web 应用内的 servlet 认为，所有代理请求都指向的是 80 端口处的 `www.mycompany.com`。  
	
5. 可以忽略 `<Connector>` 元素的 `proxyname` 属性，这是完全合法的。如果忽略，那么 `request.getServerName()` 返回值将是运行 Tomcat 的主机名——对于该例而言，它就是 `localhost`。   
6. 如果有一个 `<Connector>` （内嵌于同一 [Service](http://tomcat.apache.org/tomcat-8.0-doc/config/service.html) 元素之中）在 8080 端口处侦听。则针对这两个端口之中任何一个端口的请求将共享同样的虚拟主机和 Web 应用。  
7. 你可以利用所在操作系统的 IP 过滤功能来限制与 8081 端口的连接。（在该例中），使其  

跟8081端口的连接只能从运行 Apache 的服务器上  

8. 或者可以采用另外一种方式：可以设置一系列只能通过代理访问的 Web 应用，如下所示：   
	- 为代理端口配置另一个只包含一个 `<Connector>` 的 `<Service>`。  
	- 为能通过代理访问的虚拟主机和 Web 应用配置适宜的 Engine、Host，以及 Context 元素。   
	- 另外，还可以选择利用 IP 过滤器保护端口 8081，如前文所述。  

9. 当请求被 Apache 所代理处理时，Web 服务器会在访问日志中记下这些请求，所以通常应该禁止 Tomcat 本身执行访问记录。  

通过以上介绍的这种方式来代理请求，**所有**针对已经配置过的 Web 应用的请求（包括针对静态内容的请求）都将由 Tomcat 处理。你可以通过 Web 连接器 `mod_jk`（而不是 `mod_proxy`）来提高性能。通过配置 `mod_jk`，还可以让 Web 服务器提供静态内容服务，这些静态内容没有受到过滤器的处理，或者被 Web 应用部署描述符文件中所定义的安全限制所束缚。
  

	












## Apache 2.0 代理支持   

和 Apache 1.3 中的指令大致相同，只不过在 Apache 2.0 中，可以省略 `AddModule mod_proxy.c`。    
