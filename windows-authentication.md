# Windows 验证  
## 目录  

- 概述  
- 内建 Tomcat 支持  
	1. 域控制器  
	2. Tomcat 实例（Windows 服务器）  
	3. Tomcat 实例（Linux 服务器）  
	4. Web 应用  
	5. 客户端   
	6. 参考资料  
- 第三方库  
	1. Waffle   
	2. Apache httpd  
	3. SourceForge 的 SPNEGO 项目   
	4. Jespa  
- 反向代理  
	1. Microsoft IIS  
	2. Apache httpd   



## 概述  
**集成 Windows 验证**（Integrated Windows authentication）往往用于局域网环境中，因为需要使用服务器执行验证，被验证的用户也必须处于同一域内。为了能够自动验证用户，用户所用的客户端机器也必须处于同一域内。  

可以利用以下几种方案来实现 Tomcat 下的集成 Windows 验证：   

- 内建 Tomcat 支持。  
- 使用第三方库，比如 Waffle。  
- 使用支持 Windows 验证的反向代理来执行验证步骤（IIS 或 httpd）。  

下面将分别详细讲述这些方案。    

## 内建 Tomcat 支持  

需要仔细配置 Kerberos 身份验证服务（集成 Windows 验证的基础）。如果严格按照下列步骤去做，配置就会生效。这些配置的灵活度很小，所以必须严格按照下列方式去做。从测试到现在，已知的规则是：  

- 用于访问 Tomcat 服务器的主机名必须匹配 **服务主体名称**（Service Principal Name，SPN）中的主机名，否则验证就会失败。验证失败时，校验和错误会报告给调试日志。  
- 客户端必须明确服务器位于本地可信局域网。   
- SPN 必须是 HTTP/<主机名> 的形式，而且必须在所有用到它的位置处保持统一。
- 端口号不能放在 SPN 中。  
- 不能将多个 SPN 映射给一个域用户。  
- Tomcat 必须以 SPN 关联的域账户或域管理员的身份运行，但**不**建议采用域管理员的身份运行 Tomcat。  
- 在 ktpass 命令中，域名（`DEV.LOCAL`）不区分大小写，在jaas.conf 中也是这样。  
- 使用 ktpass 命令时，不能指定域。  

在配置 Windows 验证的 Tomcat 内建支持时，共涉及到4个组件：域控制器、托管 Tomcat 的服务器、需要使用 Windows 验证的 Web 应用，以及客户端机器。下面将讲解每个组件所需的配置。  

下面配置范例中用到的 3 个机器名称为：`win-dc01.dev.local` （域控制器）、`win-tc01.dev.local `（Tomcat 实例）、`win-pc01.dev.local` （客户端）。它们都是`DEV.LOCAL` 域成员。  

**注意**：为了在下面的步骤中使用密码，不得不放宽了域密码规则，对于生产环境，可不建议这么做。  

### 1. 域控制器  

下列步骤假设前提是：经过配置，服务器可以做为域控制器来使用。关于如何配置 Windows 服务器配置成域控制器，不在本章讨论范围之内。  
配置域控制器，使 Tomcat 支持 Windows 验证的步骤为：  

- 创建一个域用户，它将映射到 Tomcat 服务器所用的服务名称上。在本文档中，用户为 `tc01`，密码为 `tc01pass`。  
- 将 SPN 映射到用户账户上。SPN 的形式为：`<service class>/<host>:<port>/<service name>`。本文档所用的 SPN 为 `HTTP/win-tc01.dev.local`。要想将用户映射到 SPN 上，运行以下命令：  

	`setspn -A HTTP/win-tc01.dev.local tc01`  
	
- 生成 keytab 文件，Tomcat 服务器会用该文件将自身注册到域控制器上。该文件包含用于服务提供者账户的 Tomcat 私钥，所以也应该受到保护。运行以下命令生成该文件（全部命令都应写在同一行中）：  

	`ktpass /out c:\tomcat.keytab /mapuser tc01@DEV.LOCAL
          /princ HTTP/win-tc01.dev.local@DEV.LOCAL
          /pass tc01pass /kvno 0`  

- 创建客户端所用的域用户。本文档中，域用户为 `test`，密码为 `testpass`。  

以上步骤测试环境为：运行 Windows Server 2008 R2 64 位标准版的域控制器。对于域功能级别和林（forest）功能级别，使用 Windows Server 2003 的功能级别。   

