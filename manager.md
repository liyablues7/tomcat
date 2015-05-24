# 5. Tomcat Manager

## 本章目录        

<a href = "#overview">5.1 概述</a>  
5.2 配置 Manager 应用访问  
5.3 HTML 用户友好型界面  
5.4 支持的 Manager 命令  
	5.4.1 常用参数
	5.4.2 远端部署新的应用文件（WAR）  
	<a href ="#DeployANewApplicationfromaLocalPath">5.4.3 从本地路径部署新应用</a>     
		<a href = "#Deploy a previously deployed webapp">a. 部署之前部署过的 Web 应用</a>  
		b. 通过 URL 部署目录或 WAR   
		c. 从主机的 appBase 处部署目录或 WAR    
		d. 使用上下文配置 .xml 文件部署  
		e. 部署注意事项 
		f. 部署响应  
	5.4.4 列出当前已部署的应用  
	5.4.5 重新加载已存在的应用  
	5.4.6 列出系统及 JVM 的属性  
	5.4.7 列出可供使用的全局 JNDI 资源
	5.4.8 会话统计  
	5.4.9 Expire Sessions  
	5.4.10 开启已存在的应用  
	5.4.11 停止已存在的应用  
	5.4.12 取消对已存在应用的部署  
	5.4.13 发现内存泄露  
	5.4.14 连接器 SSL/TLS 诊断
	5.4.15 线程转储  
	5.4.16 虚拟机信息  
	5.4.17 保存配置  
5.5 服务器状态  
5.6 使用 JMX 代理 Servlet   
	5.6.1 什么是 JMX 代理 Servlet
	5.6.2 JMX 查询命令
	5.6.3 JMX 的 `get` 命令   
	5.6.4 JMX 的 `set` 命令   
	5.6.5 JMX 的 `invoke` 命令     
5.7 利用 Ant 执行 Manager 的命令     
	5.7.1 任务输出捕获  


## <a name = "overview">概述</a>     

很多生产环境都非常需要以下特性：在无需关闭或重启整个容器的情况下，部署新的 Web 应用或者取消对现有应用的部署。或者，即便在  Tomcat 服务器配置文件中没有指定 `reloadable` 的情况下，也可以请求重新加载现有应用。  

Tomcat 中的 Web 应用 Manager 就是来解决这些问题的，它默认安装在上下文路径：`/manager` 中，支持以下功能：  

- 用已上传的 WAR 文件内容部署新的 Web 应用。  
- 在服务器文件系统中指定上下文路径处部署新的 Web 应用。   
- 列出当前已部署的 Web 应用，以及这些应用目前的活跃会话。  
- 重新加载现有的 Web 应用，以便响应 `/WEB-INF/classes` 或 `/WEB-INF/lib` 中内容的更改。   
- 列出操作系统及 JVM 的属性值。  
- 列出可用的全局 JNDI 资源，它们将用于预备 `<ResourceLink>` 元素的部署工具中。`<ResourceLink>` 元素内嵌于 `<Context>` 部署描述中。
- 开启一个已停止的 Web 应用，从而使其再次可用。  
- 停止一个现有的 Web 应用，从而使其不可用，但并不取消对它的部署。   
- 取消对一个已部署 Web 应用的部署，删除它的文档库目录（除非它是从文件系统中部署的）。  

Tomcat 默认安装已经包含了 Manager。 将一个 Manager 应用实例的 `Context` 添加到一个新的主机中，`manager.xml` 上下文配置文件应放在 `$CATALINA_BASE/conf/[enginename]/[hostname]` 文件夹中。如下所示：   



```
<Context privileged="true" antiResourceLocking="false"
         docBase="${catalina.home}/webapps/manager">
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.0\.0\.1" />
</Context>

```  

如果将 Tomcat 配置成能够支持多个虚拟主机（网站），则需要对每个虚拟主机配置一个 Manager。   

Manager 应用的使用方式有以下三种：  

