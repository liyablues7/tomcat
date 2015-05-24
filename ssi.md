## 目录  

- 简介  
- 安装
- Servlet 配置
- 过滤器配置
- 指令
- 变量   


## 简介   

SSI（服务器端嵌入）是一组放在 HTML 页面中的指令，当服务器向客户端访问提供这些页面时，会解释执行这些指令。它们能为已有的 HTML 页面添加动态生成内容，不需要通过 CGI 程序来或其他的动态技术来重新改变整个页面。    

如果利用 Tomcat 作为 HTTP 服务器并需要 SSI 支持时，可以添加 SSI 支持。通常，如果你运行的不是像 Apache 那样的服务器，就通过开发来实现这种支持。  

Tomcat SSI 支持实现了与 Apache 完全一致的 SSI 指令。关于使用 SSI 指令的详细信息，可参考[Apache 的 SSI 简介](http://httpd.apache.org/docs/current/howto/ssi.html)。    

SSI 支持可以有两种方式来实现：servlet或过滤器。你只能利用其中的一种方式来提供 SSI 支持。   

基于 servlet 的 SSI 支持是通过 `org.apache.catalina.ssi.SSIServlet` 类来实现的。一般来说，这个 servlet 映射至 URL 模式"\*.shtml"。  

基于过滤器的 SSI 支持则利用 `org.apache.catalina.ssi.SSIFilter` 类来实现。一般而言，该过滤器映射至 URL 模式 "\*.shtml"，但是它也可以被映射至 "\*"，因为它会基于 MIME 类型选择性地启用/禁用对 SSI 的处理。初始参数 `contentType` 允许你将 SSI 处理应用于 JSP 页面、JavaScript 内容以及其他内容中。 

默认 Tomcat 是不支持 SSI 的。  


## 安装  

**警告**：SSI 指令可用于执行 Tomcat JVM 之外的程序。如果使用 Java SecurityManager，它会绕过你在 `catalina.policy` 中配置的安全策略。     

为了使用 SSI servlet，要从 `$CATALINA_BASE/conf/web.xml` 中去除 SSI servlet 及 servlet 映射配置旁边的 XML 注释。  

为了使用 SSI 过滤器，要从 `$CATALINA_BASE/conf/web.xml` 中去除 SSI 过滤器及过滤器映射配置旁边的 XML 注释。  

只有标明为 privileged 的上下文才可以使用 SSI 功能（参看 Context 元素的 privileged 属性）。  

## Servlet 配置  

以下这些 servlet 初始化参数可以配置 SSI servlet 的行为：  

- **buffered** 是否应缓存该 servlet 的输出？（0 = false，1 = true）默认为 0（false）。  
- **debug** 调试 servlet 所记录信息的调试细节度。默认为 0。
- **expires** 包含 SSI 指令的页面失效的秒数。默认行为针对的是每个请求所应执行的所有 SSI 指令。
- **isVirtualWebappRelative** “虚拟”的 SSI 指令路径是否应被解释为相对于上下文根目录的相对路径（而不是服务器根目录）？默认为 false。	
- **inputEncoding** 如果无法从资源本身确定编码，则应指定给 SSI 资源的编码。默认为系统默认编码。
- **outputEncoding** 用于 SSI 处理结果的编码。默认为 UTF-8。 
- **allowExec** 是否启用 `exec` 命令？默认为 false。

## 过滤器配置  

以下这些过滤器初始化参数可以配置 SSI 过滤器的行为：   

- **contentType** 在应用 SSI 处理之前必须匹配的正则表达式模式。在设计自己的模式时，不要忘记 MIME 内容类型后面可能会带着可选的字符集：“mime/type; charset=set”。默认为 “text/x-server-parsed-html(;.*)?”。
- **debug** 调试 servlet 所记录信息的调试细节度。默认为 0。 
- **expires** 包含 SSI 指令的页面失效的秒数。默认行为针对的是每个请求所应执行的所有 SSI 指令。
- **isVirtualWebappRelative** “虚拟”的 SSI 指令路径是否应被解释为相对于上下文根目录的相对路径（而不是服务器根目录）？默认为 false。
- **allowExec** 是否启用 `exec` 命令？默认为 false。

## 指令  

指令采取 HTML 注释的形式。在将页面发送到客户端之前，解读指令，并用所得结果来替换指令。指令的一般形式为：  

`<!--#directive [parm=value] -->`  

这些指令包括：  

- **config** `<!--#config timefmt="%B %Y" -->` 用于设定日期格式以及其他一些 SSI 处理的项目。   
- **echo**  `<!--#echo var="VARIABLE_NAME" -->` 将被变量值所取代。 
- 	**exec** 在主机系统上用于运行命令。 
- **include** `<!--#include virtual="file-name" -->` 插入内容。
- **flastmod** `<!--#flastmod file="filename.shtml" -->` 返回文件最后修改的时间。
- **fsize** `<!--#fsize file="filename.shtml" -->` 返回文件大小。
- **printenv** `<!--#printenv -->` 返回所有定义变量的列表。
- **set** `<!--#set var="foo" value="Bar" -->` 为用户自定义变量赋值。
- **if elif endif else**  用于生成条件部分，例如：

```  
<!--#config timefmt="%A" -->
<!--#if expr="$DATE_LOCAL = /Monday/" -->
<p>Meeting at 10:00 on Mondays</p>
<!--#elif expr="$DATE_LOCAL = /Friday/" -->
<p>Turn in your time card</p>
<!--#else -->
<p>Yoga class at noon.</p>
<!--#endif -->

```

关于使用 SSI 指令的详细信息，可参考[Apache 的 SSI 简介](http://httpd.apache.org/docs/current/howto/ssi.html)。    




## 变量  

SSI servlet 当前能实现下列变量：  

|变量名|描述|  
|---|---|  
| AUTH_TYPE |用于这些用户的验证类型：BASIC、FORM，等等。|
| CONTENT_LENGTH |从表单传入数据的长度（以字节或字符数）|
| CONTENT_TYPE |查询数据的 MIME 类型。比如：text/html|
| DATE_GMT |以格林威治标准时间（GMT）表示的当前时间与日期|
| DATE_LOCAL |以本地时区表示的当前日期与时间|
| DOCUMENT_NAME |当前文件名|
| DOCUMENT_URI |文件的虚拟路径|
| GATEWAY_INTERFACE |服务器所使用的通用网关接口（CGI）的修订版本，比如：CGI/1.1|
| HTTP_ACCEPT |客户端能够接受的 MIME 类型列表|
| HTTP_ACCEPT_ENCODING |客户端能够接受的压缩类型列表|
| HTTP_ACCEPT_LANGUAGE |客户端能够接受的语言类型列表|
| HTTP_CONNECTION |如何管理与客户端的连接："Close" 或 "Keep-Alive"|
| HTTP_HOST |客户端所请求的网站|
| HTTP_REFERER |客户端链接的文档的 URL|
| HTTP_USER_AGENT |客户端用于处理请求的浏览器|
| LAST_MODIFIED |当前文档的最后修改日期与时间|
| PATH_INFO |传入 servlet 的额外路径信息|
| PATH_TRANSLATED |变量 `PATH_INFO` 所提供路径的转换版本 |
| QUERY_STRING |在 URL 中，跟在 ？后面的查询字符串|
| QUERY_STRING_UNESCAPED |带有所有经过 `\` 转义的shell 元字符的未解码查询字符串|
| REMOTE_ADDR |用户作出请求的远端 IP 地址|
| REMOTE_HOST |用户作出请求的远端主机名|
| REMOTE_PORT |用户作出请求的远端 IP 地址的端口号|
| REMOTE_USER |用户的认证名称|
| REQUEST_METHOD |处理信息请求的方法：`GET`、`POST`，等等|
| REQUEST_URI |客户端所请求的最初页面|
| SCRIPT_FILENAME |当前页面在服务器上的位置|
| SCRIPT_NAME |页面名称|
| SERVER_ADDR |服务器的 IP 地址|
| SERVER_NAME |服务器的主机名或 IP 地址|
| SERVER_PORT |服务器接收请求的端口|
| SERVER_PROTOCOL |服务器所使用的协议，比如：HTTP/1.1|
| SERVER_SOFTWARE |应答客户端请求的服务器软件的名称与版本号|
| UNIQUE_ID |用于识别当前会话（如果存在）的令牌|

