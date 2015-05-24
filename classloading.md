## 概述     

与很多服务器应用一样，Tomcat 也安装了各种类加载器（那就是实现了 `java.lang.ClassLoader` 的类）。借助类加载器，容器的不同部分以及运行在容器上的 Web 应用就可以访问不同的仓库（保存着可使用的类和资源）。这个机制实现了 Servlet 规范 2.4 版（尤其是 9.4 节和 9.6 节）里所定义的功能。  

在 Java 环境中，类加载器的布局结构是一种父子树的形式。通常，类加载器被请求加载一个特定的类或资源时，它会先把这一请求委托给它的父类加载器，只有（一个或多个）父类加载器无法找到请求的类或资源时，它才开始查看自身的仓库。注意，Web 应用的类加载器模式跟这个稍有不同，下文将详细介绍，但基本原理是一样。

当 Tomcat 启动后，它就会创建一组类加载器，这些类加载器被布局成如下图所示这种父子关系，父类加载器在子类加载器之上：    

```
      Bootstrap
          |
       System
          |
       Common
       /     \
  Webapp1   Webapp2 ...

```



接下来，通过几节内容来详细说明每一个类加载器的特点，其中还将讲解这些加载器可使用的类和资源的来源。    



## 类加载器定义    


如上图所示，Tomcat 在初始化时会创建如下这些类加载器：   


- **Bootstrap** 这种类加载器包含 Java 虚拟机所提供的基本的运行时类，以及在 System Extensions 目录（`$JAVA_HOME/jre/lib/ext`）里的 JAR 文件中所有的类。**注意**：有些 JVM 会把它实现为多个类加载器，或者它根本不可见（作为类加载器）。》》  

- **System** 这种类加载器通常是根据 `CLASSPATH` 环境变量内容进行初始化的。所有的这些类对于 Tomcat 内部类以及 Web 应用来说都是可见的。不过，标准的 Tomcat 启动脚本（`$CATALINA_HOME/bin/catalina.sh` 或 `%CATALINA_HOME%\bin\catalina.bat`）完全忽略了 `CLASSPATH` 环境变量自身的内容，相反从下列仓库来构建系统类加载器：
	- `$CATALINA_HOME/bin/bootstrap.jar` 包含用来初始化 Tomcat 服务器的 `main()` 方法，以及它所依赖的类加载器实现类。  
	- `$CATALINA_BASE/bin/tomcat-juli.jar` 或 `$CATALINA_HOME/bin/tomcat-juli.jar` 日志实现类。其中包括 `java.util.logging` API 》》   
	- `$CATALINA_HOME/bin/commons-daemon.jar` [Apache Commons Daemon] 项目的类。该 JAR 文件并不存在于由 `catalina.bat` 或 `catalina.sh` 脚本所创建的 `CLASSPATH` 中，但是引用自》》》 
	

