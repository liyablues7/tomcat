# SSL/TLS 配置

## 目录   

- 快速入门  
- SSL/TLS 简介   
- SSL/TLS 与 Tomcat   
- 证书  
- 运行 SSL 通常需要注意的一些内容 
- 配置      
	1. 准备证书密钥存储库    
	2. 编辑 Tomcat 配置文件  
- 从 CA 处安装证书    
	1. 创建一个本地证书签名请求（CSR） 
	2. 导入证书   
- 疑难排解  
- 在应用中使用 SSL 跟踪会话     
- 其他技巧  

## Quick Start   

下列说明将使用变量名 $CATALINA_BASE 来表示多数相对路径所基于的基本目录。如果没有为 Tomcat 多个实例设置 CATALINA_BASE 目录，则 $CATALINA_BASE 就会设定为 $CATALINA_HOME 的值，也就是你安装 Tomcat的目录。     

在 Tomcat 中安装并配置 SSL/TLS 支持，只需遵循下列几步即可。详细信息可参看文档后续介绍。  

1. 创建一个 keystore 文件保存服务器的私有密钥和自签名证书：  
	Windows：  
	`"%JAVA_HOME%\bin\keytool" -genkey -alias tomcat -keyalg RSA`   
	UNIX：  
	`$JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA`
	
	指定密码为“changeit”。
	
2. 取消对 `$CATALINA_BASE/conf/server.xml` 中 “SSL HTTP/1.1 Connector” 一项的注释状态。按照下文中<a href="#configuration">配置</a>一节所描述的方式进行修改。  

## SSL/TLS 简介  

安全传输层协议（TLS）与其前辈加密套接字（SSL）都是用于保证 Web 浏览器与 Web 服务器通过安全连接进行通信的技术。利用这些技术，我们所要传送的数据会在一端进行加密，传输到另一端后再进行解密（在处理数据之前）。这是一种双向的操作，服务器和浏览器都能在发送数据前对它们进行加密处理。   

SSL/TLS 协议的另一个重要方面是认证。当我们初始通过安全连接与 Web 服务器进行通信时，服务器将提供给 Web 浏览器一组“证书”形式的凭证，用来证明站点的归属方以及站点的具体声明。某些情况下，服务器也会请求浏览器出示证书，来证明作为操作者的“你”所宣称的身份是否属实。这种证书叫做“客户端证书”，但事实上它更多地用于 B2B（企业对企业电子商务）的交易中，而并非针对个人用户。大多数启用了 SSL 协议的 Web 服务器都不需要客户端认证。     

## SSL/TLS 与 Tomcat  

