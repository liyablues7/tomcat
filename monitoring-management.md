# 监控与管理 Tomcat  

## 目录  

## 简介  

监控是系统管理中的重要环节。系统管理员的日常工作就包括：观察服务器的运行细节，获取统计数据，或者重新配置应用的某些内容。  

## 启用 JMX 远程》  

**注意**：该配置只适用于需用远程监控 Tomcat 的情况，使用同样的用户在本地监控 Tomcat 则不需要这么配置。  

Oracle 的网站上介绍了如何在 Java 6 上配置 JMX 远程：[http://docs.oracle.com/javase/6/docs/technotes/guides/management/agent.html](http://docs.oracle.com/javase/6/docs/technotes/guides/management/agent.html)。   

下面是在 Java 6 上的快速配置向导：  

将下列参数添加到 Tomcat 的 `setenv.bat` 脚本（具体详细信息请查看 [RUNNING.txt](http://tomcat.apache.org/tomcat-8.0-doc/RUNNING.txt)）。  

注意：该语法格式适用于 Windows 系统。命令行只能写在同一行中，包装起来更具可读性。如果 Tomcat 以 Windows 服务的形式运行，使用它的系统配置对话设置该服务的 java 选项。对于 UN\*X 系统来说，要将命令行开头的 `"set "` 去掉。   


```   

```    

1. 如果需要授权，则添加并修改下列命令：  

```
-Dcom.sun.management.jmxremote.authenticate=true
-Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access

```  

2. 编辑访问权限文件 `$CATALINA_BASE/conf/jmxremote.access`：  

```  
monitorRole readonly
controlRole readwrite

```

3. 编辑密码文件 `$CATALINA_BASE/conf/jmxremote.password`：  

```  
monitorRole tomcat
controlRole tomcat
```


**技巧**：密码文件应该是只读的，并且只能被运行 Tomcat 的操作系统用户所访问。  

**注意**：JSR 160 JMX 适配器在一个随机端口上打开了第二个数据通道。假如本地安装了防火墙，这就会出现问题。要想解决它，可以按照[侦听器](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html)文档中介绍的方法，配置一个 `JmxRemoteLifecycleListener`。  

## 利用 JMX 远程 Ant 任务来管理 Tomcat    

为了简化 JMX 的用法，加入了一些可能会与 antlib 使用的一系列任务。  

**antlib**：将 catalina-ant.jar 从 $CATALINA_HOME/lib 复制到 $ANT_HOME/lib。  

下面的例子展示了 JMX 存储器的用法。  

**注意**：为了提高可读性，这里将 `name` 属性值》。它必须写在同一行中，不允许带有空格。 

```  

```


**导入**：利用 `<import file="${CATALINA.HOME}/bin/catalina-tasks.xml" />` 导入 JMX 存取器项目，利用 jmxOpen、jmxSet、jmxGet、jmxQuery、jmxInvoke、jmxEquals 和 jmxCondition 来引用任务。  



## JMXAccessorOpenTask - JMX open connection task  

属性列表  

|属性|描述|默认值|
|---|---|---|  
|url|设定 JMX 连接 URL——`service:jmx:rmi:///jndi/rmi://localhost:8050/jmxrmi`|-|  
|host|设定主机，缩短长的 URL 格式|`localhost`|
|port|设定远程连接端口|8050|
|username|远程 JMX 连接用户名|-|  
|password|远程 JMX 连接密码|-|  
|ref|内部连接引用的名称。利用该属性，在同一个 Ant 项目中配置不止一个连接|jmx.server|  
|echo||`false`|
|if||-|
|unless||-|  

打开新的 JMX 连接的范例如下：  

```   

```  

从 URL 打开 JMX 连接的范例，with authorization and store at other reference, but only when property jmx.if exists and jmx.unless not exists  

```  

```   

**注意**：jmxOpen 任务中所有属性也存在于其他所有任务和条件中。   

## JMXAccessorGetTask: get attribute value Ant task

## JMXAccessorSetTask: set attribute value Ant task
## JMXAccessorInvokeTask: invoke MBean operation Ant task
## JMXAccessorQueryTask: query MBean Ant task
## JMXAccessorCreateTask: remote create MBean Ant task
## JMXAccessorUnregisterTask: remote unregister MBean Ant task
## JMXAccessorCondition: express condition
## JMXAccessorEqualsCondition: equals MBean Ant condition
## 使用 JMXProxyServlet















