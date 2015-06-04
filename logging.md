## 目录  

- 简介  
	1. java 日志 API——java.util.logging  
	2. servlets logging API   
	3. Console    
	4. Acces 日志    
- 使用 java.util.logging  
	1. 文档引用     
	2. 生产环境使用中的注意事项      
- 使用 Log4j  


## <a href="#introduction">简介</a>   

Tomcat 的内部日志 使用 JULI 组件，这是一个 [Apache Commons 日志](http://commons.apache.org/proper/commons-logging/)的重命名的打包分支，默认被硬编码，使用 `java.util.logging` 架构。这能保证 Tomcat 内部日志与 Web 应用的日志保持独立，即使 Web 应用使用的是 Apache Commons Logging。   

假如想用另外的日志框架来替换 Tomcat 的内部日志系统，那么就必须采用一种能够保持完整的 Commons 日志机制的 JULI 实现，用它来替换通过硬编码使用 `java.util.logging` 的 JULI 实现。通常这种替代实现都是以[额外组件](http://tomcat.apache.org/tomcat-8.0-doc/extras.html替换中文页面》)的形式出现的。利用 Log4j 框架用于 Tomcat 内部日志的配置<a href = "#usingLog4j">如下文所示</a>。   

在 Apache Tomcat 上运行的 Web 应用可以使用：  

- 任何自选的日志框架。  
- 系统日志 API，`java.util.logging`。  
-  Java Servlets 规范所提供的日志 API，`javax.servlet.ServletContext.log(...)`。  

各个应用可以使用不同的日志框架，详情参见[类加载器](换中文页面》)。`java.util.logging` 则是例外。如果日志库直接或间接地用到了这一 API，那么 Web 应用就能共享使用它的元素，因为该 API 是由系统类加载器所加载的。  

  

### 1. java 日志 API——java.util.logging  
 
Apache Tomcat 本身已经实现了 `java.util.logging` API 的几个关键元素。这种实现就是 JULI。其中的关键组件是一个自定义的 LogManager 实现，它能分辨运行在 Tomcat 上的不同 Web 应用（以及它们所用的不同的类加载器），还能针对每一应用进行私有的日志配置。另外，当 Web 应用没能从内存中加载时，Tomcat 会给予它相应通知，从而清除相应的引用类，防止内存泄露。  

在启动 Java 时，通过提供特定的系统属性，可以启用 `java.util.logging` 实现。Apache Tomcat 启动脚本可以实现这个操作，但如果使用不同工具来运行 Tomcat（比如 jsvc，或者从某个 IDE 中运行 Tomcat），就必须自己来启用实现。  

关于 `java.util.logging` 实现的详细情况可以查阅 JDK 文档，具体位于 `java.util.logging` 包的相关 javadoc 页面中。  

关于 Tomcat JULI 的详细介绍见下文。  

### 2. servlets logging API  

Tomcat 内部日志能够处理对 `javax.servlet.ServletContext.log(...)` 的调用，从而写入日志消息。这种消息都被记录到一种特定类别中，命名方式如下：    

`org.apache.catalina.core.ContainerBase.[${engine}].[${host}].[${context}]`  

这种日志是依照 Tomcat 日志配置而执行的，无法在 Web 应用中重写。   

Servlets logging API 的问世要先于 Java 所提供的 `java.util.logging` API，所以，它无法提供太多的选项，比如无法用它来控制日志级别。然而需要注意的是，在 Tomcat 实现中， 对 `ServletContext.log(String)` 和 `GenericServlet.log(String)` 的调用都被记录在 INFO 级别。对 `ServletContext.log(String, Throwable)` 或 `GenericServlet.log(String, Throwable)` 的调用都被记录在 ERROR 级别。  

### 3. Console  

在 UNIX 系统下运行 Tomcat 时，控制台输出经常会重定向到 `catalina.out` 的文件中。通过一个环境变量，可以配置该文件（参见启动脚本）。  

写入 `System.err/out` 的任何内容都会被 `catalina.out` 文件所捕获。这些内容可能包括：  

- 由 `java.lang.ThreadGroup.uncaughtException(..)` 所输出的未捕获异常。  
- 线程转储，如果通过系统信号来请求它们。  

在 Windows 上以服务形式运行时，控制台输出也会被捕获及重定向，但文件名有所不同。  

Tomcat 默认的日志配置会将同样的消息写入控制台和一个日志文件中。这一特点非常有利于使用 Tomcat 进行开发，但往往并不适用于生产环境。  

老的应用可能还在使用 `System.out` 或 `System.err`，可以通过在 Context 元素上设置 `swallowOutput` 属性来调整。如该属性设为 `true`，那么在请求阶段对 `System.out/err` 的调用就会被拦截，它们的输出也会通过 `javax.servlet.ServletContext.log(...)` 调用反馈给日志系统。  

**注意**：`swallowOutput` 虽然是一个小技巧，但还是有局限性的：它需要直接调用 `System.out/err`，并且要在请求处理周期内完成。而且，它可能还并不适用于应用所创建的其他线程。不能将其用于拦截本身写入系统流的日志框架（它们可能早先已经启动，并且在重定向发生前就已经获取了对流的直接引用）。  

### 4. Acces 日志    

Access 日志功能相近，但还是有所不同。它是一个 `Valve`，使用自包含的逻辑来编写日志文件。访问日志的基本需求是以较低开销处理大型连续数据流，所以只能使用 Commomns Logging 来处理自身的调试消息。这种实现方法避免了额外的开销，并且可能具有较复杂的配置。请参考 [Valves 文档](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Access_Logging)了解更多配置详情，其中包含了各种报告格式。  

## 使用 java.util.logging（默认）  

JDK 所提供的默认 java.util.logging 实现功能太过局限，所以根本没有什么使用价值。其关键局限在于不能实现针对每一应用进行日志记录，因为配置是针对每一 VM 的。所以按照默认配置，Tomcat 会用 JULI 这种非常适用于容器的实现来代替默认的 LogManager 实现，从而避免了 LogManager 的缺点。   

跟标准 JDK 的 `java.util.logging` 一样，JULI 也支持同样的配置机制，或者使用编程方式，或者指定属性值。它与 `java.util.logging` 的不同在于，它可以分别设置每一个类加载器属性文件（能够启用简单的、便于重新部署的应用配置），属性文件还支持扩展构造，能够更加自由地定义 handle 并将其指定给 logger。  

JULI 是默认启用的，除了普通的全局 java.util.logging 配置之外，它支持每个类加载器配置。这意味着可以在下列层级来配置日志：  

- 全局范围。`${catalina.base}/conf/logging.properties` 文件。该文件通过由启动脚本设置的系统属性  `java.util.logging.config.file` 来指定。如果它不可读或没有配置，默认采用 JRE 中的 $`{java.home}/lib/logging.properties` 文件。  
- 在 Web 应用范围内。该文件为 `WEB-INF/classes/logging.properties`。   


JRE 中默认的 `logging.properties` 指定了 `ConsoleHandler`，用于将日志输出至 `System.err`。Tomcat 中默认的 `conf/logging.properties` 也添加了几个能够写入文件的 `FileHandlers`。

handler 的日志级别容差值默认为 INFO，取值范围为：SEVERE、WARNING、INFO、CONFIG、FINE、FINER、FINEST 或 ALL。你也可以从特殊的包中收集日志，然后为这种日志指定相应的级别。  

为了启用 部分 Tomcat 内部的调试日志功能，应该配置适合的 logger 和 handle 来使用 `FINEST` 或 `ALL` 级别。比如：


```  
org.apache.catalina.session.level=ALL
java.util.logging.ConsoleHandler.level=ALL

```


当启用调试日志功能时，建议将范围尽量缩小，因为该功能会产生大量信息。  

JULI 所使用的配置与纯 `java.util.logging` 所支持的配置基本相同，只不过使用了一些扩展，以便更灵活地配置 logger 和 handler。主要的差别在于：  

- handler 名称前可以加上前缀，所以同一类可以实例化出多个 handler。前缀是一个以数字开头的字符串，并以 `.` 结尾。比如 `22foobar.` 就是个有效的前缀。  
- 系统属性  


还有一些额外的实现类，它们可以与 Java 所提供的类一起使用。在这些类中，最著名的就是 `org.apache.juli.FileHandler`。   

`org.apache.juli.FileHandler` 支持日志缓存。日志缓存默认是没有启用的。使用 handler 的 `bufferSize` 属性可以配置它：属性值为 `0` 时，代表使用系统默认的缓存（通常使用 8k 缓存）；属性值小于 0 时，将在每个日志写入上强制使用 writer flush（将缓存区中的数据强制写出到系统输出）功能；属性值大于 0 时，则使用带有定义值的 BufferedOutputStream 类——但要注意的是，这也将应用于系统默认的缓存。   

以下是一个 `$CATALINA_BASE/conf` 中的 `logging.properties` 文件：  

```   
handlers = 1catalina.org.apache.juli.FileHandler, \
           2localhost.org.apache.juli.FileHandler, \
           3manager.org.apache.juli.FileHandler, \
           java.util.logging.ConsoleHandler

.handlers = 1catalina.org.apache.juli.FileHandler, java.util.logging.ConsoleHandler

############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################

1catalina.org.apache.juli.FileHandler.level = FINE
1catalina.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.FileHandler.prefix = catalina.

2localhost.org.apache.juli.FileHandler.level = FINE
2localhost.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
2localhost.org.apache.juli.FileHandler.prefix = localhost.

3manager.org.apache.juli.FileHandler.level = FINE
3manager.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
3manager.org.apache.juli.FileHandler.prefix = manager.
3manager.org.apache.juli.FileHandler.bufferSize = 16384

java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter


############################################################
# Facility specific properties.
# Provides extra control for each logger.
############################################################

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = \
   2localhost.org.apache.juli.FileHandler

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers = \
   3manager.org.apache.juli.FileHandler

# For example, set the org.apache.catalina.util.LifecycleBase logger to log
# each component that extends LifecycleBase changing state:
#org.apache.catalina.util.LifecycleBase.level = FINE

```  


下例是一个用于 servlet-examples 应用的 `WEB-INF/classes` 中的 `logging.properties` 文件：      

```  
handlers = org.apache.juli.FileHandler, java.util.logging.ConsoleHandler

############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################

org.apache.juli.FileHandler.level = FINE
org.apache.juli.FileHandler.directory = ${catalina.base}/logs
org.apache.juli.FileHandler.prefix = ${classloader.webappName}.

java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter

```  

### 1. 文档引用    

查看下列资源获取额外的详细信息：  

- [org.apache.juli 包的相关 Tomcat 文档](http://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/juli/package-summary.html)。   
-  [java.util.logging 包的 Oracle Java 6 文档](http://docs.oracle.com/javase/6/docs/api/java/util/logging/package-summary.html)。     

### 2. 生产环境使用中的注意事项   

可能需要注意以下方面：  

- 将 `ConsoleHandler` 从配置中移除。默认（多谢 `.handlers` 设置）日志会使用 `FileHandler` 和 `ConsoleHandler`。后者的输出经常会被捕获到一个文件中，比如 `catalina.out`。从而导致同一消息可能生成了两个副本。  
- 对于不使用的应用(比如 `host-manager`)，可以考虑将 `FileHandlers` 移除。  
- handler 默认使用系统缺省编码来写入日志文件，通过 `encoding` 属性可以修改设置，详情查看相关的 javadoc 文档。  
- 配置 [Access log](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Access_Logging) 。   

## <a name = "usingLog4j">使用 Log4j</a>  

前面介绍了用于 Tomcat 内部日志的 java.util.logging
，接下来本部分内容介绍如何通过配置 Tomcat 使用 [log4j](http://logging.apache.org/log4j/2.x/)。

**注意**：当你想重新配置 Tomcat 以便利用 log4j 来进行自身日志记录时，下面的步骤都是必需的；而当你只是想在自己的 Web 应用上使用 log4j 时，这些步骤则不是必需的。在后一种情况下，只需将 `log4j.jar` 和 `log4j.properties` 放到 Web 应用的 `WEB-INF/lib` 和 `WEB-INF/classes` 中即可。  

通过下列步骤可配置 log4j 输出 Tomcat 的内部日志：  

1. 创建一个包含下列配置的 `log4j.properties` 文件，将其保存到 `$CATALINA_BASE/lib`。  

	```
	log4j.rootLogger = INFO, CATALINA

	# Define all the appenders
	log4j.appender.CATALINA = org.apache.log4j.DailyRollingFileAppender
	log4j.appender.CATALINA.File = ${catalina.base}/logs/catalina
	log4j.appender.CATALINA.Append = true
	log4j.appender.CATALINA.Encoding = UTF-8
	# Roll-over the log once per day
	log4j.appender.CATALINA.DatePattern = '.'yyyy-MM-dd'.log'
	log4j.appender.CATALINA.layout = org.apache.log4j.PatternLayout
	log4j.appender.CATALINA.layout.ConversionPattern = %d [%t] %-5p %c- %m%n
	
	log4j.appender.LOCALHOST = org.apache.log4j.DailyRollingFileAppender
	log4j.appender.LOCALHOST.File = ${catalina.base}/logs/localhost
	log4j.appender.LOCALHOST.Append = true
	log4j.appender.LOCALHOST.Encoding = UTF-8
	log4j.appender.LOCALHOST.DatePattern = '.'yyyy-MM-dd'.log'
	log4j.appender.LOCALHOST.layout = org.apache.log4j.PatternLayout
	log4j.appender.LOCALHOST.layout.ConversionPattern = %d [%t] %-5p %c- %m%n
	
	log4j.appender.MANAGER = org.apache.log4j.DailyRollingFileAppender
	log4j.appender.MANAGER.File = ${catalina.base}/logs/manager
	log4j.appender.MANAGER.Append = true
	log4j.appender.MANAGER.Encoding = UTF-8
	log4j.appender.MANAGER.DatePattern = '.'yyyy-MM-dd'.log'
	log4j.appender.MANAGER.layout = org.apache.log4j.PatternLayout
	log4j.appender.MANAGER.layout.ConversionPattern = %d [%t] %-5p %c- %m%n
	
	log4j.appender.HOST-MANAGER = org.apache.log4j.DailyRollingFileAppender
	log4j.appender.HOST-MANAGER.File = ${catalina.base}/logs/host-manager
	log4j.appender.HOST-MANAGER.Append = true
	log4j.appender.HOST-MANAGER.Encoding = UTF-8
	log4j.appender.HOST-MANAGER.DatePattern = '.'yyyy-MM-dd'.log'
	log4j.appender.HOST-MANAGER.layout = org.apache.log4j.PatternLayout
	log4j.appender.HOST-MANAGER.layout.ConversionPattern = %d [%t] %-5p %c- %m%n
	
	log4j.appender.CONSOLE = org.apache.log4j.ConsoleAppender
	log4j.appender.CONSOLE.Encoding = UTF-8
	log4j.appender.CONSOLE.layout = org.apache.log4j.PatternLayout
	log4j.appender.CONSOLE.layout.ConversionPattern = %d [%t] %-5p %c- %m%n
	
	# Configure which loggers log to which appenders
	log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost] = INFO, LOCALHOST
	log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager] =\
  INFO, MANAGER
	log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager] =\
  INFO, HOST-MANAGER

	```  


2. 下载 [log4j](http://logging.apache.org/log4j/2.x/)(Tomcat 需要 1.2.x 版本)。  
3. 下载或构建 `tomcat-juli.jar` 和 `tomcat-juli-adapters.jar`，以便作为 Tomcat 的额外组件使用。详情参考 [Additional Components documentation](http://tomcat.apache.org/tomcat-8.0-doc/extras.html)。     

	`tomcat-juli.jar` 跟默认的版本不同。它包含所有的 Commons Logging 实现，从而能够发现 log4j 并配置自身。  
	
4. 如果希望全局性地使用 log4j，则如下配置 Tomcat：  
	- 将 `log4j.jar` 和 `tomcat-juli-adapters.jar` 从 extras 中放入 `$CATALINA_HOME/lib` 中。  
	- 用 extras 中的 `tomcat-juli.jar` 替换 `$CATALINA_HOME/bin/tomcat-juli.jar`。   

5. 如果是利用独立的 `$CATALINA_HOME` 和 `$CATALINA_BASE` 来运行 Tomcat，并想在一个 `$CATALINA_BASE` 中配置使用 log4j，则需要：  
	- 创建 `$CATALINA_BASE/bin` 和 `$CATALINA_BASE/lib` 目录——如果它们不存在的话。  
	- 将 extras 中的 `log4j.jar` 与 `tomcat-juli-adapters.jar` 从 extras 放入 `$CATALINA_BASE/lib` 中。  
	- 将 extras 中的 `tomcat-juli.jar` 转换成 `$CATALINA_BASE/bin/tomcat-juli.jar`。  
	- 如果使用[安全管理器](http://tomcat.apache.org/tomcat-8.0-doc/security-manager-howto.html)运行，则需要编辑 `$CATALINA_BASE/conf/catalina.policy` 文件来修改它，以便使用不同版本的 `tomcat-juli.jar`。  
	
	**注意**：其中的工作原理在于：优先将库加载到	 `$CATALINA_HOME` 中同样的库中。  
	
	**注意**：`tomcat-juli.jar` 之所以从 `$CATALINA_BASE`/bin 加载（而不是从 `$CATALINA_BASE`/lib 加载），是因为它是用作引导进程的，而引导类都是从 bin 加载的。  
	
6. 删除 `$CATALINA_BASE/conf/logging.properties`，以防止 `java.util.logging` 生成零长度的日志文件。  
7. 启动 Tomcat。  

log4j 配置沿用了默认的 java.util.logging 设置：管理器与主机管理器应用各自获得了独立的日志文件，而所有其余内容都发送到 `catalina.log` 日志文件中。  

你可以（也应该）更加挑剔地选择日志所包括的包。Tomcat 使用 Engine 和 Host 名称来定义 logger。比如，要想得到更详细的 Catalina localhost log，可以将它放在 `log4j.properties` 属性中。注意，在 log4j 基于 XML 的配置文件的命名惯例上，目前存在一些问题，所以建议使用所前所述的属性文件，直到未来版本的 log4j 允许使用这种惯例。   

```  
log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost]=DEBUG
log4j.logger.org.apache.catalina.core=DEBUG
log4j.logger.org.apache.catalina.session=DEBUG

```


**警告**：设定为 DEBUG 级别，会产生数以兆计的日志，从而拖慢 Tomcat 的启动。只有当需要调试 Tomcat 内部操作，才应该使用这一级别。  

你的 Web 应用当然应该使用各自的 log4j 配置。上面的配置是有效的。你可以将相似的 `log4j.properties` 文件放到你的 Web 应用的 `WEB-INF/classes` 目录中，将 log4jx.y.z.jar 放入 `WEB-INF/lib` 中。 然后指定包级别日志。这是基本的 log4j 配置方法，**不需要** Commons-Logging。更多选项可参考 [log4j 文档](http://logging.apache.org/log4j/docs/documentation.html)，该页面只是一种引导指南。  

**额外注意**：   

- 通过 Commons 类加载器将 log4j 库暴露给 Web 应用。详见[类加载器](》替换中文页面)文档。  
	正是由于这一点，使用 [Apache Commons Logging] 库的 Web 应用和库有可能自动会将 log4j 选为底层日志实现。  
- `java.util.logging` API 仍适用于直接使用它的 Web 应用。`${catalina.base}/conf/logging.properties` 文件仍然可被 Tomcat 启动脚本所引用。详情可查看<a href = "#introduction">本页的简介部分</a>。   
	
	如前面相关步骤所述，删除了 `${catalina.base}/conf/logging.properties` 文件，会导致 `java.util.logging` 回退到 JRE 默认的配置，从而使用 ConsoleHandler，然而却不创建任何标准日志文件。所以必须确保：在禁止标准机制之前，所有的日志文件必须是由 log4j 创建的。    
- **Access Log Valve** 和 ** ExtendedAccessLogValve** 使用它们自包含的日志实现，所以无法配置使用 log4j，详情参看 [Valves](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Access_Logging)。   