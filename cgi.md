
## 目录  

- 简介  
- 安装  
- 配置   

## 简介  

CGI（通用网关接口）定义了一种 Web 服务器与外部内容生成程序的交互方式，这里所说的外部内容生成程序通常被称为 CGI 程序或 CGI 脚本。   

当你使用 Tomcat 做为 HTTP 服务器，并且需要 CGI 支持时，可以在 Tomcat 中添加 CGI 支持。》Tomcat 的 CGI 支持很大程度上能够跟 Apache 的httpd's 相兼容，但也存在一些局限（比如只有一个 cgi-bin 目录）。   

CGI 支持是通过 servlet 类 `org.apache.catalina.servlets.CGIServlet` 来实现的。一般而言，该 servlet 与 URL 模式“/cgi-bin/*” 相对应。  

Tomcat 默认不支持 CGI。  

## 安装  

> **警告**：CGI 脚本用于执行 Tomcat JVM 外部的程序。如果使用 Java 的 SecurityManager，则它将绕过 `catalina.policy` 中配置的安全策略。     

为了启用 CGI 支持：   

1. 在默认的 `$CATALINA_BASE/conf/web.xml` 文件中，存在被注释掉的用于 CGI servlet 的范例 servlet 及 servlet-mapping 元素。在 Web 应用中启用 CGI 支持，需要将 servlet 和 servlet-mapping 声明都复制到 Web 应用的 `WEB-INF/web.xml` 文件中。   

2. 在 Web 应用中的 Context 元素中设置 `privileged="true"`。  
	
	只有享有特权的上下文才能被允许使用 CGI servlet。注意，修改全局`$CATALINA_BASE/conf/context.xml` 文件会影响所有的 Web 应用。查阅[上下文文档](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html)来了解详情。   
	

## 配置   

下面是用来配置 CGI servlet 行为的一些 Servlet 初始参数：   

- **cgiPathPrefix** 搜索 CGI 脚本的路径，一般从 Web 应用根目录 + 文件.分隔符 + 这个前缀开始搜索。该参数默认为空值，从而使得 Web 应用根目录被用作搜索路径。建议取值为：`WEB-INF/cgi`。
- **debug** 该 servlet 所记录信息》》调试细节度。默认为 `0`。
- **executable** 用于运行脚本的可执行文件后缀名，如果脚本自身就是可执行文件（比如 .exe 文件），则可以将该参数显式设置为空的字符串。默认是 `perl`，即默认是 perl 脚本。
- **executable-arg-1 与 executable-arg-2，等等** `executable` 的其他参数。它们位于 CGI 脚本名称之前。默认不存在其他额外参数。
- **parameterEncoding** CGI Servlet 所使用的参数编码名称，默认为 `System.getProperty("file.encoding","UTF-8")`。首选系统默认编码，如果系统属性不可用，则采用 UTF-8 编码。
- **passShellEnvironment** 是否应将 Tomcat 过程的 shell 环境变量（如果存在）传入 CGI 脚本？默认为 `false`。
- **stderrTimeout** 在终止 CGI 过程之前，等待**标准错误输出信息**（stderr）读取完毕的时间（以毫秒计）。默认为 `2000`。  












 