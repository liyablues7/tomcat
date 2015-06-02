## 目录  

## 简介  

对于大多数用例来说，默认配置下的 Tomcat 都是相当安全的。有些环境可能需要更多（或更少）的安全配置。本文统一介绍了一下可能影响安全性的配置选项，并适当说明了一下修改这些选项所带来的预期影响。目的是为了在评价 Tomcat 安装时，提供一些应值得考虑的配置选项。

**注意**：本章内容毕竟有所局限，你还需要对配置文档进行深入研究。在相关文档中有更完整的属性描述。       

## 非 Tomcat 设置  

Tomcat 配置不应成为唯一的防线，也应该保障系统（操作系统、网络及数据库，等等）中的其他组件的安全。    

不应该以根用户的身份来运行 Tomcat，应为 Tomcat 进程创建并分配一个专门的用户，并为该用户配置最少且必要的操作系统权限。比如，不允许使用 Tomcat 用户实现远程登录。    

文件权限同样也应适当限制。就拿 ASF 中的 Tomcat 实例为例说明吧（禁止自动部署，Web 应用被部署为扩张的目录。），标准配置规定所有的 Tomcat 文件都由根用户及分组用户所拥有，拥有者具有读写特权，分组只有读取特权，而 World 则没有任何特权。例外之处在于，logs、temp 以及 work 目录的权限都由 Tomcat 用户而不是根用户所拥有。这意味着即使攻击者破坏了 Tomcat 进程，他们也不能改变 Tomcat 配置，无法部署新应用，也无法修改现有应用。Tomcat 进程使用掩码 007 来维护这种权限许可。  

对于网络层面，需要使用防火墙来限制进站与出站连接，只允许出现那些你希望的连接。   


## 默认的 Web 应用  

### 概述    

Tomcat 安装时自带了一些默认启用的 Web 应用。过去一段时间内发现了不少关于这些应用的漏洞。用不到的应用就该删除，以避免给系统带来相关漏洞而产生的安全风险。	

### ROOT  

ROOT 应用带来安全风险的可能性非常小，但它确实含有正在使用的 Tomcat 的版本号。应该从可公开访问的 Tomcat 实例中清除 ROOT 应用，不是出于安全性原因，而是因为这样能给用户提供一个更适合的默认页面。   

### Documentation   

Documentation 带来安全风险的可能性非常小，但》它标识出了当前正使用的 Tomcat 版本。应该从可公开访问的 Tomcat 实例中清除该应用。   

### Examples   

应该从安全敏感性安装中移除 examples 应用。虽然 examples 应用并不包含任何已知的漏洞，但现已证明，它所包含的一些功能可以被攻击者利用，特别是一些显示所有接收内容，并且能设置新 cookie 的 cookie 范例。攻击者将这些公关和部署在 Tomcat 实例中的另一个应用中的漏洞相结合，就能获取原本根本不可能得到的信息。

### Manager  

由于 Manager 应用允许远程部署 Web 应用，所以经常被攻击者利用，因为应用的密码普遍强度不够，而且大多在 Manager 应用中启用了 Tomcat 实例可公开访问的功能。Manager 应用默认是不能访问的，因为没有配置能够执行这种访问的用户。如果启用 Manager 应用，就应该遵循 **保证管理型应用的安全性** 一节中的指导原则。  

### Host Manager  

Host Manager 应用能够创建并管理虚拟主机，包括启用虚拟主机的 Manager 应用。Host Manager 应用默认是不能访问的，因为没有配置能够执行这种访问的用户。如果启用 Host Manager 应用，就应该遵循 **保证管理型应用的安全性** 一节中的指导原则。      

### 保证管理型应用的安全性

在配置能够为 Tomcat 实例提供管理功能的 Web 应用时，需要遵循下列指导原则：  

