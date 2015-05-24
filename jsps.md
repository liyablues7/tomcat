# Jasper 2 JSP Engine 

## 目录 

-  
## 简介  

Tomcat 8.0 使用 Jasper 2 JSP 引擎去实现 [JavaServer Pages 2.3](http://wiki.apache.org/tomcat/Specifications) 规范。    

Jasper 2 经过了重新设计，极大改善了上一版 Jasper 的性能。除了》》，还做出了以下改变：   

- **JSP Custom Tag Pooling** JSP 自定义标签（JSP Custom Tags）》。这极大提高了使用自定义标签的 JSP 页面的性能。   
- **Background JSP compilation** 如果你更改了一个已经编译的 JSP 页面，Jasper 2 会在后台重新编译该页面。之前编译的 JSP 页面将依然能够服务请求。一旦新页面编译成功，它就会自动代替旧页面。这能提高生产服务器上的 JSP 页面的可用性。   
- **Recompile JSP when included page changes** Jasper 2 现在能够侦测页面何时出现改动，然后重新编译父 JSP。  
- **JDT used to compile JSP pages** Eclipse JDT Java 编译器现在能用来编译 JSP java 源代码。该编译器从容器类加载器加载源代码支持。Ant 与 javac 依旧能够使用。   

Jasper 可以使用 servlet 类 `org.apache.jasper.servlet.JspServlet`。    


## 配置    

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

- **scratchdir** 编译 JSP 页面时应该使用哪个 scratch 目录？》》默认为当前 Web 应用的工作目录。   

- **suppressSmap** 用于 JSR45 调试的 SMAP 信息是否应该被抑制？》》

- **trimSpaces** 是否应清除 actions 与 directives》之间的模板文本中的空格？默认为 `false`。  

- **xpoweredBy**  是否通过生成的 servlet 添加 X-Powered-By 响应头？布尔值，默认为 `false`。   


Eclipse JDT 的 Java 编译器被指定为默认的编译器。它非常先进，能够从 Tomcat 类加载器中加载所有的依赖关系。这将非常有助于编译带有几十个 JAR 文件的大型安装。在较快的服务器上，还可能实现以次秒级周期对大型 JSP 页面进行重新编译。  

通过配置上述编译器属性，之前版本 Tomcat 所用的 Apache Ant 可以替代新的编译器。  



## 已知问题    

[bug 39089]()报告指出，在编译非常大的 JSP 时，已知的 JVM 问题 [bug 6294277]() 可能会导致出现 `java.lang.InternalError: name is too long to represent` 异常。如果出现这一问题，可以采用下列办法来解决：   

- 减少 JSP 大小。  
- 将 `suppressSmap` 设为 `true`，禁用 SMAP generation 》》与 JSR-045 支持。   


## 生产配置    

能做的最重要的 JSP 优化就是对 JSP 进行预编译。但这通常不太可能（比如说，使用jsp-property-group 功能时）或者说不太实际，这种情况下，如何配置Jasper Servlet 就变得很关键了。  

在生产级 Tomcat 服务器上使用 Jasper 2 时，应该考虑将默认配置进行如下这番修改：   

- **development**   
- **genStringAsCharArray**  
- **modificationTestInterval**  
- **trimSpaces**  



## Web 应用编译    





```

```



## 优化  