- **Common** 这种类加载器包含更多的额外类，它们对于Tomcat 内部类以及所有 Web 应用都是可见的。  
	
	通常，附加类**不能**放在这里。这种类加载器》》
	
	- `$CATALINA_BASE/lib` 中的解包的类和资源。   
	- `$CATALINA_BASE/lib` 中的 JAR 文件。  
	- `$CATALINA_HOME/lib` 中的解包类和资源。  
	- `$CATALINA_HOME/lib` 中的 JAR 文件。   

	默认，它包含以下这些内容：   
	
	- *annotations-api.jar* JavaEE 注释类。  
	- *catalina.jar* Tomcat 的 Catalina servlet 容器部分的实现。   
	- *catalina-ant.jar* Tomcat Catalina Ant 任务。   
	- *catalina-ha.jar* 高可用性包。   
	- *catalina-storeconfig.jar*   
	- *catalina-tribes.jar* 组通信包   
	- *ecj-\*.jar* Eclipse JDT Java 编译器    
	- *el-api.jar* EL 3.0 API  
	- *jasper.jar* Tomcat Jasper JSP 编译器与运行时  
	- *jasper-el.jar* Tomcat Jasper EL 实现   
	- *jsp-api.jar* JSP 2.3 API  
	- *servlet-api.jar* Servlet 3.1 API  
	- *tomcat-api.jar* Tomcat 定义的一些接口   
	- *tomcat-coyote.jar* Tomcat 连接器与工具类。    
	- *tomcat-dbcp.jar* 数据库连接池实现，将 Apache Commons Pool 和 Apache Commons DBCP 中的包重新命名》》   
	- *tomcat-i18n-\*\*.jar* 包含其他语言资源束的可选 JAR。因为默认的资源束也可以包含在每个单独的 JAR 文件中，所以如果不需要国际化信息，可以将其安全地移除。   
	- *tomcat-jdbc.jar* 一个数据库连接池替代实现，又被称作 Tomcat JDBC 池。详情参看[ JDBC 连接池文档](http://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)。   
	- *tomcat-util.jar* Apache Tomcat 多种组件所使用的常用类。
	- *tomcat-websocket.jar* WebSocket 1.1 实现
	- *websocket-api.jar* WebSocket 1.1 API  


- **WebappX** 为每个部署在单个 Tomcat 实例中的 Web 应用创建的类加载器。你的 Web 应用的 `/WEB-INF/classes` 目录中所有的解包类及资源，以及 `/WEB-INF/lib` 目录下 JAR 文件中的所有类及资源，对于该应用而言都是可见的，但对于其他应用来说则不可见。     

如上所述，Web 应用类加载器背离了默认的 Java 委托模式（根据 Servlet 规范 2.4 版的 9.7.2 Web Application Classloader一节中提供的建议）。当某个请求想从 Web 应用的 WebappX 类加载器中加载类时，该类加载器会**先**查看自己的仓库，而不是预先进行委托处理。There are exceptions。JRE 基类的部分类不能被重写。对于一些类（比如 J2SE 1.4+ 的 XML 解析器组件），可以使用 J2SE 1.4 支持的特性。最后，类加载器会显式地忽略所有包含 Servlet API 类的 JAR 文件，所以不要在 Web 应用包含任何这样的 JAR 文件。Tomcat 其他的类加载器则遵循常用的委托模式。   

因此，从 Web 应用的角度来看，加载类或资源时，要查看的仓库及其顺序如下：  

- JVM 的 Bootstrap 类  
- Web 应用的 `/WEB-INF/classes` 类   
- Web 应用的 `/WEB-INF/lib/*.jar` 类
- System 类加载器的类（如上所述）
- Common 类加载器的类（如上所述）  

如果 Web 应用类加载器配置有 `<Loader delegate="true"/>`，则顺序变为：   

- JVM 的 Bootstrap 类  
- System 类加载器的类（如上所述）  
- Common 类加载器的类（如上所述）  
- Web 应用的 `/WEB-INF/classes` 类     
- Web 应用的 `/WEB-INF/lib/*.jar` 类     


## XML解析器和 Java    

从 Java 1.4 版起，JRE 就包含了一个 JAXP API 和一个 XML 解析器。》   

在过去的 Tomcat 中，你只需在 Tomcat 库中简单地换掉 XML 解析器，就能改变所有 Web 应用使用的解析器。但对于现在版本的 Java 而言，这一技术并没有效果，因为通常的类加载器委托过程往往会优先选择 JDK 内部的实现，而 》》》   

Java 支持一种叫做“授权标准覆盖机制”，从而允许在》》 JCP 之外创建的 API的（例如 W3C 的 DOM 和 SAX）。它还可以用于更新 XML 解析器实现。关于此机制的详情，请参看 [http://docs.oracle.com/javase/1.5.0/docs/guide/standards/index.html](http://docs.oracle.com/javase/1.5.0/docs/guide/standards/index.html)。     

为了利用该机制，Tomcat 在启动容器的命令行中包含了系统属性设置 `-Djava.endorsed.dirs=$JAVA_ENDORSED_DIRS`。该选项的默认值为 `$CATALINA_HOME/endorsed`。但要注意，这个 `endorsed` 目录并非默认创建的。    


## Running under a security manager    

当在安全管理器下运行类时，类被允许加载的位置也是基于策略文件中的内容，详情可查看 [Security Manager HOW-TO](http://tomcat.apache.org/tomcat-8.0-doc/security-manager-howto.html)。       




