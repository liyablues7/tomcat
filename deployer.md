# Tomcat Web 应用部署

## 本章目录   

- 本章概述 
- 安装  
- 关于上下文 
- Tomcat 启动部署  
- 在运行中的 Tomcat 服务器上进行部署  
- 使用 Tomcat Manager 进行部署  
- 使用客户端部署器进行部署     

## 本章概述  

**部署**这个术语描述的就是，将 Web 应用（第三方的 WAR 文件，或是你自己定制的 Web 应用）安装到 Tomcat 服务器上的整个过程。  

在 Tomcat 服务器上，可以通过多种方法部署 Web 应用：  

- 静态部署。在启动 Tomcat 之前安装 Web 应用。  
- 动态部署。使用 Tomcat 的 Manager 应用直接操控已经部署好的 Web 应用（依赖 auto-deployment 特性）。    

Tomcat Manager 是一种能交互使用（利用 HTML GUI）的 Web 应用，还可以利用编程的方式（通过基于 URL 的 API）来部署并管理 Web 应用。    

依靠 Manager 这种 Web 应用，可以实施多种部署。Tomcat 为 Apache Ant 构建工具提供了多个任务。 [Apache Tomcat Maven Plugin](http://tomcat.apache.org/maven-plugin.html) 工程则提供了与 Apache Maven 的集成。另外还有一种工具叫做**客户端配置器** （Client Deployer，TCD），它通过命令行来使用，提供一些额外的功能，比如编译与验证 Web 应用，以及将 Web 应用打包成 Web 应用资源（WAR）文件。   

## 安装   

静态部署 Web 应用时，并不需要附加的安装，因为 Tomcat 已经提供了这项功能。利用 Tomcat Manager 部署应用也不需要任何安装，不过需要进行一番配置，详见[Tomcat Manager 手册](》链接中文的5-Manager页面》)。如果使用客户端配置器的话，就必须要进行安装了。     

Tomcat 的核心分发版并不包括 TCD，必须从下载区独立下载它，下载文件通常冠名为：*apache-tomcat-8.0.x-deployer*。  

要想使用 TCD，必须事先配置有 Apache Ant 1.6.2+ 以及 Java 安装。另外，还必须定义一个指向 ANT 安装根目录的 ANT_HOME 环境变量，以及一个指向 Java 安装目录的 JAVA_HOME 值。另外，还必须确保必须在操作系统所提供的命令 shell 中运行 ANT 的 `ant` 命令，以及 Java 的 `javac` 编译器命令。  

1. 下载 TCD 分发版； 
2. TCD 包可以解压缩到任何地方，不需要解压缩到任何已存在的 Tomcat 安装下。  
3. 相关用法，参考下文的<a href = "#TomcatClientDeployer">使用客户端部署器进行部署</a>  

## 关于上下文     

在谈到 Web 应用的配置时，需要理解一下**上下文**（Context）这个概念。上下文在 Tomcat 中其实就是 Web 应用的意思。   

为了在 Tomcat 中配置上下文，需要用到**上下文描述符文件**（ Context Descriptor）。上下文描述符文件其实就是一个 XML 文件，含有 Tomcat 与上下文相关的配置信息，例如命名资源或会话管理器配置信息。在 Tomcat 的早期版本中，上下文描述符文件配置的内容经常保存在 Tomcat 的主要配置文件 server.xml 中，但现在不再推荐采用这一方式（虽然目前它依然有效）。   

上下文描述符文件不仅能帮助 Tomcat 了解如何配置上下文，而且其他工具（如 Manager 与 TCD）也经常会借助上下文描述符文件来正确履行它们的职责。       

上下文描述符文件位于：   

1. $CATALINA_BASE/conf/[enginename]/[hostname]/[webappname].xml   
2. $CATALINA_BASE/webapps/[webappname]/META-INF/context.xml

在目录 1 中的文件名为 [webappname].xml，但在目录2中，文件名为context.xml。如果某个 Web 应用没有相应的上下文描述符文件，Tomcat 就会使用默认值配置该应用。     


## 在 Tomcat 启动时进行部署   

如果你对使用 Manager 或 TCD 不是很感兴趣，那就需要先把 Web 应用静态地部署到 Tomcat 中，然后再启动 Tomcat。 这种情况下应用部署的位置由 `appBase` 目录属性来决定，每台主机都指定有这样一个位置。该位置既可以放入未经压缩的 Web 应用资源文件（通常被称为exploded web application，“膨胀 Web 应用”），也可以放置已压缩过的 Web 应用资源文件（.WAR 文件）。    

再次解释一下，应用部署位置由主机（默认主机为 localhost）的 `appBase` 属性来指定。默认的 `appBase` 属性所指定的目录为  $CATALINA_BASE/webapps。只有当主机的 `deployOnStartup` 属性为 `true`, 应用才会在 Tomcat 启动时进行自动部署。    

在以上情况下，当 Tomcat 启动时，部署的具体步骤如下：   

1. 先部署上下文描述符文件。   
2. 然后再对没被任何上下文描述符文件引用过的膨胀 Web 应用进行部署。 如果在 `appBase` 中已存在与这种应用有关的 .WAR 文件，而且要比膨胀应用文件更新，那么就会将膨胀应用的文件夹清除，转而从 .WAR 文件中部署 Web应用。 
3. 部署 .WAR 文件。  


## 在运行中的 Tomcat 服务器上进行动态应用部署  

除了静态部署之外，也可以在运行中的 Tomcat 服务器上进行应用部署。   

如果主机的 `autoDeploy` 属性为 `true`，主机就会在必要时尝试着动态部署并更新 Web 应用。 例如，当把一个新 .WAR 文件放入 `appBase` 所指定的名录时。为了实现这种操作，主机就需要启用后台处理，当然这也是默认的配置。   

当 `autoDeploy` 设置为 `true` 时，运行中的 Tomcat 服务器能够允许实现以下行为：     

- 对放入主机 `appBase`指定目录下的 .WAR 文件进行配置。
- 对放入主机的膨胀 Web 应用进行配置。  
- 对于已通过 .WAR 文件配置好的应用，如果又提供了更新的 .WAR 文件，则使用新 .WAR 文件对该应用重新进行配置。在这种情况下，会先移除原有的膨胀 Web 应用，然后再次对 .WAR 文件进行扩展（膨胀）。注意，如果在主机配置中，没有把 `unpackWARs` 属性设为 `false`，则 WAR 文件将不会膨胀，这时 Web 应用将部署为一个压缩文档。
- 如果 /WEB-INF/web.xml 文件（或者任何其他被定义为 WatchedResource 的资源）更新，则重新加载 Web 应用。  
- 如果用来部署 Web 应用的上下文描述符更新，则重新部署该 Web 应用。  
- 如果 Web 应用所使用的全局或者每台主机中的上下文描述符已更新，则重新部署与该应用有依赖关系的 Web 应用。
- 如果一个上下文描述符被添加到 `$CATALINA_BASE/conf/[enginename]/[hostname]/` 目录中，并且该描述文件带有与之前部署的 Web 应用的上下文路径相对应的文件名，则重新部署该 Web 应用。  
- 如果某个 Web 应用的文档库（docBase）被删除，则取消对该应用的部署。注意，在 Windows 系统下，要想实现这样的行为，必须开启防锁死功能（参看 [Context 配置](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html)文档），否则无法删除运行中的 Web 应用的资源文件。    


注意，也可以在加载器中对 Web 应用的重新加载进行配置，在这种情况下，会跟踪已加载的类所产生的更改。  


## 使用 Tomcat Manager 进行部署  

详情参看 [Tomcat Manager 文档](http://tomcat.apache.org/tomcat-8.0-doc/manager-howto.html》》)。   


## <a name = "TomcatClientDeployer">使用客户端部署器进行部署</a>   

最后要介绍的是利用客户端部署器（TCD）对 Web 应用进行部署。客户端部署器可以实施的行为包括：验证并编译 Web 应用，将资源文件压缩成 .WAR 文件，并将 Web 应用部署到用于生产或开发环境的 Tomcat 服务器上。一定要注意，该特性的实现需要使用 Tomcat Manager，而且目标 Tomcat 服务器也应处于运行状态。  

因为会用到 TCD，所以要求用户还必须熟悉 Apache Ant。Apache Ant 是一个脚本编译工具。TCD 每个包都会带有一个编译脚本。只需大体能够了解 Apache Ant 即可（本节前面列有其安装细则，这里需要熟练使用操作系统命令 shell 以及配置环境变量）。   

TCD 包括一些 Ant 任务，在配置前用于 JSP 编译的 Jasper 页面编译器，以及验证 Web 应用上下文描述符的任务。验证器任务（`org.apache.catalina.ant.ValidatorTask`类）只允许传入一个参数：膨胀 Web 应用的基本路径。  

TCD 使用膨胀 Web 应用作为输入（下面列出了其所用的属性列表）。通过部署器，以编程方式部署的 Web 应用可能会在 /META-INF/context.xml 中包含一个上下文描述符。   

TCD 包含一个可即时使用的 Ant 脚本， 其中包含以下目标。

- `compile`（默认）：编译并验证 Web 应用。可以单独使用，并不需要运行着的 Tomcat 服务器。已编译的应用只能运行在相关的 Tomcat X.Y.Z 版本的服务器上，又因为 Jasper 生成的代码依赖它的运行时组件，所以已编译应用并不一定能在其他版本的 Tomcat 版本上运行。另外值得注意的是，该目标也能自动编译位于 /WEB-INF/classes 这一应用目录下的任何 Java 源文件。  
- `deploy` 在 Tomcat 服务器上部署 Web 应用（无论其是否编译过）。   
- `undeploy` 取消对某个 Web 应用的部署。  
- `start` 开启 Web 应用。  
- `reload` 重新加载 Web 应用。  
- `stop` 停止 Web 应用。   


为了能够配置部署，还需要在 TCD 安装的根目录下创建一个叫做 `deployer.properties` 的文件，并在该文件中的每行添加下列名值对：   

除此之外，你还必须确定为 TCD 所使用的目标 Tomcat Manager 创建了一个用户，否则 TCD 就无法验证 Tomcat Manager，从而造成配置失败，详细信息参看 [Tomcat Manager 文档](》》)。

- **`build`** build 文件夹默认位置是 `${build}/webapp/${path}` （其中 `${build}` 的默认指向位置是 `${basedir}/build）`。`compile` 目标执行完毕后，Web 应用的 .WAR 文件将位于 `${build}/webapp/${path}.war`。   
- **`webapp`** 该文件夹包含后续将进行编译与验证的膨胀 Web 应用。默认情况下，该文件夹是 `myapp`。   
- **`path`** Web 应用已部署的上下文路径，默认为 `/myapp`。   
- **`url`** 指向运行中的 Tomcat 服务器中的某个 Tomcat Manager Web 应用的绝对路径，用于对 Web 应用的部署与取消部署。默认情况下，部署器会尝试访问运行在 localhost 上的 Tomcat 实例，其url为 `http://localhost:8080/manager/text`。   
- **`username`** Tomcat Manager 的用户名（用户应具有读写manager-script的权限）。  
- **`password`** Tomcat Manager 的密码。   
















