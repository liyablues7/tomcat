## 目录  

## 什么是DefaultSevelet    

## 在什么位置声明它？  

它在 `$CATALINA_HOME/conf/web.xml` 中被全局声明。默认形式的声明是这样的： $CATALINA_HOME/conf/web.xml

```  

```   

因此在默认的情况下，默认servlet在Web 应用启动时被装载，目录列表可被使用，日志调试功能被关掉。  

## what can i change?  

DefaultServlet 允许设置以下初始化参数：  

|属性|描述|  
|---|---|  
|`debug`|调试级别。如果不是 tomcat 开发人员，则没有什么太大的用处。截止本文写作时，有用的值是 0、1、11、1000。默认值为 0|  
|`listings`|如果没有欢迎文件，要不要显示目录列表？值可以是true 或 false。<br/>欢迎文件是servlet api的一部分。<br/>**警告**：目录列表中含有的很多项目都是非常消耗服务性能的，如果对大型目录列表多次进行请求，会严重消耗服务器资源。|    
|`gzip`|如果某个文件存在gzip格式的文件（带有`gz`后缀名的文件通常就在原始文件旁边）。如果用户代理支持 gzip 格式，并且启用了该选项，Tomcat 就会提供该格式文件的服务。默认为 false。<br/>如果直接请求带有 `gz` 后缀名的文件，是可以访问它们的，所以如果原始资源受安全挟制的保护，则 gzip 文件也同样是受保护的。|  
|`readmeFile`|如果提供了目录列表，那么可能也会提供随带的 readme 文件。这个文件是被插入的，因此可能会包含 HTML。默认值是null。|    
|`globalXsltFile`|如果你希望自定义目录列表，你可以使用一个XSL转换（transformation）。这个值是一个可用于所有目录列表的相对路径文件名（既相对于 `CATALINA_BASE/conf/` 也相对于 `$CATALINA_HOME/conf/`）。》这可以每个上下文或每一目录》可参看下面介绍的 `contextXsltFile` 和 `localXsltFile`。该 xml 文件的格式会在下文介绍。|   
|`contextXsltFile`|你可以通过`contextXsltFile` 来自定义你的目录列表。这必须是一个上下文相对路径（例如：`/path/to/context.xslt`），相对于带有 `.xsl` 或 `.xslt` 扩展名的文件。它将覆盖 `globalXsltFile`。如果提供了该值，但相对文件却不存在，则将使用 `globalXsltFile`。如果 `globalXsltFile` 也不存在，则显示默认的目录列表。|
|`localXsltFile`|你还可以在每个目录通过配置 `localXsltFile` 定制你的目录列表。它应该是在产生列表的目录里的一个相对路径文件名。它覆盖 `globalXsltFile` 和 `contextXsltFile`。如果该值存在，但是文件不存在，那么就使用 `contextXsltFile`。如果`contextXsltFile` 也不存在，那么就会使用 `globalXsltFile`。如果 `globalXsltFile` 也不存在，那么默认的目录列表就会被显示出来。|
|`input`|在读取用于服务的资源时的输入缓冲大小（以字节计）。默认为 2048。|
|`output`|在修改用于服务的资源时的输出缓冲大小（以字节计）。默认为 2048。|
|`readonly`|上下文是否为“只读”，从而拒绝执行 PUT 或 DELETE 这样的 HTTP 命令。默认为 true。|
|`fileEncoding`|文件编码用于读取静态资源时。默认取平台默认值。|
|`sendfileSize`|如果所用的连接器支持 sendfile，这个参数表示所用的 sendfile 最小的文件大小（以 KB 计）。使用负数往往表示可以禁止使用 sendfile。默认为 48。|
|`useAcceptRanges`|如果为 true，则将设置 Accept-Ranges 报头，在适于响应时。|
|`showServerInfo`|当使用》》》目录列表，服务器信息是否应该提供给发往客户端的响应中。默认为 true。|   


## 我该如何自定义目录列表   

你可以用自定义实现来覆盖 DefaultServlet，并将它用在 web.xml 声明中。如果你能明白刚才所说的是什么意思，我们就认为你能读懂 DefaultServlet 的代码并作出适当的调整。（如果不能明白，则说明这种方式不适合你。）      

`localXsltFile` 或 `globalXsltFile `   

格式如下：   

```  

```

- 如果 `type = 'dir'`，则》》  
- Readme 是一个 CDATA 项。   

下面是一个能够模仿 Tomcat 默认行为的范例 xsl 文件：   

```   

```   

## 如何保证目录列表的安全性  

在每一个单独的 Web 应用中使用 web.xml。可查看 Servlet 规范的安全性部分的内容。   











