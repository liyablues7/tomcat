# Jasper 2 JSP 引擎 

## 目录 

- <a href = "#Introduction">简介</a>  
- <a href = "#Configuration">配置</a>  
- <a href = "#Knownissues">已知问题</a>  
- <a href = "#ProductionConfiguration">产品配置</a>  
- <a href = "#WebApplicationCompilation">Web 应用编译</a>  
- <a href = "#Optimisation">优化</a>  


## <a name = "Introduction">简介</a>  

Tomcat 8.0 使用 Jasper 2 JSP 引擎去实现 [JavaServer Pages 2.3 规范](http://wiki.apache.org/tomcat/Specifications)。    

Jasper 2 经过了重新设计，极大改善了上一版 Jasper 的性能。除了一般性的代码改进之外，还做出了以下改变：   

- **JSP 自定义标签池化** 针对 JSP 自定义标签（JSP Custom Tags）实例化的 Java 对象现已可以被池化和重用，从而极大提高了使用自定义标签的 JSP 页面的性能。   
- **JSP 后台编译** 如果你更改了一个已经编译的 JSP 页面，Jasper 2 会在后台重新编译该页面。之前编译的 JSP 页面将依然能够服务请求。一旦新页面编译成功，它就会自动代替旧页面。这能提高生产服务器上的 JSP 页面的可用性。   
- **能够重新编译发生改动的 JSP 页面** Jasper 2 现在能够侦测页面何时出现改动，然后重新编译父 JSP。  
- **用于编译 JSP 页面的 JDT 编译器** Eclipse JDT Java 编译器现在能用来编译 JSP java 源代码。该编译器从容器类加载器加载源代码支持。Ant 与 javac 依旧能够使用。   

Jasper 可以使用 servlet 类 `org.apache.jasper.servlet.JspServlet`。    


## <a name = "Configuration">配置</a>    

Jasper 默认就是用于开发 Web 应用的。关于如何在 Tomcat 生产服务器中配置并使用 Jasper，可参考[生产环境配置](http://tomcat.apache.org/tomcat-8.0-doc/jasper-howto.html#Production_Configuration)一节内容。  

在全局性的 `$CATALINA_BASE/conf/web.xml` 中使用如下初始参数，来配置实现 Jasper 的 servlet。   

- **checkInterval** 如果 `development` 为 false，并且`checkInterval` 大于 0，则开启后台编译。`checkInterval` 参数的含义就是在检查某个 JSP 页面（以及从属文件）是否需要重新编译时，几次检查的间隔时间（以秒计）。默认为 0 秒。   

- **classdebuginfo** 是否应在编译类文件时带上调试信息？布尔值，默认为 `true`。  

- **classpath** 对生成的 servlet 进行编译时将要使用的类路径。如果 `ServletContext` 属性 `org.apache.jasper.Constants.SERVLET_CLASSPATH` 没有设置。当在 Tomcat 中使用 Jasper 时，该属性经常设置。默认情况下，根据当前 Web 应用，动态创建类路径。    

- **compiler** 应用何种编译器 Ant 编译 JSP 页面。该参数的有效值与 Ant 的 [javac](http://ant.apache.org/manual/Tasks/javac.html#compilervalues) 任务的编译器属性值完全相同。如果没有设置该值，则采用默认的 Eclipse JDT Java 编译器，而不是 Ant。该参数没有默认值，如果该属性被设置，则就应该使用 `setenv.[sh|bat]` 将 `ant.jar`、`ant-launcher.jar` 与 `tools.jar` 添加到 `CLASSPATH` 环境变量中。   

- **compilerSourceVM** 与源文件所兼容的 JDK 版本？（默认值：`1.7`）  

- **compilerTargetVM** 与生成文件所兼容的 JDK 版本？（默认值：`1.7`）    

- **development** Jasper 是否被用于开发模式？如果为 true，可能通过 `modificationTestInterval` 参数来指定检查 JSP 更改情况的频率。布尔值，默认为 `true`。    

- **displaySourceFragment** 异常信息是否应包含源代码片段？布尔值，默认为 `true`。  

- **dumpSmap** JSR45 调试的 SMAP 信息是否应转储到一个文件？布尔值，默认为 `false`。如果 `suppressSmap` 为 `true`，则该参数值为 `false`。  

- **enablePooling** 是否启用标签处理池（tag handler pooling）？这是一个编译选项。它不会影响已编译的 JSP 行为。布尔值，默认为 `true`。   

- **engineOptionsClass** 允许指定用来配置 Jasper 的 Options 类。如果不存在，则使用默认的 EmbeddedServletOptions。   

- **errorOnUseBeanInvalidClassAttribute** 当 useBean 行为中的类属性值不是合法的 bean 类时，Jasper 是否弹出一个错误？布尔值，默认为 `true`。   

- **fork**	 Ant 是否应该分叉（fork）编译 JSP 页面，以便在单独的 JVM 中执行编译？布尔值，默认为 `true`。   

- **genStringAsCharArray** 为了在一些情况下提高性能，是否应将文本字符串生成字符数组？默认为 `false`。   

- **ieClassId** 使用 <jsp:plugin> 标记时，被送入 IE 浏览器中的类 ID 值。默认值为：`clsid:8AD9C840-044E-11D1-B3E9-00805F499D93`。    

- **javaEncoding** 用于生成 Java 源文件的 Java 文件编码。默认为 `UTF-8`。  

- **keepgenerated** 对于每个页面所生成的 Java 源代码，应该保留还是删除？布尔值，默认为 `true`（保留）。   

- **mappedfile** 为便于调试，是否应该生成静态内容，每行输入都带有一个打印语句？布尔值，默认为 `true`。  

- **maxLoadedJsps** Web 应用所能加载的 JSP 的最大数量。如果超出此数目，就卸载最近最少使用的 JSP，以防止任何时刻加载的 JSP 数目不超过此限。0 或负值代表没有限制。默认为 -1。   

- **jspIdleTimeout** JSP 在被卸载前，处于空闲状态的时间（以秒计）。0 或负值代表永远不会卸载。默认为 `-1`。  

- **modificationTestInterval** 自上次检查 JSP 修改起，造成 JSP（以及从属文件）没有被检查修改的指定时间间隔（以秒计）。取值为 0 时，每次访问都会检查 JSP 修改。只用于开发模式下。默认为 `4` 秒。   

- **recompileOnFail**  如果 JSP 编译失败，是否应该忽略 `modificationTestInterval`，下一次访问是否触发重新编译的尝试？只用在开发模式下，并且默认是禁止的，因为编译会使用大量的资源，是极其昂贵的过程。   

- **scratchdir** 编译 JSP 页面时应该使用的临时目录（scratch directory）。默认为当前 Web 应用的工作目录。   

- **suppressSmap** 是否禁止 JSR45 调试时生成的 SMAP 信息？`true` 或 `false`，缺省为 `false`。

- **trimSpaces** 是否应清除模板文本中行为与指令之间的的空格？默认为 `false`。  

- **xpoweredBy**  是否通过生成的 servlet 添加 X-Powered-By 响应头？布尔值，默认为 `false`。   


Eclipse JDT 的 Java 编译器被指定为默认的编译器。它非常先进，能够从 Tomcat 类加载器中加载所有的依赖关系。这将非常有助于编译带有几十个 JAR 文件的大型安装。在较快的服务器上，还可能实现以次秒级周期对大型 JSP 页面进行重新编译。  

通过配置上述编译器属性，之前版本 Tomcat 所用的 Apache Ant 可以替代新的编译器。  



## <a name = "Knownissues">已知问题</a>      

[bug 39089]()报告指出，在编译非常大的 JSP 时，已知的 JVM 问题 [bug 6294277]() 可能会导致出现 `java.lang.InternalError: name is too long to represent` 异常。如果出现这一问题，可以采用下列办法来解决：   

- 减少 JSP 大小。  
- 将 `suppressSmap` 设为 `true`，禁止生成 SMAP 信息与 JSR-045 支持。   


## <a name = "ProductionConfiguration">生产配置</a>    

能做的最重要的 JSP 优化就是对 JSP 进行预编译。但这通常不太可能（比如说，使用jsp-property-group 功能时）或者说不太实际，这种情况下，如何配置Jasper Servlet 就变得很关键了。  

在生产级 Tomcat 服务器上使用 Jasper 2 时，应该考虑将默认配置进行如下这番修改：   

- **development** 针对 JSP 页面编译，禁用访问检查，可以将其设为 `false`。 
- **genStringAsCharArray** 设定为 `true` 可以生成稍微更有效率的字符串数组。
- **modificationTestInterval**  如果由于某种原因（如动态生成 JSP 页面），必须将 `development` 设为 `true`，提高该值将大幅改善性能。
- **trimSpaces** 设为 `true` 可以去除响应中的无用字节。  



## <a name = "WebApplicationCompilation">Web 应用编译</a>    

使用 Ant 是利用 JSPC 编译 Web 应用的首选方式。注意在预编译 JSP 页面时，如果 `suppressSmap` 为 `false`，而 `compile` 为 true，则 SMAP 信息只能包含在最后的类中。使用下面的脚本来预编译 Web 应用（在 deployer 下载中也包含类似的脚本）。



```
<project name="Webapp Precompilation" default="all" basedir=".">

   <import file="${tomcat.home}/bin/catalina-tasks.xml"/>

   <target name="jspc">

    <jasper
             validateXml="false"
             uriroot="${webapp.path}"
             webXmlFragment="${webapp.path}/WEB-INF/generated_web.xml"
             outputDir="${webapp.path}/WEB-INF/src" />

  </target>

  <target name="compile">

    <mkdir dir="${webapp.path}/WEB-INF/classes"/>
    <mkdir dir="${webapp.path}/WEB-INF/lib"/>

    <javac destdir="${webapp.path}/WEB-INF/classes"
           optimize="off"
           debug="on" failonerror="false"
           srcdir="${webapp.path}/WEB-INF/src"
           excludes="**/*.smap">
      <classpath>
        <pathelement location="${webapp.path}/WEB-INF/classes"/>
        <fileset dir="${webapp.path}/WEB-INF/lib">
          <include name="*.jar"/>
        </fileset>
        <pathelement location="${tomcat.home}/lib"/>
        <fileset dir="${tomcat.home}/lib">
          <include name="*.jar"/>
        </fileset>
        <fileset dir="${tomcat.home}/bin">
          <include name="*.jar"/>
        </fileset>
      </classpath>
      <include name="**" />
      <exclude name="tags/**" />
    </javac>

  </target>

  <target name="all" depends="jspc,compile">
  </target>

  <target name="cleanup">
    <delete>
        <fileset dir="${webapp.path}/WEB-INF/src"/>
        <fileset dir="${webapp.path}/WEB-INF/classes/org/apache/jsp"/>
    </delete>
  </target>

</project>

```  


下面的代码可以用来运行该脚本（利用 Tomcat 基本路径与指向应被预编译 Web 应用的路径来取代令牌）  

`$ANT_HOME/bin/ant -Dtomcat.home=<$TOMCAT_HOME> -Dwebapp.path=<$WEBAPP_PATH>`  

然后，必须在 Web 应用部署描述符文件中添加预编译过程中生成的 servlet 的声明与映射。将 `${webapp.path}/WEB-INF/generated_web.xml` 插入 `${webapp.path}/WEB-INF/web.xml` 文件中合适的位置。使用 Manager 重启 Web 应用，测试应用，以便验证应用能正常使用预编译 servlet。利用Web 应用部署描述符文件中的一个适当的令牌，也能使用 Ant 过滤功能自动插入生成的 servlet 声明与映射。这实际上就是 Tomcat 所分配的所有 Web 应用能作为构建进程中的一部分而自动编译的原理。    

在 Jasper 任务中，还可以使用选项 `addWebXmlMappings`，它可以将 `${webapp.path}/WEB-INF/web.xml` 中的当前 Web 应用部署描述符文件自动与 `${webapp.path}/WEB-INF/generated_web.xml` 进行合并。当你想在 JSP 页面中使用 Java 6 功能时，添加下列 javac 编译器任务属性：`source="1.6" target="1.6"`。对于动态应用而言，还可以使用 `optimize="on"` 进行编译，注意，不用带调试信息：`debug="off"`。   

当首次出现 jsp 语法错误时，假如你不想停止 jsp 生成，可以使用 `failOnError="false"` 和 `showSuccess="true"`，将所有成功生成的 *jsp to java* 打印出来。这种做法有时非常有用，比如当你想要在 `${webapp.path}/WEB-INF/src` 中清除生成的 java 源文件以及 `${webapp.path}/WEB-INF/classes/org/apache/jsp` 中的编译 jsp 的 servlet 类时。  

**提示**：  

- 当你换用另一版本的 Tomcat 时，需要重新生成和编译 JSP 页面。  
- 在服务器运行时使用 Java 系统属性，通过设定 `org.apache.jasper.runtime.JspFactoryImpl.USE_POOL=false` 禁用 PageContext 池化，利用 `org.apache.jasper.runtime.BodyContentImpl.LIMIT_BUFFER=true` 限制缓存。注意，改变默认值可能会影响性能，但这种情况跟具体的应用有关。  

## <a name = "Optimisation">优化</a>      

Jasper 还提供了很多扩展点，能让用户针对具体的环境而优化行为。  

标签插件机制就是首先要谈到的一个扩展点。对于提供给 Web 应用使用的标签处理器而言，它能提供多种替代实现。标签插件 通过位于 `WEB-INF` 的 `tagPlugins.xml` 进行注册。Jasper 本身还包含了一个 JSTL 的范例插件。  

表达式语言（EL，Expression Language）解释器则是另外一个扩展点。通过 `ServletContext` 可以配置替代的 EL 解释器。可以参看 ELInterpreterFactory Java 文档来了解如何配置替代的 EL 解释器。   