### 2. Tomcat 实例（Windows 服务器）  

下列步骤假定前提为：已经安装并配置好了 Tomcat 和 Java 6 JDK/JRE，并以 `tc01@DEV.LOCAL` 用户来运行 Tomcat。配置用于 Windows 验证的 Tomcat 实例的步骤如下：  

- 将域控制器所创建的 `tomcat.keytab` 文件复制到 `$CATALINA_BASE/conf/tomcat.keytab`。  
- 创建 kerberos 配置文件 `$CATALINA_BASE/conf/krb5.ini`。本文档使用的文件包含以下内容：  

	```  
	[libdefaults]
	default_realm = DEV.LOCAL
	default_keytab_name = FILE:c:\apache-tomcat-8.0.x\conf\tomcat.keytab
	default_tkt_enctypes = rc4-hmac,aes256-cts-hmac-sha1-96,aes128-cts-	hmac-sha1-96
	default_tgs_enctypes = rc4-hmac,aes256-cts-hmac-sha1-96,aes128-cts-	hmac-sha1-96
	forwardable=true

	[realms]
	DEV.LOCAL = {
   	     kdc = win-dc01.dev.local:88
	}

	[domain_realm]
	dev.local= DEV.LOCAL
	.dev.local= DEV.LOCAL
	```  
	该文件的位置可以通过 `java.security.krb5.conf` 系统属性来修改。
