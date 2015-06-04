# 附加组件 

## 目录  

- 简介  
- 下载  
- 构建
- 组件列表  
	- 完整的通用日志实现
	- Web 服务支持（JSR 109）
	- JMX Remote Lifecycle Listener

## 简介  

Tomcat 可以使用许多附件组件。这些附加组件有可能是由用户在需要时创建的，或者是从镜像下载站下载而来的。  

## 下载  

打开 [Tomcat 下载页面](http://tomcat.apache.org/download-80.cgi)，在“快速导航链接”（Quick Navigation Links）中点击“浏览”（browse）链接。在随后打开页面的 bin/extras 中可以找到附加组件。  


## 构建    

附加组件使用 Tomcat 标准的 Ant 脚本的 `extras` 目标构建而成。Ant 脚本位于 Tomcat 的资源包中。   

构建过程为：  

- 按照 [构建指令](http://tomcat.apache.org/tomcat-8.0-doc/building.html)，从资源包中构建一个 Tomcat 二进制文件（注意：附加组件的构建过程将会用到它，但以后不需要实际用到。）  
- 执行命令 `ant extras`，运行构建脚本。  
- 附加组件的 JAR 文件放到 `output/extras` 文件夹内。   
- 参考下文提到的文档来了解这些 JAR 文件的使用方法。     

## 组件列表   

### 完整的通用日志实现   

Tomcat 使用一个改名的包，硬编码的通用日志 API（commons-logging API）实现来使用 java.util.logging API。通用日志额外的组件构建了一个完备的包，重新命名的通用日志实现来替代 Tomcat 所提供的实现。参考[日志记录](http://tomcat.apache.org/tomcat-8.0-doc/logging.html)页面了解使用方法。	

### Web 服务支持（JSR 109）   

Tomcat 为可能用于解决 Web 服务引用的 JSR 109 提供了》》工厂。将生成的 catalina-ws.jar 以及 jaxrpc.jar 和 wsdl4j.jar（或 JSR 109 的另一个实现）放在 Tomcat 的 lib 文件夹下。  

用户应注意的是，wsdl4j.jar 遵循 CPL 1.0 许可，而不是 Apache License version 2.0。   

 

### JMX 远程生命周期侦听器（JMX Remote Lifecycle Listener）  

JMX 协议需要 JMX 服务器（在这里指的就是 Tomcat）在两个网络端口上进行侦听。其中一个端口通过配置可以是固定端口，而另外一个则是随机选择的。这就很难穿越防火墙来使用 JMX 。JMX 远端生命周期侦听器能实现两个固定端口，从而简化了穿越防火墙连接到 JMX 的过程。