一定要注意的是，通常只有当 Tomcat 是独立运行的 Web 服务器时，才有必要去配置 Tomcat 以便利用加密套接字。具体细节可参看 [Security Considerations Document](http://tomcat.apache.org/tomcat-8.0-doc/security-howto.html)。当 Tomcat 以 Servlet/JSP 容器的形式在其他 Web 服务器（比如 Apache 或 Microsoft IIS）背后运行时，通常需要配置的是主 Web 服务器，用主服务器来处理与用户的 SSL 连接。主服务器一般会利用所有的 SSL 相关功能，将任何只有 Tomcat 才能处理的请求进行解密后再传给 Tomcat。同样，Tomcat 会返回明文形式的响应，这些响应在被传输到用户浏览器之前会被主服务器进行加密处理。在这种情境下，Tomcat 知道主服务器与客户端的所有通信都是通过安全连接进行的（因为应用要求），但 Tomcat 自身无法参与到加密与解密的过程中。  


## 证书   

为了实现 SSL，Web 服务器必须对每一个接受安全连接的外部接口（IP 地址）配备相应的证书。这种设计方式的理论在于，服务器必须提供某种可信的保证（尤其对于接收敏感信息而言），保证它的拥有者是你所认为的角色。限于本章篇幅，不再赘述关于证书的详细解释，只需要把它认为成是一种 IP 地址的“数字驾照”即可。它声明了与站点相关的组织，以及一些关于站点拥有者或管理者的基本联系信息。    

这种“数字驾照”的持有者对其进行了加密签名，从而使得它极难伪造。对于参与电子商务的站点或者其他一些身份验证显得非常重要的商业交易来说，证书通常都是从一些比较权威公正的 CA （ Certificate Authority，认证机构）购买的，比较知名的 CA 有 VeriSign 、Thawte等。这些证书可经电子验证。实际上，CA 会保证所颁发证书的真实性，所以你完全可以放心。    

不过，在很多情况下，验证并不是问题的关键。管理员可能只想保证服务器所传输与接收的数据是秘密的，不会被某些人通过连接来窃取。幸运的是，Java 提供了一个简单的命令行工具：`keytool`。它能轻松创建一个“自签名”的证书，这种自签名证书是由用户生成的一种证书，未经任何知名 CA 给予官方保证，因此它的真实性也无法确定。再说一次，是否认证，完全根据你的需求。  


## 运行 SSL 通常需要注意的一些内容    

当用户首次访问你站点上的安全页面时，页面通常会提供给他一个对话框，包含证书相关细节（比如组织及联系方式等），并且询问他是否愿意承认该证书为有效证书，然后再进行下一步的事务。一些浏览器可能会提供一个选项，允许永远承认给出的证书的有效性，这样就不会在用户每次访问站点时打扰他们了。但有些浏览器不会提供这种选项。一旦用户承认了证书的有效性，那么在整个的浏览器会话期间，证书都被认为是有效的。  

虽然 SSL 协议的意图是尽可能有助于提供安全且高效的连接，但从性能角度来考虑，加密与解密是非茶馆耗费计算资源的，因此将整个 Web 应用都运行在 SSL 协议下是完全没有必要的，开发者需要挑选需要安全连接的页面。对于一个相当繁忙的网站来说，通常只会在特定页面上使用 SSL 协议，也就是可能交换敏感信息的页面，比如：登录页面、个人信息页面、购物车结账页面（可能会输入信用卡信息），等等。应用中的任何一个页面都可以通过加密套接字来请求访问，只需将页面地址的前缀 `http:` 换成 `https:` 即可。绝对**需要**安全连接的页面应该检查该页面请求所关联的协议类型，如果发现没有指定 `https:`，则采取适当行为。   

最后，在安全连接上使用基于名称的虚拟主机可能会造成一定的问题。这正是 SSL 协议的局限。SSL 握手过程，即客户端浏览器接收服务器证书，必须发生在 HTTP 请求被访问前。因此包含虚拟主机名称的请求信息不能先于认证而确定，也不可能为单个 IP 地址指定多个证书。如果单个 IP 地址的所有虚拟主机都需要利用同样证书来认证的话，那么多个虚拟主机不应该干涉服务器通常的 SSL 操作。但是要注意一点，多数客户端浏览器会将服务器域名与证书中的多个域名（如果存在的话，）进行对比，如果域名出现不匹配，则浏览器会向用户显示警告信息。一般来说，生产环境中，通常只有使用基于地址的虚拟主机利用 SSL。  


## <a name = "configuration">配置</a>    
 
### <a name ="Prepare Certificate Keystore">1. 准备证书密钥存储库</a>

Tomcat 目前只能操作 `JKS`、`PKCS11`、`PKCS12` 格式的密钥存储库。`JKS` 是 Java 标准的“Java 密钥存储库”格式，是通过 `keytool` 命令行工具创建的。该工具包含在 JDK 中。`PKCS12` 格式一种互联网标准，可以通过 OpenSSL 和 Microsoft 的 Key-Manager 来。  

密钥存储库中的每一项都通过一个别名字符串来标识。尽管许多密码存储库实现都在处理别名时不区分大小写，但区分大小写的实现也是允许的。比如，`PKCS11` 规范需要别名是区分大小写的。为了避免别名大小写敏感的问题，不建议使用只有大小写不同的别名。  

为了将现有的证书导入 `JKS` 密码存储库，请查阅关于 `keytool` 的相关文档（位于 JDK 文档包里）。注意，OpenSSL 经常会在密码前加上易于理解的注释，但 `keytool` 并不支持这一点。所以如果证书里的密码数据前面有注释的话，在利用 `keytool` 导入证书前，一定要清除它们。   

要想把一个已有的由你自己的 CA 所签名的证书导入使用 OpenSSL 的 `PKCS12` 密码存储库，应该执行如下命令：  

```
openssl pkcs12 -export -in mycert.crt -inkey mykey.key
                       -out mycert.p12 -name tomcat -CAfile myCA.crt
                       -caname root -chain

   
```

更复杂的实例，请参考 [OpenSSL 文档](https://www.openssl.org)的相关内容。   

下面这个实例展示的是如何利用终端命令行，从零开始创建一个新的 `JKS` 密码存储库，该密码库包含一个自签名的证书。  

Windows：    

`"%JAVA_HOME%\bin\keytool" -genkey -alias tomcat -keyalg RSA`  

Unix：   

`$JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA`   

（RSA 算法应该作为首选的安全算法，这同样也能保证与其他服务器和组件的大体的兼容性。）  

该命令将在用户的主目录下创建一个新文件：`.keystore`。要想指定一个不同的位置或文件名，可以在上述的 `keytool` 命令上添加 `-keystore` 参数，后跟到达 keystore 文件的完整路径名。你还需要把这个新位置指定到 `server.xml` 配置文件上，见后文介绍。例如：   

Windows：   

```
"%JAVA_HOME%\bin\keytool" -genkey -alias tomcat -keyalg RSA
  -keystore \path\to\my\keystore
```  

Unix：   

```  
$JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA  
  -keystore /path/to/my/keystore     
```   

执行该命令后，首先会提示你提供 keystore 的密码。Tomcat 默认使用的密码是 `changeit`（全部字母都小写），当然你可以指定一个自定义密码（如果你愿意）。同样，你也需要将这个自定义密码在 `server.xml` 配置文件内进行指定，稍后再予以详述。     

接下来会提示关于证书的一般信息，比如组织、联系人名称，等等。当用户试图在你的应用中访问一个安全页面时，该信息会显示给用户，所以一定要确保所提供的信息与用户所期望看到的内容保持一致。   

最后，还需要输入**密钥密码**（key password），这个密码是这一证书（而不是存储在同一密码存储库文件中的其他证书）的专有密码。`keytool` 提示会告诉你，如果按下回车键，则自动使用密码存储库 keystore 的密码。当然，除了这个密码，你也可以自定义自己的密码。如果选择自定义密码，那么不要忘了在 `server.xml` 配置文件中指定这一密码。     


如果操作全部正常，我们现在就会得到一个服务器能使用的有证书的密码存储库文件。


### <a name="TomcatConfigFile">2. 编辑 Tomcat 配置文件</a>   

Tomcat 能够使用两种 SSL 实现：   

- JSSE 实现，它是Java 运行时（从 1.4 版本起）的一部分。   
- APR 实现，默认使用 OpenSSL 引擎。   

详细的配置信息依赖于所用的实现方式。如果通过指定通用的 `protocol="HTTP/1.1"` 来配置连接起，那么就会自动选择 Tomcat 所要用到的实现方式。如果安装使用的是 [APR](需要中文页面链接)（比如你安装了 Tomcat 的原生库），那么它将使用 APR 的 SSL 实现，否则就将使用 Java 所提供的 JSSE 实现。   

由于这两种 SSL 实现在 SSL 支持的配置属性上有很大差异，所以**强烈建议**不要自动选择实现方式。选择实现应该采取这种方式：在[连接器](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)的 `protocol` 属性中，通过指定类名来确立实现方式。      

定义 Java（JSSE）连接器，不管 APR 库是否已经加载，都可以使用下列方式：   

```  

<!-- Define a HTTP/1.1 Connector on port 8443, JSSE NIO implementation -->
<Connector protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" .../>

<!-- Define a HTTP/1.1 Connector on port 8443, JSSE NIO2 implementation -->
<Connector protocol="org.apache.coyote.http11.Http11Nio2Protocol"
           port="8443" .../>

<!-- Define a HTTP/1.1 Connector on port 8443, JSSE BIO implementation -->
<Connector protocol="org.apache.coyote.http11.Http11Protocol"
           port="8443" .../>

```  

另一种方法，指定 APR 连接器（APR 库必须可用），则使用：   

```  
<!-- Define a HTTP/1.1 Connector on port 8443, APR implementation -->
<Connector protocol="org.apache.coyote.http11.Http11AprProtocol"
           port="8443" .../>


```  

如果使用 APR，则会出现一个选项，从而可以配置另一种 OpenSSL 引擎。   

```  
<Listener className="org.apache.catalina.core.AprLifecycleListener"
          SSLEngine="someengine" SSLRandomSeed="somedevice" />

```  

默认值为：  

```   
<Listener className="org.apache.catalina.core.AprLifecycleListener"
          SSLEngine="on" SSLRandomSeed="builtin" />

```   

所以要想使用 APR 实现，一定要确保 `SSLEngine` 属性值不能为 `off`。该属性值默认为 `on`，如果指定的是其他值，它也会成为一个有效的引擎名称。     

`SSLRandomSeed` 属性指定了一个熵源。生产系统需要可靠的熵源，但熵可能需要大量时间来采集，因此测试系统会使用非阻塞的熵源，比如像“/dev/urandom”，从而能够更快地启动 Tomcat。     

最后一步是在 `$CATALINA_BASE/conf/server.xml` 中配置连接器，`$CATALINA_BASE` 表示的是 Tomcat 实例的基本目录。Tomcat 安装的默认 `server.xml` 文件中包含一个用于 SSL 连接器的 `<Connector>` 元素的范例。为了配置使用 JSSE 的 SSL 连接器，你可能需要清除注释并按照如下的方式来编辑它。   

```   
<!-- Define a SSL Coyote HTTP/1.1 Connector on port 8443 -->
<Connector
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           keystoreFile="${user.home}/.keystore" keystorePass="changeit"
           clientAuth="false" sslProtocol="TLS"/>

```   


APR 连接器会使用很多不同的属性来设置 SSL，特别是密钥和证书。 APR 配置范例如下：   

```   
<!-- Define a SSL Coyote HTTP/1.1 Connector on port 8443 -->
<Connector
           protocol="org.apache.coyote.http11.Http11AprProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           SSLCertificateFile="/usr/local/ssl/server.crt"
           SSLCertificateKeyFile="/usr/local/ssl/server.pem"
           SSLVerifyClient="optional" SSLProtocol="TLSv1+TLSv1.1+TLSv1.2"/>

```   

每个属性所用的配置信息与选项都是强制的，可查看 [HTTP 连接器](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html#SSL_Support)配置参考文档中的 SSL 支持部分。一定要确保对所使用的连接器采用正确的属性。BIO、NIO 以及 NIO2 连接器都使用 JSSE，然而APR以及原生的连接器则使用 APR。   

`port`属性指的是 Tomcat 用以侦听安全连接的 TCP/IP 端口号。你可以随意改变它，比如改成 `https` 的默认端口号 443。但是在很多操作系统中，在低于 1024 的端口上运行 Tomcat 必须进行一番特殊的配置，限于篇幅，本文档不再赘述。  


**如果在这里，你更改了端口号，那么也应该在 非 SSL 连接器的 `redirectPort` 属性值。从而使 Tomcat 能够根据 Servlet 规范，自动对访问带有安全限制（指定需要 SSL）页面的用户进行重定向。**  


配置完全部信息后，你应该像往常一样，重新启动 Tomcat，从而能够利用 SSL 来访问任何 Tomcat 所支持的 Web 应用了。比如：  

`https://localhost:8443/`  

你将看到跟往常一样的 Tomcat 主页面（除非你修改了 ROOT 应用）。如果出现问题，这样做没有任何效果，请看下面的故障排除小节。  

## 从 CA 处安装证书   

如果想从 CA（比如 verisign.com、thawte.com、trustcenter.de）处获取并安装证书，请阅读之前的内容，然后按照下列操作进行：   

### 1. 创建一个本地证书签名请求（CSR）  

为了从选择的 CA 处获取证书，必须创建一个证书签名请求（CSR）。CA 通过 CSR 来创建出一个证书，用来证明你的网站是“安全的”。创建 CSR 的步骤如下：   

- 创建一个本地自签名证书（如前文所述）：   

	```
	keytool -genkey -alias tomcat -keyalg RSA
    -keystore <your_keystore_filename>
  
	```
	注意：在某些情况下，为了创建一个能耐生效的证书，你必须在“first- and lastname”字段中输入网站的域名（比如：`www.myside.org`）。

-  然后利用下列代码创建 CSR：     

	```
	keytool -certreq -keyalg RSA -alias tomcat -file certreq.csr
    -keystore <your_keystore_filename>

	```

现在，我们得到了一个 `certreq.csr` 的文件，可以把它提交给 CA 了（CA 的网站上应有关于如何提交的文档），审核通过后就会收到一个证书。   

### 2. 导入证书    
  

接下来可以把证书导入到本地密钥存储库中。首先你需要把链证书（Chain Certificate）或根证书（Root Certificate）导入到密钥存储库中。然后继续导入证书。  

- 从获得证书的 CA 那里下载链证书。
	如选择 Verisign.com 的商业证书，则点击：[http://www.verisign.com/support/install/intermediate.html](http://www.verisign.com/support/install/intermediate.html)。  
	如选择 Verisign.com 的试用证书，则点击：[http://www.verisign.com/support/verisign-intermediate-ca/Trial_Secure_Server_Root/index.html](http://www.verisign.com/support/verisign-intermediate-ca/Trial_Secure_Server_Root/index.html)。   
	如选择 Trustcenter.de，则点击：[http://www.trustcenter.de/certservices/cacerts/en/en.htm#server](http://www.trustcenter.de/certservices/cacerts/en/en.htm#server)。   
	如选择 Thawte.com，则点击：[http://www.thawte.com/certs/trustmap.html](http://www.thawte.com/certs/trustmap.html)。    
	
- 将链证书导入密钥存储库：  
	
	```   
	keytool -import -alias root -keystore <your_keystore_filename>
    -trustcacerts -file <filename_of_the_chain_certificate>

	```   
	
- 最后导入你的新证书：  
	
	```  
	keytool -import -alias tomcat -keystore <your_keystore_filename>
    -file <your_certificate_filename>

	```  

## 疑难排解  

以下列出了一些在设置 SSL 通信时经常会遇到的问题及其解决方法：   

- 当 Tomcat 启动时，出现这样的异常信息：“ava.io.FileNotFoundException: {some-directory}/{some-file} not found.”     

	这很有可能是因为 Tomcat 无法在指定位置处找到密钥存储库。默认情况下，密钥存储库文件应以 `.keystore` 为后缀，位于用户的 home 目录下（也许很你的设置不同）。如果密钥存储库文件在别处，则需要在 <a href="#TomcatConfigFile">Tomcat 配置文件</a>的 `<Factory>` 元素中添加一个 `keystoreFile` 属性。     
	
- 当 Tomcat 启动时，出现这样的异常信息：“java.io.FileNotFoundException: Keystore was tampered with, or password was incorrect.”      

	假如排除了有人恶意篡改密钥存储库文件的因素，那么出现这样的异常，最有可能是因为 Tomcat 现在所用的密码不同于你当初创建密钥存储库文件时所用密码。为了解决这个问题，你可以<a href="#Prepare Certificate Keystore">重新创建密钥存储库文件</a>，或者在 <a href="#TomcatConfigFile">Tomcat 配置文件</a>的 `<Connector>` 元素中添加或更新一个 `keystoreFile` 属性。**注意**：密码都是区分大小写的。   
	
- 当 Tomcat 启动时，出现这样的异常：“java.net.SocketException: SSL handshake error javax.net.ssl.SSLException: No available certificate or key corresponds to the SSL cipher suites which are enabled.”  

	出现这样的异常，很有可能是因为 Tomcat 无法在指定的密钥存储库中找到服务器密钥的别名。查看一下 <a href="#TomcatConfigFile">Tomcat 配置文件</a>的 `<Connector>` 元素中所指定的 `keystoreFile` 和 `keyAlias` 值是否正确。**注意**：`keyAlias` 值可能区分大小写。   
	
如果出现了其他问题，可以求助 **TOMCAT-USER** 邮件列表，你可以在该邮件列表内找到之前消息的归档文件，以及订阅和取消订阅的相关信息。Tomcat 邮件列表的链接是：[http://tomcat.apache.org/lists.html](http://tomcat.apache.org/lists.html)。     

## 在应用中使用 SSL 跟踪会话     

这是一个 Servlet 3.0 规范中的新功能。由于它将 SSL 会话 ID 关联到物理的客户端服务器连接上，所以导致了一些约束与局限：   

- Tomcat 必须有一个属性 **isSecure** 设为 `true` 的连接器。   
- 如果 SSL 连接器通过代理或硬件加速器来管理，则它们必须填充 SSL 请求报头（参见 [SSLValve](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html)），从而使 SSL 会话 ID 可见于 Tomcat。 
- 如果 Tomcat 终止了 SSL 连接，则不可能采用会话重复，因为 SSL 会话 ID 在两个端点处都是不一样的。    

为了开启 SSL 会话跟踪，只需使用上下文侦听器将上下文的跟踪模式设定为 SSL 即可（如果还开启了其他跟踪模式，则会优先使用它）。如下所示：   

```  
package org.apache.tomcat.example;

import java.util.EnumSet;

import javax.servlet.ServletContext;
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.SessionTrackingMode;

public class SessionTrackingModeListener implements ServletContextListener {

    @Override
    public void contextDestroyed(ServletContextEvent event) {
        // Do nothing
    }

    @Override
    public void contextInitialized(ServletContextEvent event) {
        ServletContext context = event.getServletContext();
        EnumSet<SessionTrackingMode> modes =
            EnumSet.of(SessionTrackingMode.SSL);

        context.setSessionTrackingModes(modes);
    }

}


```   

注意：SSL 会话跟踪是针对 BIO、NIO 以及 NIO2 连接器来实现的，目前还没有针对 APR 连接器的实现。

## 其他技巧

要想从请求中访问 SSL 会话 ID，可使用：   

`String sslID = (String)request.getAttribute("javax.servlet.request.ssl_session_id");`  

关于这一方面的其他内容，可参看[Bugzilla](https://bz.apache.org/bugzilla/show_bug.cgi?id=22679)。   

为了终止 SSL 会话，可以使用：   

```   
// Standard HTTP session invalidation
session.invalidate();

// Invalidate the SSL Session
org.apache.tomcat.util.net.SSLSessionManager mgr =
    (org.apache.tomcat.util.net.SSLSessionManager)
    request.getAttribute("javax.servlet.request.ssl_session_mgr");
mgr.invalidateSession();

// Close the connection since the SSL session will be active until the connection
// is closed
response.setHeader("Connection", "close");

```   

注意：由于使用了 SSLSessionManager 类，所以这段代码是专门针对 Tomcat 的。当前只适用于 BIO、NIO 以及 NIO2 连接器，不适合 APR/原生连接器。  