- 创建 JAAS 逻辑配置文件 `$CATALINA_BASE/conf/jaas.conf`。本文档使用的文件包含以下内容：  
	
	```    
	com.sun.security.jgss.krb5.initiate {
    com.sun.security.auth.module.Krb5LoginModule required
    doNotPrompt=true
    principal="HTTP/win-tc01.dev.local@DEV.LOCAL"
    useKeyTab=true
    keyTab="c:/apache-tomcat-8.0.x/conf/tomcat.keytab"
    storeKey=true;
};

	com.sun.security.jgss.krb5.accept {
	    com.sun.security.auth.module.Krb5LoginModule required
	    doNotPrompt=true
	    principal="HTTP/win-tc01.dev.local@DEV.LOCAL"
   	 useKeyTab=true
	    keyTab="c:/apache-tomcat-8.0.x/conf/tomcat.keytab"
   	 storeKey=true;
	};

	
	```  
	
	本文件位置可以通过 `java.security.auth.login.config` 系统属性来修改。所用的 LoginModule 是 JVM 所专有的，从而能保证所指定的 LoginModule 匹配所用的 JVM。登录配置名称必须与[验证 valve](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#SPNEGO_Valve) 所用值相匹配。	  
	
SPNEGO 验证器适用于任何 Realm，但如果和 JNDI Realm 一起使用的话，JNDI Realm 默认将使用用户的委托凭证（delegated credentials）连接 Active 目录。  

上述步骤测试环境为：Tomcat 服务器运行于 Windows Server 2008 R2 64 位标准版上，带有 Oracle 1.6.0_24 64 位 JDK。  

### 3. Tomcat 实例（Linux 服务器）  

测试环境如下：  

- Java 1.7.0, update 45, 64-bit
- Ubuntu Server 12.04.3 LTS 64-bit
- Tomcat 8.0.x (r1546570)    

虽然建议使用最新的稳定版本，但其实所有 Tomcat 8 的版本都能使用。  

配置与 Windows 基本相同，但存在以下一些差别：  

- Linux 服务器不必位于 Windows 域。  
- 应该更新 krb5.ini 和 jass.conf 中的 keytab 文件路径，以便适应使用 Linux 文件路径风格（比如：/usr/local/tomcat/...）的 Linux 服务器。  

### 4. Web 应用  

配置 Web 应用，以便使用 web.xml 中的 Tomcat 专有验证方法 `SPNEGO`（而不是 `BASIC` 等）。和其他的验证器一样，通过显式地配置[验证 valve](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#SPNEGO_Valve)并且在 Valve 中设置属性来自定义行为。  

### 5. 客户端  

配置客户端，以便使用 Kerberos 认证。对于 IE 浏览器来说，这就需要 Tomcat 实例位于“本地局域网”安全域中，并且需要在“工具 > Internet 选项 > 高级”中启用集成 Windows 认证。注意：客户端和 Tomcat 实例不能使用同一台机器，因为 IE 会使用未经证实的 NTLM 协议。   
 

### 6. 参考资料    

正确配置 Kerberos 验证是有一定技巧性的。下列参考资料有一定帮助。一般来说，[Tomcat 用户邮件列表](http://tomcat.apache.org/lists.html#tomcat-users)中的建议也是可取的。  

1. [IIS 与 Kerberos](http://www.adopenstatic.com/cs/blogs/ken/archive/2006/10/19/512.aspx)
2. [SourceForge 的 SPNEGO 项目](http://spnego.sourceforge.net/index.html)
3. [Oracle Java GSS-API 教程（Java 7）](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/index.html)
4. [Oracle Java GSS-API 教程 - 疑难解答（Java 7）](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/Troubleshooting.html)
5. [用于 Windows 验证的Geronimo 配置](https://cwiki.apache.org/confluence/display/GMOxDOC21/Using+SPNEGO+in+Geronimo#UsingSPNEGOinGeronimo-SettinguptheDomainControllerMachine)
5. [Kerberos 交换中的加密选择](http://blogs.msdn.com/b/openspecification/archive/2010/11/17/encryption-type-selection-in-kerberos-exchanges.aspx)
6. [受支持的 Kerberos Cipher 套件](https://support.microsoft.com/zh-cn/kb/977321)

## 第三方库   

### 1. Waffle   

关于该解决方案的完整详情，可浏览 [Waffle 网站](http://waffle.codeplex.com)。其关键特性为：  

- Drop-in   
- 配置简单（无需 JAAS 或 keytab 配置）
- 使用原生库  

### 2. Spring Security - Kerberos 扩展  

关于该解决方案的完整详情，可浏览 [Kerberos 扩展网站](http://static.springsource.org/spring-security/site/extensions/krb/index.html)。其关键特性为：  

- xx  
- 需要生成 Kerberos keytab 文件  
- 纯粹 Java 解决方案  




### 3. SourceForge 的 SPNEGO 项目  

关于该解决方案的完整详情，可浏览 [该项目网站](http://sourceforge.net/projects/spnego/)。其关键特性为：  

- 使用 Kerberos。  
- 纯 Java 解决方案。  


### 4. Jespa
关于该解决方案的完整详情，可浏览 [该项目网站](http://www.ioplex.com)。其关键特性为：   

- 纯 Java 解决方案  
- 高级 Active 目录集成  

## 反向代理  

### 1. Microsoft IIS  

通过配置 IIS 提供 Windows 验证的步骤如下：  

1. 将 IIS 配置成 Tomcat 的反向代理（参看 [IIS Web 服务器文档](http://tomcat.apache.org/connectors-doc/webserver_howto/iis.html)）。  
2. 配置 IIS 使用 Windows 验证。  
3. 将 [AJP 连接器](http://tomcat.apache.org/tomcat-8.0-doc/config/ajp.html)上的 `tomcatAuthentication` 属性设为 `false`，从而配置 Tomcat 使用来自 IIS 的验证用户信息。另一种方法是，将 `tomcatAuthorization` 设为 `true`，从而在Tomcat 执行授权时，允许 IIS 进行验证。  


### 2. Apache httpd

Apache httpd 默认并不支持 Windows 验证，但可以使用很多第三方模块来实现：  

1. 针对 Windows 平台的 [mod_auth_sspi](http://sourceforge.net/projects/mod-auth-sspi/)  
2. 针对非 Windows 平台的 [mod_auth_ntlm_winbind](http://adldap.sourceforge.net/wiki/doku.php?id=mod_auth_ntlm_winbind)。目前已知适用于 32 位平台上的 httpd 2.0.x。有些用户已经报告了 httpd 2.2.x 构建与 64 位Linux 构建所存在的稳定性问题。  

采用以下步骤配置 httpd，以便提供 Windows 验证：  

1. 将 httpd 配置成 Tomcat 的反向代理（参看 [Apache httpd Web 服务器文档](http://tomcat.apache.org/connectors-doc/webserver_howto/apache.html)）。  
2. 配置 httpd 使用 Windows 验证。  
3. 将 [AJP 连接器](http://tomcat.apache.org/tomcat-8.0-doc/config/ajp.html)上的 `tomcatAuthentication` 属性设为 `false`，从而配置 Tomcat 使用来自 httpd 的验证用户信息。