- 作为带有用户界面的应用，在浏览器中运行。在随后这个范例 URL 中，你可以将 `localhost` 替换为你的网站主机名称：`http://localhost:8080/manager/html`。   
- 只使用 HTTP 请求的一个功能最少的版本。它适合系统管理员通过创建脚本来进行使用。将命令指定在请求的 URI 中，响应是简单格式的文本（易于解析与处理）。详情查看 [支持的 Manager 命令](http://tomcat.apache.org/tomcat-8.0-doc/manager-howto.html#Supported_Manager_Commands)。     
- 用于 Ant 构建工具（1.4或更新版本）的一套方便的任务定义。详情参见[ 利用 Ant 执行 Manager 命令](http://tomcat.apache.org/tomcat-8.0-doc/manager-howto.html#Executing_Manager_Commands_With_Ant)。   


## 配置 Manager 应用访问


> 下文的描述将使用变量名 `$CATALINA_BASE` 来指代基本目录，和面将要处理的多数路径都是相对于该目录的。如果你还没有为多个 Tomcat 实例设置 CATALINA_BASE 目录，那么 $CATALINA_BASE 就将被设置为 $CATALINA_HOME（Tomcat 的安装目录）的值。      

Tomcat 以默认值运行是非常危险的，因为这能让互联网上的任何人都可以在你的服务器上执行 Manager 应用。因此，Manager 应用要求任何用户在使用前必须验证自己的身份，提供自己的用户名和密码，以及相应配置的 **manager-\*\*** 角色（角色名称根据所需的功能而定）。另外，**默认用户文件**（`$CATALINA_BASE/conf/tomcat-users.xml`）中的用户名称都没有指定角色名称，所以默认是不能访问 Manager 应用的。  

这些角色名称位于 Manager 应用的 `web.xml` 文件中。可用的角色包括：   

- **manager-gui** 能够访问 HTML 界面。    
- **manager-status** 只能访问“服务器状态”（Server Status）页面。  
- **manager-script** 能够访问文档中描述的适用于工具的纯文本界面，以及“服务器状态”页面。   
- **manager-jmx** 能够访问 JMX 代理界面以及“服务器状态”（Server Status）页面。   

HTML 界面不会遭受 CSRF（Cross-Site Request Forgery，跨站请求伪造）攻击，但纯文本界面及 JMX 界面却有可能无法幸免。这意味着，如果用户能够访问纯文本界面及 JMX 界面，那么在利用 Web 浏览器去访问 Manager 应用时，必须要万分谨慎。要想保持对 CSRF 免疫，则必须：   

- 在使用 Web 浏览器访问 Manager 应用时，假如用户具有 **manager-script** 或 **manager-jmx** 角色（比如为了测试纯文本界面或 JMX 界面），那么**必须**关闭所有的浏览器窗口，终止会话。如果不关闭浏览器，访问了其他站点，就可能会遭受 CSRF 攻击。     
- 建议永远不要将 **manager-script** 或 **manager-jmx** 角色授予那些拥有 **manager-gui** 角色的用户。   

> **注意** JMX 代理界面是 Tomcat 中非常高效的底层、类似于根级别的管理界面。如果用户知道了该调用的命令，就能实现大量行为，所以一定不要轻易授予用户 **manager-jmx** 角色。  

为了能够访问 Manager 应用，必须创建一个新的用户名/密码组合，并为之授予一种  **manager-\*\*** 角色，或者把一种 **manager-\*\*** 角色授予现有的用户名/密码组合。因为本文档的大部分内容都在描述纯文本界面的命令，所以为了将来讨论实例方便起见，把角色名称定为 **manager-script**。而涉及到具体如何配置用户名及密码，则是跟你所使用的[Realm 实现](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html)有关：       

- *UserDatabaseRealm* 加上 *MemoryUserDatabase* 或 *MemoryRealm*——*UserDatabaseRealm* 和 *MemoryUserDatabase* 配置在默认的 `$CATALINA_BASE/conf/server.xml` 文件中。*MemoryUserDatabase* 和 *MemoryRealm* 都会读取储存在 `CATALINA_BASE/conf/tomcat-users.xml` 里的 XML 格式文件，它可以用任何文本编辑器进行编辑。该文件会为每个用户定义一个 XML 格式的 `<user>` ，如下所示：   

	`<user username="craigmcc" password="secret" roles="standard,manager-script" />`    

	它定义了用户登录时所用的用户名和密码，以及他或她采用的角色名称。你可以把 **manager-script** 角色添加到由逗号分隔的 `roles` 属性中，从而将该角色赋予一个或多个用户，也可以利用指定角色来创建新的用户。   
 
- *DataSourceRealm* 或 *JDBCRealm*——用户和角色信息都存储在一个经由 JDBC 访问的数据库中。按照当前环境的标准流程，将 **manager-script** 角色赋予一个或多个用户，或者利用该角色创建一个或多个新用户。     
    
- *JNDIRealm*——你的用户和角色信息被存储在经由 LDAP 访问的一个目录服务器中。按照当前环境的标准流程，为一个或更多的现有用户添加 **manager-script** 角色，和（/或）利用指定角色创建一个或更多的新用户。  


在下一节，当你第一次尝试使用 Manager 的一个命令时，将会使用**基本**验证进行登录。用户名和密码的具体内容并不重要，只要它们能够证明，用户数据库中拥有 **manager-script** 角色的用户是有效用户，我们的目的就达到了。   

除了密码限制访问之外，Manager 还可以配置 `RemoteAddrValve` 和 `RemoteHostValve` 这两个参数，分别通过 **远程 IP 地址** 或远程主机名来进行限制访问。详情可查看 [Valve 文档](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Remote_Address_Filter)。下列范例是通过 IP 地址来限制访问本地主机： 


```
<Context privileged="true">
         <Valve className="org.apache.catalina.valves.RemoteAddrValve"
                allow="127\.0\.0\.1"/>
</Context>  

```

## 易用的 HTML 界面   

Manager 应用易用的 HTML 界面位于：   

`http://{host}:{port}/manager/html`    

正像前面讲过的那样，需要被授予 **manager-gui** 角色才能访问它。关于这个界面，还有一个独立的文档，请访问以下页面：

- [Manager 的 HTML 界面文档》》需要链接附属文档](http://tomcat.apache.org/tomcat-8.0-doc/html-manager-howto.html)          

HTML 界面可免受 CSRF（跨站请求伪造）攻击。对 HTML 页面的每次访问都会生成一个随机令牌，存储在会话中，包含在页面的所有链接中。如果你的下一个操作没有正确的令牌值，操作就会被拒绝。如果令牌过期，可以从主页或者 Manager 的 *List Applications*（列出的应用）页面 重新开始。  


## Manager 支持的命令   


Manager 应用能够处理的命令都是通过下面这样的请求 URL 来指定的：     

`http://{host}:{port}/manager/text/{command}?{parameters}`  
 
`{host}` 和 `{port}` 分别代表运行 Tomcat 服务器所在的主机名和端口号。`{command}` 代表所要执行的 Manager 命令。`{parameters}` 代表该命令所专有的查询参数。在后面的实例中，可以为你的安装自定义适当的主机和端口。     

这些命令通常是被 HTTP `GET` 请求执行的。`/deploy` 命令有一种能够被 HTTP `PUT` 请求所执行的形式。    

## 常见参数      

多数 Manager 命令都能够接受一个或多个查询参数，这些查询参数如下所示：     

- **path** 要处理的 Web 应用的上下文路径（包含前面的斜杠）。要想选择 ROOT Web 应用，指定 `/` 即可。**注意**：无法对 Manager 应用自身执行管理命令。  
- **version** [并行部署](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 所用的 Web 应用版本号。   
- **war** Web 应用归档（WAR）文件的 URL，或者含有 Web 应用的目录路径名，或者是上下文配置 .xml 文件。你可以按照以下任一格式使用 URL：   
	- **file:/absolute/path/to/a/directory** 解压缩后的 Web 应用所在的目录的绝对路径。它将不做任何改动，直接附加到你所指定的上下文路径上。  
	- **file:/absolute/path/to/a/webapp.war** Web 应用归档（WAR）文件的绝对路径。**只**对 `/deploy` 命令有效，也是该命令所唯一能接受的格式。   
	- **jar:file:/absolute/path/to/a/warfile.war!/** 本地 WAR 文件的 URL。另外，为了能够完整引用一个JAR 文件，你可以采用 `JarURLConnection` 类的任何有效语法。   
	- **file:/absolute/path/to/a/context.xml** Web 应用上下文配置 .xml 文件的绝对路径，上下文配置文件包含着上下文配置元素。  
	- **directory** 主机应用的基本目录中的 Web 应用的目录名称。  
	- **webapp.war** 主机应用的基本目录中的 WAR 文件的名称。   

每个命令都会以 `text/plain` 的形式（比如，没有 HTML 标记的纯 ASCII 码文本 ）返回响应，从而便于开发者与程序阅读。响应的第一行以 `OK` 或 `FAIL` 开头，表明请求的命令是否成功。如果失败，响应第一行随后部分就会带有遇到问题的描述。一些包含其他信息行的命令会在下文予以介绍。     

**国际化说明** Manager 应用会在资源包中查找消息字符串，所以这些字符串可能已经转化为你所用平台的语言版本了。下文的范例展示的全都是消息的英文版本。     

## 远程部署新应用   

`http://localhost:8080/manager/text/deploy?path=/foo`   

将作为请求数据指定在 HTTP `PUT` 请求中的 Web 应用归档文件（WAR）上传，将它安装到相应虚拟主机的 `appBase` 目录中，启动，使用目录名或不带 .war 后缀的 WAR 文件名作为路径。稍后，可以通过 `/undeploy` 取消对应用的部署，相应的应用目录也会被删除。   

该命令通过 HTTP `PUT` 请求来执行。       

通过在 `/META-INF/context.xml` 中包含上下文配置 XML 文件，.WAR 文件能够包含 Tomcat 特有的部署配置信息。   

URL 参数包括：  

- `update` 设置为 true 时，任何已有的更新将会首先取消部署。默认值为 false。  
- `tag` 指定一个标签名称。这个参数能将已部署的 Web 应用与标签连接起来。如果 Web 应用被取消部署，则以后在需要重新部署时，只需使用标签就能实现。   

**注意**：该命令是 `/undeploy` 命令在逻辑上是对立的。   

如果安装或启动成功，会接受到这样一个响应：   

`OK - Deployed application at context path /foo`  

否则，响应会以 `FAIL` 开始，并包含一个错误消息。出现问题的可能原因为：   

- *Application already exists at path /foo*   
	当前运行的 Web 应用的上下文路径必须是唯一的。否则，必须使用这一上下文路径取消对现有 Web 应用的部署，或者为新应用选择另外一个上下文路径。`update` 参数可以指定为 URL 中的参数。true 值可避免这种错误。这种情况下，会在部署前，取消对现有应用的部署。     
	
- *Encountered exception*  
	遇到试图开启新的 Web 应用。可查看 Tomcat 日志了解详情。但有可能是在解析 `/WEB-INF/web.xml` 文件时遇到了问题，或者在初始化应用的事件侦听器与过滤器时出现遗失类的情况。



## <a name = "DeployANewApplicationfromaLocalPath">5.4.3 从本地路径处部署新的应用</a>     

部署并启动一个新的 Web 应用，附加到指定的上下文 `path` 上（不能被其他 Web 应用同时使用）。该命令与 `/undeploy` 在逻辑上是对立的。    

该命令由一个 HTTP `GET` 请求执行。部署命令的应用方式有很多种。  

### <a name = "Deploy a previously deployed webapp">a. 部署之前部署过的 Web 应用</a>

`http://localhost:8080/manager/text/deploy?path=/footoo&tag=footag`  

用来部署之前曾通过 `tag` 属性部署过的 Web 应用。注意，Manager 应用的工作目录包含之前部署过的 WAR 文件；如果清除它则将使部署失败。  

### b. 通过 URL 部署一个目录或 WAR 文件  

部署位于 Tomcat 服务器上的 Web 应用目录或 .war 文件。如果没有指定上下文路径参数 `path`，就会把目录名或未带 .war 后缀的 war 文件名当做路径来使用。`war` 参数指定了目录或 WAR 文件的URL（也包含 `file:`格式）。引用 WAR 文件的 URL 所采用的语法详见 `java.net.JarURLConnection` 类的Java文档页面。只使用引用了整个 WAR 文件的 URL。   

下面这个实例中，Web 应用位于 Tomcat 服务器上的 `/path/to/foo` 目录中，被部署为上下文路径为 `/footoo` 的  Web 应用。   

`http://localhost:8080/manager/text/deploy?path=/footoo&war=file:/path/to/foo`

在下例中，Tomcat 服务器上的 .war 文件 `/path/to/bar.war` 被部署为上下文路径为 `/bar` 的 Web 应用。注意，这里没有 `path` 参数，因此上下文路径默认为没有 .war 后缀的 WAR 文件名。   

`http://localhost:8080/manager/text/deploy?war=jar:file:/path/to/bar.war!/`  


### c. 从主机的 `appBase` 目录中部署一个目录或 WAR   

对位于主机 `appBase` 目录中的 Web 应用目录或 .war 文件进行部署。目录名或没有 .war 后缀名的 WAR 文件名被用作上下文路径名。  

在下面的范例中，Web 应用位于 Tomcat 服务器中主机 `appBase` 目录下名为 `foo` 的子目录中，被部署为上下文路径名为 `/foo` 的 Web 应用。注意，用到的上下文路径名就是 Web 应用的目录名。   

`http://localhost:8080/manager/text/deploy?war=foo`  

在下面的范例中，位于主机 `appBase` 目录中的 `bar.war` 文件被部署为上下文名为 `/bar` 的 Web 应用。   

`http://localhost:8080/manager/text/deploy?war=bar.war`  

### d. 使用上下文配置 .xml 文件来进行部署   

如果主机的 `deployXML` 标志设定为 true，就可以使用上下文配置 .xml 文件以及一个可选的 .war  文件（或 Web 应用目录）来进行 Web 应用部署。在使用上下文 .xml 文件配置文件进行部署时，不会用到上下文路径参数 `/path`。   

上下文配置 .xml 文件包含用于 Web 应用上下文的有效 XML，就好像是在 Tomcat 的 `server.xml` 配置文件中进行配置一样。范例如下：   

```
<Context path="/foobar" docBase="/path/to/application/foobar">
</Context>

```

可选的 `war` 参数被设定为指向 Web 应用的 .war 文件或目录的 URL，它会覆盖掉上下文配置 .xml 文件中的任意 `docBase`。   

在下面这个实例中，使用上下文配置 .xml 文件部署 Web 应用：   

`http://localhost:8080/manager/text/deploy?config=file:/path/context.xml`  


在下面这个应用部署范例中，使用了上下文配置 .xml 文件和位于服务器中的 Web 应用的 .war 文件。   

`http://localhost:8080/manager/text/deploy
 ?config=file:/path/context.xml&war=jar:file:/path/bar.war!/`



### e. 部署中的一些注意事项  

如果主机配置中将 `unpackWARs` 设为 true，而且你部署了一个 war 文件，那么这个 war 文件将解压缩至主机的 `appBase` 目录下的一个目录中。   

如果应用的 war 文件或目录安装在主机的 `appBase` 目录中，那么或者主机应该被部署为 `autoDeploy` 为 true，或者上下文路径必须匹配目录名或不带 .war 后缀的 war 文件名。   

为了避免不可信用户作出对 Web 应用的侵害，主机的 `deployXML` 标志可以设为 false。这能保证不可信用户通过使用 XML 配置文件来部署 Web 应用，也能阻止他们部署位于其主机 `appBase` 之外的应用目录或 .war 文件。      


### f. 部署响应   

如果安装及启动都正常，会得到以下这样的响应：  

`OK - Deployed application at context path /foo`  

否则，响应会以 `FAIL` 开头并包含一些错误消息，引起问题的原因可能有以下几种：   

- *Application already exists at path /foo*   
	当前运行的 Web 应用的上下文路径必须是唯一的。否则，必须使用这一上下文路径取消对现有 Web 应用的部署，或者为新应用选择另外一个上下文路径。`update` 参数可以指定为 URL 中的参数。true 值可避免这种错误。这种情况下，会在部署前，取消对现有应用的部署。     

- *Document base does not exist or is not a readable directory*   
	通过 `war` 指定的 URL 必须要确认服务器中的某个目录含有解压缩后的 Web 应用，包含该应用的WAR 文件的绝对 URL 。更正 `war` 参数所提供的值。  
	
- *Encountered exception*  
	遇到试图开启新 Web 应用。可查看 Tomcat 日志了解详情。但有可能是在解析 `/WEB-INF/web.xml` 文件时遇到了问题，或者在初始化应用的事件侦听器与过滤器时出现遗失类的情况。

- *Invalid application URL was specified*
	所指定的指向目录或 Web 应用的 URL 无效。有效的 URL 必须以 `file:` 开始，用于 WAR 文件的 URL 必须以 .war 结尾。   

- *Invalid context path was specified*  
	上下文路径必须以斜杠字符开始，引用 ROOT 应用必须使用 `/`。   
	
- *Context path must match the directory or WAR file name*   
	如果应用的 .war 文件或目录安装在主机的 `appBase` 目录，那么或者主机应该被部署为 `autoDeploy` 为 true，或者上下文路径必须匹配目录名或不带 .war 后缀的 war 文件名。     
	
- *Only web applications in the Host web application directory can be installed*
	如果主机的 `deployXML` 标志为设为 false，那么当要部署的 Web 应用目录或 .war 文件位于主机 `appBase` 目录之外时，就会产生这样的错误。  
	
## 5.4.4 列出当前已部署的应用   

`http://localhost:8080/manager/text/list`  

列出当前所有部署的 Web 应用的上下文路径、当前状态（`running` 或 `stopped`），以及活跃会话。开启 Tomcat 后，一般立刻会产生如下这样的响应：   

```
OK - Listed applications for virtual host localhost
/webdav:running:0
/examples:running:0
/manager:running:0
/:running:0

```
<sup>Listed applications for virtual host localhost：列出虚拟主机本地主机的所有应用。</sup>


## 5. 重新加载一个现有应用  

`http://localhost:8080/manager/text/reload?path=/examples`  

标记一个现有应用，关闭它并重新加载。这一功能的适用情况为：当 Web 应用上下文不能重新加载；你已经更新了 `/WEB-INF/classes` 目录中的类和属性文件时；或者当你在 `/WEB-INF/lib` 目录添加或更新了 jar 文件。   

> **注意**：在重新加载时，Web 应用配置文件 `/WEB-INF/web.xml`无法重新读取。如果对 web.xml 文件作出改动，则必须停止并启动 Web 应用。   

如果命令成功执行，应得如下所示的响应：   

`OK - Reloaded application at context path /examples`  

否则，返回的响应以 `FAIL` 开头，并包含相关的错误消息。引起问题的可能原因有以下几种：   

- *Encountered exception*  
	遇到试图重启 Web 应用的异常。可查看 Tomcat 日志了解详情。   
	
- *Invalid context path was specified*  
	上下文路径必须以斜杠开始，引用 ROOT Web 应用必须使用 `/`。   
	
- *No context exists for path /foo*  
	在所指定的上下文路径中没有发现部署好的应用。  
	
- *No context path was specified*    
	需要 `path` 参数。   
	
- *Reload not supported on WAR deployed at path /foo*  
	当前，如果主机配置为不解压缩 WAR 文件时，直接从一个 WAR 文件安装 Web 应用时，不支持重新加载应用（以便使类或 web.xml 文件中的更改生效）。由于只有在从已解压缩目录安装 Web 应用时才生效，所以在使用 WAR 文件时，应该先取消对应用的部署，然后重新部署该应用，以便使更改生效。   

	
	
## 6. 列出系统及 JVM 属性  

`http://localhost:8080/manager/text/serverinfo`    

列出 Tomcat 版本、操作系统以及 JVM 属性等相关信息。   

如果出现错误，响应会以 `FAIL` 开始并包含一系列错误消息，导致错误的可能原因包括有：   

- *Encountered exception*  
	碰到异常试图列举系统属性。可查看 Tomcat 日志了解详情。   
	
## 7. 列出可能的全局 JNDI 资源   

`http://localhost:8080/manager/text/resources[?type=xxxxx]`  

列出上下文配置文件资源链接中所使用的全局 JNDI 资源。如果指定 `type` 请求参数，参数值必须是所需资源类型的完整 Java 类名（比如，指定 `javax.sql.DataSource` 获取所有可用的 JDBC 数据资源的名称）。如果没有指定 `type` 请求参数，则将返回所有类型的资源。   

根据是否指定了 `type` 请求参数，常见响应的第一行将如下所示：    

`OK - Listed global resources of all types`  

或  

`OK - Listed global resources of type xxxxx`  

后面将每个资源都单列一行，每一行内的字段都由冒号（`:`）分隔，如下所示：      

- *Global Resource Name* 全局 JNDI 资源的名称，将用在 `<ResourceLink>` 元素的 `global` 属性中。  
- *Global Resource Type* 该全局 JNDI 资源的完整描述的 Java 类名。   

如果出现错误，响应将会以 `FAIL` 开始，并包含一个错误消息。出错的原因可能包括以下几方面：   

- *Encountered exception*  
	碰到异常试图列举 JNDI 资源，可查看 Tomcat 日志了解详情。  

- *No global JNDI resources are available*  
	运行的 Tomcat 服务器没有配置全局 JNDI 资源。   
	
## 8. 会话统计  

`http://localhost:8080/manager/text/sessions?path=/examples`   

显示 Web 应用默认的会话超时，当前活跃会话在一分钟范围内实际的超时次数。比如，重启 Tomcat 并随后执行 `/examples` Web 应用中的一个 JSP 范例，有可能得到如下信息：  

```
OK - Session information for application at context path /examples
Default maximum session inactive interval 30 minutes
<1 minutes: 1 sessions
1 - <2 minutes: 1 sessions

```

## 9. 过期会话   

`http://localhost:8080/manager/text/expire?path=/examples&idle=num`  

显示会话统计信息（比如上面的 `/sessions` 命令）以及超出 `num` 所指定的分钟数的过期会话。要想使所有会话都过期，可使用 `&idle = 0`。   

```
OK - Session information for application at context path /examples
Default maximum session inactive interval 30 minutes
1 - <2 minutes: 1 sessions
3 - <4 minutes: 1 sessions
>0 minutes: 2 sessions were expired

```
	
实际上，`/sessions` 和 `/expire` 是同一个命令的两种异名，唯一不同之处在于 `idle` 参数。     

## 10. 开启一个现有应用       

`http://localhost:8080/manager/text/start?path=/examples`    

标记一个已停止的应用，重新开启它，使其再次可用。停止并随后重新开启应用有时显得非常重要，比如当应用所需的服务器暂时变得不可用时。通常情况下，与其让用户频繁碰到数据库异常，倒不如停止基于该数据库的 Web 应用运行。     

如果该命令成功执行，将得到类似如下的响应：   

`OK - Started application at context path /examples`  

否则，将返回出错响应，该响应以 `FAIL` 开头，并包含一些错误。出错原因可能是由于：   

- *Encountered exception*  
	碰到异常情况，试图开启 Web 应用。可检查 Tomcat 日志了解详情。   

- *Invalid context path was specified*  
	上下文路径必须以斜杠字符开始，引用 ROOT Web 应用必须使用反斜杠（`/`）。  
	
- *No context exists for path /foo*  
	在指定的上下文路径处没有部署的应用。  

- *No context path was specified*  
	需要指定 `path` 参数。   
	
## 11. 停止已有应用     

`http://localhost:8080/manager/text/stop?path=/examples`  

标记现有应用，使其不可用，但仍使其处于已部署状态。当应用停止时，任何请求都将得到著名的 HTTP 404错误。在应用列表中，该应用将显示为“stopped”。  

如果该命令成功执行，将得到类似如下的响应：   

`OK - Stopped application at context path /examples`  


否则，将返回出错响应，它以 `FAIL` 开头，并包含一个出错消息，可能导致出误的原因包括：   

- *Encountered exception*  
	碰到异常情况，试图开启 Web 应用。可检查 Tomcat 日志了解详情。    
	
- *Invalid context path was specified*  
	上下文路径必须以斜杠字符开始，引用 ROOT Web 应用必须使用反斜杠（`/`）。    
	
- *No context exists for path /foo*	在指定的上下文路径处没有部署的应用。     

- *No context path was specified*  
	需要指定 `path` 参数。   	 
	

## 12. 取消对现有应用的部署  

`http://localhost:8080/manager/text/undeploy?path=/examples`  

警告：**该命令将删除虚拟主机 `appBase` 目录（通常是 webapps ）中的所有 Web 应用。**该命令将从未解压缩（或已解压缩）的 .WAR 式部署中，以及 `$CATALINA_BASE/conf/[enginename]/[hostname]/` 中以 XML 格式保存的上下文描述符中，删除应用的 .WAR 文件及目录。如果你只是想让某个应用暂停服务，则应该使用 `/stop` 命令。    

标记一个已有的应用，将其恰当地关闭，从 Tomcat 中移除（从而使得以后可以重新使用该上下文路径）。另外，如果文档根目录位于虚拟主机的 `appBase` 目录（通常是 webapps）中，则它也将被移除。该命令是 `/deploy` 的逆向命令。   

如果该命令成功执行，将得到类似如下的响应：       

`OK - Undeployed application at context path /examples`  

否则，将返回出错响应，它以 `FAIL` 开头，并包含一个出错消息，可能导致出误的原因包括：   

- *Encountered exception*  
	碰到异常情况，试图取消对某个 Web 应用的部署。可检查 Tomcat 日志了解详情。       

- *Invalid context path was specified*  
	上下文路径必须以斜杠字符开始，引用 ROOT Web 应用必须使用反斜杠（`/`）。    
 
- *No context exists for path /foo*	在指定的上下文路径处没有部署的应用。     

- *No context path was specified*  
	需要指定 `path` 参数。   	 


 

## 13. 寻找内存泄露  

`http://localhost:8080/manager/text/findleaks[?statusLine=[true|false]]`  


**寻找内存泄露的诊断将触发一个彻底的垃圾回收（GC）方案，所以如果在生产环境中使用它，需要非常谨慎才行。**    

寻找内存泄露的诊断会试图确认已导致内存泄露的 Web 应用（当其处于停止、重新加载，以及被取消部署状态时）。通常由一种分析器来确认结论。诊断使用了由 StandardHost（标准主机）实现所提供的附加功能。如果使用的是没有扩展自 StandHost 的自定义主机，则该诊断无法生效。  

已有一些文档介绍，从 Java 代码中显式地触发彻底的垃圾回收方案是不可靠的。此外，在不同的 JVM 中，也有很多选项禁止显式触发垃圾回收，比如像 `-XX:+DisableExplicitGC`。 如果你需要确认诊断是否成功地实现了彻底的垃圾回收，可以使用 GC 日志、JConsole 分析器，或其他类似工具。  

如果该命令成功执行，将得到类似如下的响应：     

`/leaking-webapp`  

如果你希望在响应中看到状态行，那么可以在请求中加入 `statusLine` 查询参数，并将其设定为 `true`。   

对于已停止运行、被重新加载或被取消部署的Web 应用，由于之前运行所用到的类可能仍然加载在内存中，从而会造成内存泄露。响应将把这种应用的每个上下文路径都单列一行。如果应用被重新加载了数次，就可能会列出几次。  

如果命令并没有成功执行，响应将以 `FAIL` 开头，并包含一个错误消息。   


## 连接器 SSL/TLS 诊断  

`http://localhost:8080/manager/text/sslConnectorCiphers`  

SSL 连接器/加密诊断会列出当前每一连接器所配置的 SSL/TLS 加密算法。对于 BIO 和 NIO，将列出每个加密算法套件的名称；对于 APR，则返回 SSLCipherSuite 的值。    

响应类似如下所示：   

```
OK - Connector / SSL Cipher information
Connector[HTTP/1.1-8080]
  SSL is not enabled for this connector
Connector[HTTP/1.1-8443]
  TLS_ECDH_RSA_WITH_3DES_EDE_CBC_SHA
  TLS_DHE_RSA_WITH_AES_128_CBC_SHA
  TLS_ECDH_RSA_WITH_AES_128_CBC_SHA
  TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA
  ...


```


## 线程转储  

`http://localhost:8080/manager/text/threaddump`  

编写 JVM 线程转储。   

响应类似如下所示：  

```
OK - JVM thread dump
2014-12-08 07:24:40.080
Full thread dump Java HotSpot(TM) Client VM (25.25-b02 mixed mode):

"http-nio-8080-exec-2" Id=26 cpu=46800300 ns usr=46800300 ns blocked 0 for -1 ms waited 0 for -1 ms
   java.lang.Thread.State: RUNNABLE
        locks java.util.concurrent.ThreadPoolExecutor$Worker@1738ad4
        at sun.management.ThreadImpl.dumpThreads0(Native Method)
        at sun.management.ThreadImpl.dumpAllThreads(ThreadImpl.java:446)
        at org.apache.tomcat.util.Diagnostics.getThreadDump(Diagnostics.java:440)
        at org.apache.tomcat.util.Diagnostics.getThreadDump(Diagnostics.java:409)
        at org.apache.catalina.manager.ManagerServlet.threadDump(ManagerServlet.java:557)
        at org.apache.catalina.manager.ManagerServlet.doGet(ManagerServlet.java:371)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:618)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:725)
...

```



## 虚拟机（VM）相关信息  

`http://localhost:8080/manager/text/vminfo`  

写入一些关于 Java 虚拟机（JVM）的诊断信息。    

响应类似如下所示：   

```
OK - VM info
2014-12-08 07:27:32.578
Runtime information:
  vmName: Java HotSpot(TM) Client VM
  vmVersion: 25.25-b02
  vmVendor: Oracle Corporation
  specName: Java Virtual Machine Specification
  specVersion: 1.8
  specVendor: Oracle Corporation
  managementSpecVersion: 1.2
  name: ...
  startTime: 1418012458849
  uptime: 393855
  isBootClassPathSupported: true

OS information:
...

```


## 保存配置信息  

`http://localhost:8080/manager/text/save`  

如果不指定任何参数，该命令将把服务器的当前配置信息保存到 server.xml 中。已有的配置信息 .xml 文件将被重命名，作为必要时的备份文件。   

如果指定了 `path` 参数，而且该参数与已部署应用的路径相匹配，那么该 Web 应用的配置将保存为一个命名恰当的上下文 .xml 文件中，位于当前主机的 `xmlBase` 中。  

要想使用该命令，则 StoreConfig MBean 必须存在。通常需要用 [StoreConfigLifecycleListener](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html#StoreConfig_Lifecycle_Listener_-_org.apache.catalina.storeconfig.StoreConfigLifecycleListener)来配置。  

如果命令不能成功执行，响应将以 `FAIL` 开头，并包含一个错误消息。   

## 服务器状态  

可从下面这些链接中观察有关服务器的状态信息。任何一个 **manager-**** 角色都能访问这一页面。   

```
http://localhost:8080/manager/status
http://localhost:8080/manager/status/all

```

上面是用 HTML 格式显示服务器状态信息的命令。   

```
http://localhost:8080/manager/status?XML=true
http://localhost:8080/manager/status/all?XML=true

```

上面是用 XML 格式显示服务器状态信息的命令。  

首先，显示的是服务器和 JVM 的版本号、JVM 提供者、操作系统的名称及其版本号，然后还显示了系统体系架构类型。    

其次，显示的是关于 JVM 的内存使用信息。   

最后，显示的是关于 Tomcat AJP 和 HTTP 连接器的信息。对两者来说，这些信息都很有用：   

- 线程信息：最大线程数、最少及最多的空闲线程数、当前线程数量以及当前繁忙线程。  
- 请求信息：最长及最短的处理时间、请求和错误的数量，以及接受和发送的字节数量。  
- 一张完整显示线程阶段、时间、发送字节数、接受字节数、客户端、虚拟主机及请求的表。它将列出所有现有线程。下面列出了所有可能的线程阶段：     

	- **解析及准备请求** 将对请求报头进行解析，或进行必要的准备，以便读取请求主体（如果指定了传输编码）。  
	- **服务** 线程处理请求并生成响应。该阶段中至少有一个线程（可查看服务器状态页）。  
	- **完成** 请求处理结束。所有仍在输出缓冲区中的剩余响应都被传送至客户端。如果有必要保持连接活跃，则下一个阶段是“持续活跃”阶段，否则接下来直接进入“就绪”阶段。   
	- **持续活跃** 当客户端发送另一请求时，线程能使连接对客户端保持开放。如果接收到另一请求，下一阶段就将是“解析及准备请求”阶段。如果持续活跃超时结束，仍没有接收到请求，则连接关闭，进入下一阶段“就绪”阶段。
	- **就绪** 线程空闲，等待再此被使用。  

使用 `/status/all` 命令可查看每一个已配置 Web 应用的额外信息。   

## 使用 JMX 代理 Servlet  

### 什么是 JMX 代理 Servlet   

JMX 代理 Servlet 是一款轻量级的代理。它的用途对用户来说并不是特别友好，但是其 UI 却非常有助于整合命令行脚本，从便于监控和改变 Tomcat 的内部运行。通过这个代理，我们可以获取和设置信息。要想真正了解 JMX 代理 Servlet，首先应该大概了解 JMX。如果不知道 JMX 的基本原理，那有些内容就很难理解了。   

### JMX 查询命令  

JMX 的查询命令格式如下所示：   

`http://webserver/manager/jmxproxy/?qry=STUFF`    

`STUFF` 是所要执行的 JMX 查询。比如，可以执行以下这些查询：  

- `qry=*%3Atype%3DRequestProcessor%2C* --> type=RequestProcessor` 定位所有能够处理请求并汇报各自状态的 Worker。  
- `qry=*%3Aj2eeType=Servlet%2c* --> j2eeType=Servlet` 查询返回所有加载的 Servlet。  
- `qry=Catalina%3Atype%3DEnvironment%2Cresourcetype%3DGlobal%2Cname%3DsimpleValue --> Catalina:type=Environment,resourcetype=Global,name=simpleValue` 按照指定名称查找 MBean。    

需要实际地试验一下才能真正理解这些功能。如果没有提供 `qry` 参数，则将显示全部的 MBean。我们强烈建议你去阅读 Tomcat 源代码，真正了解 JMX 规范，更好地掌握所有能够执行的查询。  

### JMX 的 `get` 命令  

JMXProxyServlet 还支持一种 `get` 命令来获取特定 MBean的属性值。该命令的一般格式如下所示：  

`http://webserver/manager/jmxproxy/?get=BEANNAME&att=MYATTRIBUTE&key=MYKEY`  

必须提供如下参数：  

1. `get`：MBean 的完整名称。  
2. `att`：希望获取的属性。  
3. `key`：（可选参数）CompositeData MBean 的属性中的键。  

如果命令成功执行，则一切正常，否则就会返回一个出错消息。举两个例子，比如当希望获取当前的堆内存数据时，可以采用如下命令：   

`http://webserver/manager/jmxproxy/?get=java.lang:type=Memory&att=HeapMemoryUsage`  

再或者，如果只希望获取“用过的”键，可以采用如下命令：   

`http://webserver/manager/jmxproxy/?get=java.lang:type=Memory&att=HeapMemoryUsage&key=used`  


### JMX 的 `set` 命令   

上面介绍了如何查询一个 MBean。下面来看看 Tomcat 的内部运行吧！`set` 命令的一般格式为：   

`http://webserver/manager/jmxproxy/?set=BEANNAME&att=MYATTRIBUTE&val=NEWVALUE`  

需要提供三个请求参数：   

- `set`：完整的 bean 名称。   
- `att`：想要改变的属性。  
- `val`：新的属性值。   

如果命令成功执行，则一切正常，否则就会返回一个出错消息。比如，假如想为 `ErrorReportValve` 进行立即调试，可以将属性 `debug` 设为 10：  

```
http://localhost:8080/manager/jmxproxy/
 ?set=Catalina%3Atype%3DValve%2Cname%3DErrorReportValve%2Chost%3Dlocalhost
 &att=debug&val=10  
 
```

所得结果如下（你的有可能不同）：   

`Result: ok`

下面来看看如果传入一个不恰当数值时的情况，比如使用一个URL，并试图将属性 debug 设置为 'cow'。  

```
http://localhost:8080/manager/jmxproxy/
 ?set=Catalina%3Atype%3DValve%2Cname%3DErrorReportValve%2Chost%3Dlocalhost
 &att=debug&val=cow

```  

运行结果如下：   

`Error: java.lang.NumberFormatException: For input string: "cow"  `


### JMX 的 `invoke` 命令    

使用 `invoke` 命令，我们就可以在 MBean 中调用方法。该命令的一般格式为：  

```
http://webserver/manager/jmxproxy/
 ?invoke=BEANNAME&op=METHODNAME&ps=COMMASEPARATEDPARAMETERS
  
```

比如，使用如下方式来调用 **Service** 的 `findConnectors()` 方法：  

```
http://localhost:8080/manager/jmxproxy/
 ?invoke=Catalina%3Atype%3DService&op=findConnectors&ps=

```




## 利用 Ant 执行 Manager 的命令    

上面的文档介绍了如何利用 HTTP 请求来执行 Manager 的命令。除此之外，Tomcat 还专为Ant（1.4 版或更新版本）构建工具准备了一套方便的任务定义。为了使用这些命令，必须执行下面这些操作：    

- 下载 Ant 二进制分发包，地址为：[http://ant.apache.org](http://ant.apache.org)。必须使用 1.4 版本或更新版本。  
- 将分发包安装到合适的目录中（下面将把它叫做 ANT_HOME）。  
- 将文件 `server/lib/catalina-ant.jar` 从 Tomcat 安装目录中复制到 Ant 的库目录（`$ANT_HOME/lib`）。  
- 将 `$ANT_HOME/bin` 目录添加到环境变量 `PATH` 中。  
- 在 Tomcat 用户数据库中，至少配置一个拥有 `manager-script` 角色的用户名/密码组合数据。   

为了在 Ant 中使用自定义任务，必须首先用 `<taskdef>` 元素来声明它们，因而 `build.xml` 文件应类似如下这样：    

```
<project name="My Application" default="compile" basedir=".">

  <!-- Configure the directory into which the web application is built -->
  <property name="build"    value="${basedir}/build"/>

  <!-- Configure the context path for this application -->
  <property name="path"     value="/myapp"/>

  <!-- Configure properties to access the Manager application -->
  <property name="url"      value="http://localhost:8080/manager/text"/>
  <property name="username" value="myusername"/>
  <property name="password" value="mypassword"/>

  <!-- Configure the custom Ant tasks for the Manager application -->
  <taskdef name="list"      classname="org.apache.catalina.ant.ListTask"/>
  <taskdef name="deploy"    classname="org.apache.catalina.ant.DeployTask"/>
  <taskdef name="start"     classname="org.apache.catalina.ant.StartTask"/>
  <taskdef name="reload"    classname="org.apache.catalina.ant.ReloadTask"/>
  <taskdef name="stop"      classname="org.apache.catalina.ant.StopTask"/>
  <taskdef name="undeploy"  classname="org.apache.catalina.ant.UndeployTask"/>
  <taskdef name="resources" classname="org.apache.catalina.ant.ResourcesTask"/>
  <typedef name="sessions"  classname="org.apache.catalina.ant.SessionsTask"/>
  <taskdef name="findleaks" classname="org.apache.catalina.ant.FindLeaksTask"/>
  <typedef name="vminfo"    classname="org.apache.catalina.ant.VminfoTask"/>
  <typedef name="threaddump" classname="org.apache.catalina.ant.ThreaddumpTask"/>
  <typedef name="sslConnectorCiphers" classname="org.apache.catalina.ant.SslConnectorCiphersTask"/>

  <!-- Executable Targets -->
  <target name="compile" description="Compile web application">
    <!-- ... construct web application in ${build} subdirectory, and
            generated a ${path}.war ... -->
  </target>

  <target name="deploy" description="Install web application"
          depends="compile">
    <deploy url="${url}" username="${username}" password="${password}"
            path="${path}" war="file:${build}${path}.war"/>
  </target>

  <target name="reload" description="Reload web application"
          depends="compile">
    <reload  url="${url}" username="${username}" password="${password}"
            path="${path}"/>
  </target>

  <target name="undeploy" description="Remove web application">
    <undeploy url="${url}" username="${username}" password="${password}"
            path="${path}"/>
  </target>

</project>
  
```  

注意：上面的资源任务定义将覆盖 Ant 1.7 中所添加的资源数据类型。如果你希望使用这些资源数据类型，需要使用 Ant 命名空间支持，将 Tomcat 的任务分配到它们自己的命名空间中。  

现在，可以执行类似 `ant deploy` 这样的命令将应用部署到 Tomcat 的一个运行实例上，或者利用 `ant reload` 通知 Tomcat 重新加载应用。另外还需注意的是，在这个 `build.xml` 文件中，多数比较有价值的属性值都是可以被可替换的，因而可以利用命令行方式来重写这些值。比如，考虑到在 `build.xml` 文件中包含真正的管理员密码是非常危险的，可以通过一些命令来忽略密码属性，如下所示：  

`ant -Dpassword=secret deploy`    

### 任务输出捕获  

使用 Ant **1.6.2** 版或更新版本，Catalina 任务提供选项，利用属性或外部文件捕获输出。它们直接支持 `<redirector>` 类型属性的子集：   



|属性|属性说明|是否必需|  
|---|---|---|  
|`output`|输出文件名。如果错误流没有重定向到一个文件或属性上，它将出现在输出中。|否|  
|`error`|命令的标准错误应该被重定向到的文件。|否|  
|`logError`|用于在 Ant 日志中显示错误输出，将输出重定向至某个文件或属性。错误输出不会包含在输出文件或属性中。如果利用 `error` 或 `errorProperty` 属性重定向错误，则没有任何效果。|否|  
|`append`|输出和错误文件是否应该附加或覆盖。默认为 `false`。|否|  
|`createemptyfiles`|是否应该创建输出和错误文件，哪怕是空的文件。默认为 `true`。|否|  
|`outputproperty`|用于保存命令输出的属性名。除非错误流被重定向至单独的文件或流，否则这一属性将包含错误输出。|否|
|`errorproperty`|用于保存命令标准错误的属性名。|否|  

还可以指定其他一些额外属性：   

|属性|属性说明|是否必需|  
|---|---|---|  
|`alwaysLog`|该属性用于查看捕获的输出，这个输出也出现在 Ant 日志中。除非捕获任务输出，否则千万不要使用它。默认为 `false`。**Ant 1.6.3 通过 `<redirector>` 直接支持该属性。**|否|  
|`failonerror`|用于避免因为 manager 命令处理中错误而导致 Ant 执行终止情况的发生。默认为 `true`。如果希望捕获错误输出，则必须设为`false`，否则 Ant 执行将有可能在未捕获任何输出前就被终止。<br/><br/>该属性只用于 manager 命令的执行上，任何错误的或丢失的命令属性仍然会导致 Ant 执行终止。|否|   


它们还支持内嵌的 `<redirector>` 元素，你可以在这些元素中指定全套的属性。但对于`input`、`inputstring`、`inputencoding`，即使接收，也无法使用，因为在这种上下文中它们没有任何意义。详情可参考 [Ant手册](http://ant.apache.org)以了解 `<redirector>` 元素的各个属性。   

下面这个范例摘录了一段构建文件，展示了这种对输出重定向的支持是如何运作的。


```
    <target name="manager.deploy"
        depends="context.status"
        if="context.notInstalled">
        <deploy url="${mgr.url}"
            username="${mgr.username}"
            password="${mgr.password}"
            path="${mgr.context.path}"
            config="${mgr.context.descriptor}"/>
    </target>

    <target name="manager.deploy.war"
        depends="context.status"
        if="context.deployable">
        <deploy url="${mgr.url}"
            username="${mgr.username}"
            password="${mgr.password}"
            update="${mgr.update}"
            path="${mgr.context.path}"
            war="${mgr.war.file}"/>
    </target>

    <target name="context.status">
        <property name="running" value="${mgr.context.path}:running"/>
        <property name="stopped" value="${mgr.context.path}:stopped"/>

        <list url="${mgr.url}"
            outputproperty="ctx.status"
            username="${mgr.username}"
            password="${mgr.password}">
        </list>

        <condition property="context.running">
            <contains string="${ctx.status}" substring="${running}"/>
        </condition>
        <condition property="context.stopped">
            <contains string="${ctx.status}" substring="${stopped}"/>
        </condition>
        <condition property="context.notInstalled">
            <and>
                <isfalse value="${context.running}"/>
                <isfalse value="${context.stopped}"/>
            </and>
        </condition>
        <condition property="context.deployable">
            <or>
                <istrue value="${context.notInstalled}"/>
                <and>
                    <istrue value="${context.running}"/>
                    <istrue value="${mgr.update}"/>
                </and>
                <and>
                    <istrue value="${context.stopped}"/>
                    <istrue value="${mgr.update}"/>
                </and>
            </or>
        </condition>
        <condition property="context.undeployable">
            <or>
                <istrue value="${context.running}"/>
                <istrue value="${context.stopped}"/>
            </or>
        </condition>
    </target>

```


> **警告**：多次调用 Catalina 任务往往并不是一个好主意，退一步说这样做的意义也不是很大。如果 Ant 任务依赖链设定糟糕的话，即使本意并非如此，也会导致在一次 Ant 运行中多次运行任务。必须提前对你稍加警告，因为有可能当你从任务中捕获输出时，会出现一些意想不到的情况：  
> 
> - 当用属性捕获时，你将只能从其中找到**最初**调用的输出，因为 Ant 属性是不变的，一旦设定就无法改变。  
> - 当用文件捕获时，你将只能从其中找到**最后**调用的输出，除非使用 `append = "true"` 属性——在这种情况下，你将看到附加在文件内容末尾的每一个任务调用的相关输出。








  



