- 保证任何被允许访问管理应用的用户的密码是强密码。  
- 不要放弃使用 [LockOutRealm](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html#LockOut_Realm_-_org.apache.catalina.realm.LockOutRealm)，它能防止暴力破解者攻击用户密码。  
- 将 `/META-INF/context.xml` 中的限制访问 localhost 的[RemoteAddrValve](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Remote_Address_Filter) 取消注释。如果需要远程访问，使用该值可限制到特定的 IP 地址。







## 安全管理器（Security Manager）

启用安全管理器能让 Web 应用运行在沙盒中，从而极大限制 Web 应用执行恶意行为的能力——比如调用 `System.exit()`，在 Web 应用根目录或临时目录外建立网络连接或访问文件系统。但应注意的是，有些恶意行为是安全管理器所无法阻止的，比如利用无限循环产生 CPU 极大开销。  

启用安全管理器经常用来限制潜在影响，比如防止攻击者通过某种方式危害受信任的 Web 应用。安全管理器也可能被用来减少运行不受信任的 Web 应用（比如在托管环境中）所带来的风险，但它只能减少这种风险，并不能终止不受信任的 Web 应用。如果运行多个不受信任的 Web 应用，强烈建议将每个应用都部署为独立的 Tomcat 实例（理想情况下还需要部署在独立的主机上），以便尽量减少恶意 Web 应用对其他应用产生的危害。  

Tomcat 已经过安全管理器的测试。但大多数 Tomcat 用户却没有运行过安全管理器，所以 Tomcat 也没有相关的用户测试过的配置。现在已经（并会继续）报告指出了一些关于运行安全管理器时产生的 Bug。    

安全管理器在运行时所暴露出的限制可能在于中断很多应用。所以，在未经大量测试前，还是不要使用它为好。理想情况下，安全管理器应该在开发前期使用，因为对于一个成熟的应用来说，启用安全管理器后，记录修补问题会极大浪费时间。  

启用安全管理器会改变下列设置的默认值：  

- Host 元素的 `deployXML` 属性默认值会被改为 `false`。   

## server.xml 中的关键配置    

### 1. 综述  

默认的 server.xml 包含大量注释，比如一些被注释掉的范例组件定义。去掉这些注释将会使其更容易阅读和理解。  

如果某个组件类型没有列出，那么该类型也没有能够直接影响安全的相关设置。  

### 2. server    

将 `port` 属性设为 `-1` 能禁用关闭端口。  

如果关闭端口未被禁用，会为 **shutdown** 配置一个强密码。  


### 3. 侦听器  

如果在 Solaris 上使用 gcc 编译 APR 生命周期侦听器，你会发现 APR 生命周期侦听器并不稳定。如果在 Solaris 上使用 APR（或原生）连接器，需要用 Sun Studio 编译器进行编译。  

应该启用并恰当地配置 Security 侦听器。   

### 4. 连接器  

默认配置了一个 HTTP 和 AJP 连接器。没有用到的连接器应从 server.xml 中清除掉。  

`address` 属性用来控制连接器在哪个  IP 地址上侦听连接。默认，连接器会在所有配置好的 IP 地址上进行侦听。  

`allowTrace` 属性可启用能够利于调试的 TRACE 请求。由于一些浏览器处理 TRACE 请求的方式（将浏览器暴露给 XSS 攻击），所以默认是不支持 TRACE 请求的。  

`maxPostSize` 属性控制解析参数的 POST 请求的最大尺寸。在整个请求期间，参数会被缓存，所以该值默认会被限制到 2 MB大小，以减少 DOS 攻击的风险。    

`maxSavePostSize` 属性控制在 FORM 和 CLIENT-CERT 验证期间，saving of POST requests。在整个验证期间（可能会占用好几分钟），参数会被缓存，所以该值默认会被限制到 4 KB大小，以减少 DOS 攻击的风险。   

`maxParameterCount` 属性控制可解析并存入请求的参数与值对（GET + POST）的最大数量。过多的参数将被忽略。如果想拒绝这样的请求，配置[FailedRequestFilter](http://tomcat.apache.org/tomcat-8.0-doc/config/filter.html)。   

`xpoweredBy` 属性控制是否 X-Powered-By HTTP 报头会随每一个请求发送。如果发送，则该报头值包含 Servlet 和 JSP 规范版本号、完整的 Tomcat 版本号（比如 Apache Tomcat/8.0）、JVM Vendor 名称，以及 JVM 版本号。默认禁用该报头。该报头可以为合法用户和攻击者提供有用信息。  

`server` 属性控制 Server HTTP 报头值。对于 Tomcat 4.1.x 到 8.0.x，该报头默认值为 Apache-Coyote/1.1。该报头为合法用户和攻击者提供的有用信息是有限的。  

`SSLEnabled`、`scheme` 和 `secure` 这三个属性可以各自独立设置。这些属性通常应用场景为：当 Tomcat 位于反向代理后面，并且该代理通过 HTTP 或 HTTPS 连接 Tomcat 时。通过这些属性，可查看客户端与代理间（而不是代理与 Tomcat之间）连接的 SSL 属性。例如，客户端可能通过 HTTPS 连接代理，但代理连接 Tomcat 却是通过 HTTP。如果 Tomcat 有必要区分从代理处接收的安全与非安全连接，那么代理就必须使用单独分开的连接，向 Tomcat 传递安全与非安全请求。如果代理使用 AJP，客户端连接的 SSL 属性会经由 AJP 协议传递，那么就不需要使用单独的连接。   


`sslEnabledProtocols` 属性用来确定所使用的 SSL/TLS 协议的版本。从 2014 年发生的 POODLE 攻击起，SSL 协议被认为是不安全的，单独 Tomcat 设置中该属性的安全设置为 `sslEnabledProtocols="TLSv1,TLSv1.1,TLSv1.2"`。  

`ciphers` 属性控制 SSL 连接所使用的 cipher。默认使用 JVM 的缺省 cipher。这往往意味着，可用 cipher 列表将包含弱导出级 cipher。安全环境通常需要配置更受限的 cipher 集合。该属性可以利用 [OpenSSL 语法格式](https://www.openssl.org/docs/apps/ciphers.html)来包括/排除 cipher 套件。截止 2014 年 11 月 19 日，对于单独 Tomcat 8 与 Java 8，可使用 `sslEnabledProtocols` 属性，并且排除非 DH cipher，以及弱/失效 cipher 来指定 TLS 协议，从而实现正向加密（Forward Secrecy）技术。对于以上这些设定工作来说，[Qualys SSL/TLS test](https://www.ssllabs.com/ssltest/index.html)是一个非常不错的配置工具。  

`tomcatAuthentication` 和 `tomcatAuthorization` 属性都用于 AJP 连接，用于确定 Tomcat 是否应该处理所有的认证和授权，或者是否应委托反向代理来认证（认证用户名作为 AJP 协议的一部分被传递给 Tomcat），而让 Tomcat 继续执行授权。

`allowUnsafeLegacyRenegotiation` 属性提供对 [CVE-2009-3555 漏洞](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3555)（一种 TLS 中间人攻击）的应对方案，应用于 BIO 连接器中。如果底层 SSL 实现易受 CVE-2009-3555 漏洞影响，才有必要使用该属性。参看 [Tomcat 8 安全文档](http://tomcat.apache.org/security-8.html)可详细了解这种缺陷的当前状态及其可使用的解决方案。  


AJP 连接器中的 `requiredSecret` 属性配置了 Tomcat 与 Tomcat 前面的反向代理之间的共享密钥，从而防止通过 AJP 协议进行非授权连接。  

### 5. Host 元素

Host 元素控制着部署。自动部署能让管理更为轻松，但也让攻击者更容易部署恶意应用。自动部署由 `autoDeploy` 和 `deployOnStartup` 属性来控制。如果两个属性值为 `false`，则 server.xml 中定义的上下文将会被部署，任何更改都将需要重启 Tomcat 才能生效。  

在 Web 应用不受信任的托管环境中，将 `deployXML` 设置为 `false` 将忽略任何包装 Web 应用的 context.xml，可能会把增加特权赋予 Web 应用。注意，如果启用安全管理器，则 `deployXML` 属性默认为 `false`。  


### 6. Context 元素  

`server.xml`、默认的 `context.xml` 文件，每个主机的 `context.xml.default` 文件、Web 应用上下文文件  

`crossContext` 属性能够控制是否允许上下文访问其他上下文资源。默认为 `false`，而且只应该针对受信任的 Web 应用。  

`privileged` 属性控制是否允许上下文使用容器提供的 servlet，比如 Manager servlet。默认为 `false`，而且只针对受信任的 Web 应用。  

内嵌 Resource 元素中的 `allowLinking` 属性控制是否允许上下文使用链接文件。如果启用而且上下文未经部署，那么当删除上下问资源时，也会一并将链接文件删除。默认值为 `false`。在大小写敏感的操作系统上改变该值，将会禁用一些安全措施，并且允许直接访问 WEB-INF 目录。  

### 7. Valve  

强烈建议配置 AccessLogValve。默认的 Tomcat 配置包含一个 AccessLogValve。通常会对每个 Host 上进行配置，但必要时也可以在每个 Engine 或 Context 上进行配置。   

应通过 RemoteAddrValve 来保护管理应用。注意：这个 Valve 也可以用作过滤器。*allow* 属性用于限制对一些已知信任主机的访问。  

默认的 ErrorReportValve 在发送给客户端的响应中包含了 Tomcat 版本号。为了避免这一点，可以在每个 Web 应用上配置自定义错误处理器。另一种方法是，可以配置 [`ErrorReportValve`](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html)，将其 `showServerInfo` 属性设为 `false`。另外，通过创建带有下列内容的 CATALINA_BASE/lib/org/apache/catalina/util/ServerInfo.properties 文件，可以改变版本号。  

`server.info=Apache Tomcat/8.0.x`   

根据需要来改变该值。注意，这也会改变一些管理工具所报告的版本号，可能难于确定实际安装的版本号。`CATALINA_HOME/bin/version.bat|sh` 脚本依然能够报告版本号。   

当出现错误时，默认的 `ErrorReportValve` 能向客户端显示堆栈跟踪信息以及/或者 JSP 源代码。为了避免这一点，可以在每个 Web 应用内配置自定义错误处理器。另一种方法是，可以显式配置一个 `ErrorReportValve`，并将其 `showReport` 属性设为 `false`。   


### 8. Realm   

MemoryRealm 并不适用于生产用途，因为要想让 Tomcat-users.xml 中的改动生效，就必须重启 Tomcat。  

JDBCRealm 也不建议用于生产环境，因为所有的认证和授权选项都占用一个线程。可以用 DataSourceRealm 来替代它。  

UserDatabaseRealm 不适合大规模安装。它适合小规模且相对静态的环境。  

JAASRealm 使用并不广泛，因此也不如其他几个 Realm 成熟。在未进行大量测试之前，建议不采用这种 Realm。  

默认，Realm 并不能实现账户锁定。这就给暴力破解者带来了方便。要想防范这一点，需要将 Realm 包装在 LockOutRealm 中。  


### 9. Manager   

manager 组件用来生成会话 ID。  

可以利用 `randomClass` 属性来改变生成随机会话 ID 的类。  

可以利用 `sessionIdLength` 属性来改变会话 ID 的长度。   


## 系统属性    

将系统属性 `org.apache.catalina.connector.RECYCLE_FACADES` 设为 `true`，将为每一个请求创建一个新的门面（facade）对象，这能减少因为应用 bug 而将一个请求中数据暴露给其他请求的风险。

系统属性 `org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH` 和 `org.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH ` 允许对请求 URI 的非标准解析。使用这些选项，当处于反向代理后面时，攻击者可以绕过由代理所强制设定的各种安全限制。

如果禁用系统属性 `org.apache.catalina.connector.Response.ENFORCE_ENCODING_IN_GET_WRITER` 可以会带来不利后果。许多违反 RFC2616 的用户代理在应该使用 ISO-8859-1 的规范强制默认值时，试图猜测文本媒体类型的字符编码。一些浏览器会解析为 UTF-7 编码，这样做的后果是：如果某一响应包含的字符对 ISO-8859-1 是安全的，但如果解析为 UTF-7，却能触发 XSS 漏洞。   


## Web.xml      

如果Web 应用中默认的 `conf/web.xml` 和 `WEB-INF/web.xml` 文件定义了本文提到的组件，》》  

在配置 [DefaultServlet](http://tomcat.apache.org/tomcat-8.0-doc/default-servlet.html) 时，将 `readonly` 设为 `true`。将其变为 `false` 能让客户端删除或修改服务器上的静态资源，进而上传新的资源。由于不需要认证，故而通常也不需要改变。  

将 `DefaultServlet` 的 `listings` 设为 `false`。之所以这样设置，不是因为允许目录列表是不安全之举，而是因为要对包含数千个文件的目录生产目录列表，会大量消耗计算资源，会容易导致 DOS 攻击。   

`DefaultServlet` 的 `showServerInfo` 设为 `true`。当启用目录列表后，Tomcat 版本号就会包含在发送给客户端的响应中。为了避免这一点，可以将 `DefaultServlet` 的 `showServerInfo` 设为 `false` 。另一种方法是，另外，通过创建带有下列内容的 CATALINA_BASE/lib/org/apache/catalina/util/ServerInfo.properties 文件，可以改变版本号。  

`server.info=Apache Tomcat/8.0.x`   

根据需要来改变该值。注意，这也会改变一些管理工具所报告的版本号，可能难于确定实际安装的版本号。`CATALINA_HOME/bin/version.bat|sh` 脚本依然能够报告版本号。   

可以设置 `FailedRequestFilter` 来拒绝那些请求参数解析时发生错误的请求。没有过滤器，默认行为是忽略无效或过多的参数。  

`HttpHeaderSecurityFilter` 可以为响应添加报头来提高安全性。如果客户端直接访问 Tomcat，你可能就需要启用这个过滤器以及它所设定的所有报头（除非应用已经设置过它们）。如果通过反向代理访问 Tomcat，该过滤器的配置需要与反向代理所设置的任何报头相协调。     


## 综述  

BASIC 与 FORM 验证会将用户名及密码存为明文。在不受信任的网络情况下，使用这种认证机制的 Web 应用和客户端间的连接必须使用 SSL。   

会话 cookie 加上已认证用户，基本上就将用户密码摆在攻击者面前了，无论何时给予跟密码级别相同的保护。通常这就需要经过 SSL 来认证，或者在整个会话期间都使用 SSL。


























