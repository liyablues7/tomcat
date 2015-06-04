# 8. JNDI 资源


## 目录  
- 本章概述  
- web.xml 配置  
- context.xml 配置  
- 全局配置  
- 使用资源  
- Tomcat 标准资源工厂（Standard Resource Factories）  
	1. 常见 JavaBean 资源  
	2. UserDatabase 资源  
	3. JavaMail 会话  
	4. JDBC 数据资源  
- 添加自定义资源工厂  


## 本章概述   

Tomcat 为每个在其上运行的 Web 应用都提供了一个 JNDI 的 `InitialContext` 实现实例，它与[Java 企业版](http://www.oracle.com/technetwork/java/javaee/overview/index.html)应用服务器所提供的对应类完全兼容。Java EE 标准在 `/WEB-INF/web.xml` 文件中提供了一系列标准元素，用来引用或定义资源。  

可通过下列规范了解如何编写针对 JNDI 的 API 以及 Java 企业版（Java EE）服务器所支持的功能，这也是 Tomcat 针对其所提供的服务而仿效的功能。  

- [Java 命名与目录接口](http://docs.oracle.com/javase/7/docs/technotes/guides/jndi/index.html)（包括在 JDK 1.4 或更前的版本） 
- [Java EE 平台规范](http://www.oracle.com/technetwork/java/javaee/documentation/index.html)，查看其中的第5章：*Naming*（命名）    


## web.xml 配置    

可在 Web 应用的部署描述符文件（`/WEB-INF/web.xml`）中使用下列元素来定义资源：     

- `<env-entry>` 应用的环境项。一个可用于配置应用运行方式的单值参数。  
- `<resource-ref>` 资源引用，通常是引用保存某种资源的对象工厂，比如 JDBC `DataSource` 或 JavaMail `Session` 这样的资源；或者引用配置在 Tomcat 中的自定义对象工厂中的资源。     
- `<resource-env-ref>` 资源环境引用。Servlet 2.4 所添加的一种新 `resource-ref`，它简化了不需要认证消息的资源的配置。     

有了这些，Tomcat就能利用适宜的资源工厂来创建资源，再也不需要其他配置信息了。Tomcat 将使用 `/WEB-INF/web.xml` 中的信息来创建资源。  

另外，Tomcat 还提供了一些用于 JNDI 的特殊选项，它们没有指定在 web.xml 中。比如，其中包括的 `closeMethod` 能在 Web 应用停止时，迅速清除 JNDI 资源；`singleton` 控制是否会在每次 JNDI 查找时创建资源的新实例。要想使用这些配置选项，资源必须指定在 Web 应用的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素内，或者位于 `$CATALINA_BASE/conf/server.xml` 的 [`<GlobalNamingResources>`](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 元素中。




## context.xml 配置  

如果 Tomcat 无法确定合适的资源工厂，并且/或者需要额外的配置信息，就必须在 Tomcat 创建资源之前指定好额外的具体配置。Tomcat 特定资源配置应位于 `<Context>` 元素内，它可以指定在 `$CATALINA_BASE/conf/server.xml`，或者，最好放在每个 Web 应用的上下文 XML 文件中（`META-INF/context.xml`）。   

要想完成 Tomcat 的特定资源配置，需要使用 `<Context>` 元素中的下列元素：  

- [`<Environment>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Environment_Entries) 对将通过 JNDI 的 `InitialContext` 方法暴露给 Web 应用的环境项的名称与数值加以配置（等同于 Web 应用部署描述符文件中包含了一个 `<env-entry>` 元素）。  
  
- [`<Resource>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Definitions) 定义应用所能用到的资源名称和数据类型（等同于 Web 应用部署描述符文件中包含了一个 `<resource-ref>` 元素）。  

- [`<ResourceLink>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Links) 添加一个链接，使其指向全局 JNDI 上下文中定义的资源。使用资源链接可以使 Web 应用访问定义在 [`<Server>`](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html) 元素中子元素 [`<GlobalNamingResources>`](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 中的资源。  

- [`<Transaction>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Transaction) 添加一个资源工厂，用于对从 `java:comp/UserTransaction` 获得的 UserTransaction 接口进行实例化。     

以上这些元素内嵌于 `<Context>` 元素中，而且是与特定应用相关联的。

如果资源已经定义在 `<Context>` 元素中，那就不必再在部署描述符文件中定义它了。但建议在部署描述符文件中保留相关项，以便记录应用资源需求。   

加入同样一个资源名称既被定义在 Web 应用部署描述符文件的 `<env-entry>` 元素中，又被定义在 Web 应用的 `<Context>` 元素的 `<Environment>` 元素内，那么**只有**当相应的 `<Environment>` 元素允许时（将其中的 `override` 属性设为 `true`），部署描述符文件中的值**才**会优先对待。    


## 全局配置  

Tomcat 为整个服务器维护着一个全局资源的独立命名空间。这些全局资源配置在 `$CATALINA_BASE/conf/server.xml` 的 [`<GlobalNamingResources>`](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 元素内。可以使用 [`<ResourceLink>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Links)将这些资源暴露给 Web 应用，以便在每一应用上下文中将其包含进来。    


如果资源已经定义在 `<Context>` 元素中，那就不必再在部署描述符文件中定义它了。但建议在部署描述符文件中保留相关项，以便记录应用资源需求。  

## 使用资源   

当 Web 应用最初部署时，就配置 `InitialContext`，使其可被 Web 应用的各组件所使用（只读访问）。JNDI 命名空间的 `java:comp/env` 部分中包含着所有的配置项与资源，所以访问资源（在下例中，就是一个JDBC 数据源）应按如下形式进行：   

```
// 获取环境命名上下文
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");

// 查找数据源
DataSource ds = (DataSource)
  envCtx.lookup("jdbc/EmployeeDB");

// 分配并使用池中的连接
Connection conn = ds.getConnection();
... use this connection to access the database ...
conn.close();

```






## Tomcat 标准资源工厂  

Tomcat 包含一系列资源工厂，能为 Web 应用提供各种服务，而且无需修改 Web 应用或部署描述符文件即能灵活配置（通过  `<Context>` 元素）。下面所列出的每一小节都详细介绍了标准资源工厂的配置与用途。  

要想了解如何创建、安装、配置和使用你自己的自定义资源工厂类，请参看[添加自定义资源工厂](http://tomcat.apache.org/tomcat-8.0-doc/jndi-resources-howto.html#Adding_Custom_Resource_Factories)。   

**注意**：在标准资源工厂中，只有“JDBC DataSource”和“User Transaction”工厂可适用于其他平台，而且这些平台必须实现了 Java EE 规范。而其他所有标准资源工厂，以及你自己编写的自定义资源工厂，则都是 Tomcat 所专属的，不适用于其他容器。   

### 一般 JavaBean 资源  

####0. 简介  

该资源工厂能创建出**任何**符合标准 JavaBean 命名规范<sup>1</sup>的 Java 类的对象。如果工厂的 `singleton` 属性被设为 `false`，那么每当对该项进行 `lookup` 时，资源工厂将会创建出适合的 bean 类的新实例。

<sup>1. 标准的 JavaBean 命名规范，比如：构造函数没有任何参数，属性设置器遵守 `setFoo()` 命名模式，等等。</sup>

使用该功能所需的步骤将在下文介绍。   

####1. 创建 JavaBean 类   

创建一个 JavaBean 类，在每次查找资源工厂时，就创建它的实例。比如，假设你创建了一个名叫 `com.mycompany.MyBean` 的 JavaBean 类，如下所示：  

``` 
package com.mycompany;

public class MyBean {

  private String foo = "Default Foo";

  public String getFoo() {
    return (this.foo);
  }

  public void setFoo(String foo) {
    this.foo = foo;
  }

  private int bar = 0;

  public int getBar() {
    return (this.bar);
  }

  public void setBar(int bar) {
    this.bar = bar;
  }


}

```


####2. 声明资源需求  

接下来，修改 Web 应用部署描述符文件（`/WEB-INF/web.xml`），声明 JNDI 名称，并据此请求该 Bean 类的新实例。最简单的方法是使用 `<resource-env-ref>` 元素，如下所示：  

``` 
<resource-env-ref>
  <description>
    Object factory for MyBean instances.
  </description>
  <resource-env-ref-name>
    bean/MyBeanFactory
  </resource-env-ref-name>
  <resource-env-ref-type>
    com.mycompany.MyBean
  </resource-env-ref-type>
</resource-env-ref>

```


**警告**：一定要遵从 Web 应用部署描述符文件中 DTD 所需要的元素顺序。关于这点，可参看[Servlet 规范](http://wiki.apache.org/tomcat/Specifications)中的解释。   

####3. 使用资源

资源引用的典型用例如下所示：     

``` 
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");
MyBean bean = (MyBean) envCtx.lookup("bean/MyBeanFactory");

writer.println("foo = " + bean.getFoo() + ", bar = " +
               bean.getBar());

```


####4. 配置 Tomcat 资源工厂  

为了配置 Tomcat 资源工厂，为 Web 应用的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html)元素添加下列元素：   

``` 
<Context ...>
  ...
  <Resource name="bean/MyBeanFactory" auth="Container"
            type="com.mycompany.MyBean"
            factory="org.apache.naming.factory.BeanFactory"
            bar="23"/>
  ...
</Context>

```


注意这里的资源名称，这里 `bean/MyBeanFactory` 必须跟部署描述符文件中所指定的值完全一样。这里还初始化了 `bar` 属性值，从而当返回新的 bean 时，`setBar(23)` 就会被调用。由于我们没有初始化 `foo` 属性（虽然我们完全可以这么做），所以 bean 依然采用构造函数中设置的默认值。     


假设我们的 Bean 如下所示：   

```   
package com.mycompany;

import java.net.InetAddress;
import java.net.UnknownHostException;

public class MyBean2 {

  private InetAddress local = null;

  public InetAddress getLocal() {
    return local;
  }

  public void setLocal(InetAddress ip) {
    local = ip;
  }

  public void setLocal(String localHost) {
    try {
      local = InetAddress.getByName(localHost);
    } catch (UnknownHostException ex) {
    }
  }

  private InetAddress remote = null;

  public InetAddress getRemote() {
    return remote;
  }

  public void setRemote(InetAddress ip) {
    remote = ip;
  }

  public void host(String remoteHost) {
    try {
      remote = InetAddress.getByName(remoteHost);
    } catch (UnknownHostException ex) {
    }
  }

}


```


该 Bean 有两个 `InetAddress` 类型的属性。第一个属性 `local` 还有第二种 setter 方法，传入的是一个字符串参数。默认 Tomcat BeanFactory 会使用自动侦测到的 setter 方法，并将其参数类型作为属性类型，然后抛出一个 NamingException（命名异常），因为它还没有准备好将给定字符串值转化为 `InetAddress`。我们可以让 Tomcat BeanFactory 使用其他的 setter 方法，如下所示：    

```   

<Context ...>
  ...
  <Resource name="bean/MyBeanFactory" auth="Container"
            type="com.mycompany.MyBean2"
            factory="org.apache.naming.factory.BeanFactory"
            forceString="local"
            local="localhost"/>
  ...
</Context>


```

bean 属性 `remote` 也可以从字符串中设置，但必须使用非标准方法 `host`。如下设置 `local` 和 `remote`：     

```
<Context ...>
  ...
  <Resource name="bean/MyBeanFactory" auth="Container"
            type="com.mycompany.MyBean2"
            factory="org.apache.naming.factory.BeanFactory"
            forceString="local,remote=host"
            local="localhost"
            remote="tomcat.apache.org"/>
  ...
</Context>

```


如上所示，可以利用逗号作分隔符，将多个属性描述串联在一起放在 `forceString` 中。每一属性描述要么只包含属性名，要么由 `name = method` 的结构所组成。对于前者的情况，BeanFactory 会直接调用属性名的 setter 方法；而对于后者，则通过调用方法 `method` 来设置属性名 `name`。对于 `String` 或基本类型，或者相应的基本包装器类的属性，不必使用 `forceString`。会自动侦测正确的 setter 并实施参数类型转换。      



### UserDatabase 资源       

#### 0. 简介  

UserDatabase 资源通常被配置成通过 UserDataBase Realm 所使用的全局资源。Tomcat 包含一个 UserDatabaseFactoory，能够创建基于 XML 文件（通常是 `tomcat-users.xml`）的 UserDatabase 资源。   

建立全局的 UserDataBase 资源的步骤如下。     

#### 1. 创建/编辑 XML 文件    

XML 文件通常位于 `$CATALINA_BASE/conf/tomcat-users.xml`，但也可以放在文件系统中的任何位置。我们建议把该文件放在 `$CATALINA_BASE/conf`。典型的 XML 应如下所示：   

```
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
  <role rolename="tomcat"/>
  <role rolename="role1"/>
  <user username="tomcat" password="tomcat" roles="tomcat"/>
  <user username="both" password="tomcat" roles="tomcat,role1"/>
  <user username="role1" password="tomcat" roles="role1"/>
</tomcat-users>

```    

#### 2. 声明资源     

接下来，修改 `$CATALINA_BASE/conf/server.xml` 来创建基于此文件的 UserDataBase 资源。如下所示：  

```
<Resource name="UserDatabase"
          auth="Container"
          type="org.apache.catalina.UserDatabase"
          description="User database that can be updated and saved"
          factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
          pathname="conf/tomcat-users.xml"
          readonly="false" />

```  

属性 `pathname` 可以采用绝对路径或相对路径。相对路径意味着是相对于 `$CATALINA_BASE`。  

`readonly` 属性是可选属性，如果不采用，则默认为 `true`。如果该 XML 文件可写，那么当 Tomcat 开启时，就会被修改。**警告**：当该文件被修改后，它会继承 Tomcat 目前运行用户的默认文件权限。所以要确保这样做是否能保持应用的安全性。     



#### 3. 配置 Realm    

配置 UserDatabase Realm 以便使用该资源，详情可参看 [Realm 配置文档](》连接至已翻译好的页面》)  



### JavaMail 会话  


#### 0. 简介  

很多 Web 应用都会把发送电子邮件作为系统的必备功能。[JavaMail](http://www.oracle.com/technetwork/java/javamail/index.html) API 可以让这一过程变得相对简单些，但需要很多的配置细节，客户端应用必须知道的（包括用于发送消息的 SMTP 主机的名称）。  

Tomcat 所包含的标准资源工厂可以为你创建 `javax.mail.Session` 会话实例，并且已经配置好连接到 SMTP 服务器上，从而使应用完全与电子邮件配置环境相隔离，不受后者变更的影响，无论何时，只需请求并接受预配置的会话即可。  

所需步骤如下所示。

#### 1. 声明资源需求    

首先应该做的是修改 Web 应用的部署描述符文件（`/WEB-INF/web.xml`），声明 JNDI 名称以便借此查找预配置会话。按照惯例，所有这样的名字都应该解析到 `mail` 子上下文（相对于标准的 `java:comp/env` 命名上下文而言的，这个命名上下文是所有资源工厂的基准。）典型的 `web.xml` 项应该如下所示：   

```
<resource-ref>
  <description>
    Resource reference to a factory for javax.mail.Session
    instances that may be used for sending electronic mail
    messages, preconfigured to connect to the appropriate
    SMTP server.
  </description>
  <res-ref-name>
    mail/Session
  </res-ref-name>
  <res-type>
    javax.mail.Session
  </res-type>
  <res-auth>
    Container
  </res-auth>
</resource-ref>
  
```

**警告**：一定要遵从 Web 应用部署描述符文件中 DTD 所需要的元素顺序。关于这点，可参看[Servlet 规范](http://wiki.apache.org/tomcat/Specifications)中的解释。   

#### 2. 使用资源    

资源引用的典型用例如下所示：      

```
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");
Session session = (Session) envCtx.lookup("mail/Session");

Message message = new MimeMessage(session);
message.setFrom(new InternetAddress(request.getParameter("from")));
InternetAddress to[] = new InternetAddress[1];
to[0] = new InternetAddress(request.getParameter("to"));
message.setRecipients(Message.RecipientType.TO, to);
message.setSubject(request.getParameter("subject"));
message.setContent(request.getParameter("content"), "text/plain");
Transport.send(message);

```  

注意，该应用所用的资源引用名与 Web 应用部署符中声明的完全相同。这是与下文会讲到的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素里所配置的资源工厂相匹配的。


#### 3. 配置 Tomcat 资源工厂    

为了配置 Tomcat 的资源工厂，在 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素中添加以下元素：  

```
<Context ...>
  ...
  <Resource name="mail/Session" auth="Container"
            type="javax.mail.Session"
            mail.smtp.host="localhost"/>
  ...
</Context>

```  

注意，资源名（在这里，是 `mail/Session`）必须与 Web 应用部署描述符文件中所指定的值相匹配。对于 `mail.smtp.host` 参数值，可以用为你的网络提供 SMTP 服务的服务器来自定义。   

额外的资源属性与值将转换成相关的属性及值，并被传入 `javax.mail.Session.getInstance(java.util.Properties)`，作为参数集 `java.util.Properties` 中的一部分。除了 JavaMail 规范附件A中所定义的属性之外，个别的提供者可能还支持额外的属性。

如果资源配置中包含 `password` 属性，以及 `mail.smtp.user` 或 `mail.user` 属性，那么 Tomcat 资源工厂将配置并添加 `javax.mail.Authenticator` 到邮件会话中。  


#### 4. 安装 JavaMail 库    

[下载 JavaMail API](https://java.net/projects/javamail/pages/Home)   

解压缩文件分发包，将 mail.jar 放到 $CATALINA_HOME/lib 中，从而使 Tomcat 能在邮件会话资源初始化期间能够使用它。**注意**：不能将这一文件同时放在 $CATALINA_HOME/lib 和 Web 应用的 /lib 文件夹中，否则就会出错，只能将其放在 $CATALINA_HOME/lib 中。  

#### 5. 重启 Tomcat    

为了能让 Tomcat 使用这个额外的 jar 文件，必须重启 Tomcat 实例。    

#### 范例应用    

Tomcat 中的 `/examples` 应用中带有一个使用该资源工厂的范例。可以通过“JSP 范例”的链接来访问它。实际发送邮件的 servlet 的源代码则位于 `/WEB-INF/classes/SendMailServlet.java` 中。  


**警告**：默认配置在 `localhost` 的 端口 25 上的 SMTP 服务器。如果实际情况不符，则需要编辑该 Web 应用的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素，将 `mail.smtp.host` 参数的值修改为你的网络上的 SMTP 服务器的主机名。   



### JDBC 数据源     

#### 0. 简介   

许多 Web 应用都需要 JDBC 驱动来访问数据库，以便能够支持该应用所需要的功能。Java EE 平台规范要求 Java EE 应用服务器针对该需求提供一个 DataSource 实现（也就是说，用于 JDBC 连接的连接池）。Tomcat 就能提供同样的支持，因此在 Tomcat 上，由于使用了这种服务，基于数据库的应用可以不用修改就能移植到任何 Java EE 服务器上运行。  

要想详细了解 JDBC，可以参考以下网站或信息来源：   

- [http://www.oracle.com/technetwork/java/javase/jdbc/index.html](http://www.oracle.com/technetwork/java/javase/jdbc/index.html) 一个 Java 数据库连接性能相关信息网站。   
- [http://java.sun.com/j2se/1.3/docs/guide/jdbc/spec2/jdbc2.1.frame.html](http://java.sun.com/j2se/1.3/docs/guide/jdbc/spec2/jdbc2.1.frame.html) JDBC 2.1 API 规范。   
- [http://java.sun.com/products/jdbc/jdbc20.stdext.pdf](http://java.sun.com/products/jdbc/jdbc20.stdext.pdf) JDBC 2.0 标准扩展 API（包括 `javax.sql.DataSource` API）。该包现在被称为“JDBC 可选包”。   
- [http://www.oracle.com/technetwork/java/javaee/overview/index.htm](http://www.oracle.com/technetwork/java/javaee/overview/index.htm) Java EE 平台规范（介绍了所有 Java EE 平台必须为应用提供的 JDBC 附加功能）。    

**注意**：Tomcat 默认所支持的数据源是基于 [Commons 项目](http://commons.apache.org) 的 **DBCP** 连接池。但也可以通过编写自定义的资源工厂，使用其他实现了 `javax.sql.DataSource` 的连接池，详见<a href="#AddCustomResFactories">下文</a>。  
   

#### 1. 安装 JDBC 驱动     

使用 JDBC 数据源的 JNDI 资源工厂需要一个适合的 JDBC 驱动，要求它既能被 Tomcat 内部类所使用，也能被你的 Web 应用所使用。这很容易实现，只需将驱动的 JAR 文件（或多个文件）安装到 `$CATALINA_HOME/lib` 目录中即可，这样资源工厂和应用就都能使用了这一驱动了。



#### 2. 声明资源需求    

下一步，修改 Web 应用的部署描述符文件（`/WEB-INF/web.xml`），声明 JNDI 名称以便借此查找预配置的数据源。按照惯例，所有这样的名称都应该在`jdbc` 子上下文中声明（这个“子”是相对于标准的 `java:comp/env` 环境命名上下文而言的。`java:comp/env` 环境命名上下文是所有资源工厂的根引用）。典型的 `web.xml` 文件应如下所示：   

```
<resource-ref>
  <description>
    Resource reference to a factory for java.sql.Connection
    instances that may be used for talking to a particular
    database that is configured in the <Context>
    configurartion for the web application.
  </description>
  <res-ref-name>
    jdbc/EmployeeDB
  </res-ref-name>
  <res-type>
    javax.sql.DataSource
  </res-type>
  <res-auth>
    Container
  </res-auth>
</resource-ref>

```

**警告**：一定要遵从 Web 应用部署描述符文件中 DTD 所需要的元素顺序。关于这点，可参看[Servlet 规范](http://wiki.apache.org/tomcat/Specifications)中的解释。   




#### 3. 使用资源    

资源引用的典型用例如下所示：     

```   
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");
DataSource ds = (DataSource)
  envCtx.lookup("jdbc/EmployeeDB");

Connection conn = ds.getConnection();
... use this connection to access the database ...
conn.close();


```

注意，该应用所用的资源引用名与 Web 应用部署符中声明的完全相同。这是与下文会讲到的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素里所配置的资源工厂相匹配的。

#### 4. 配置 Tomcat 资源工厂    

为了配置 Tomcat 的资源工厂，在 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素中添加以下元素：  

```
<Context ...>
  ...
  <Resource name="jdbc/EmployeeDB"
            auth="Container"
            type="javax.sql.DataSource"
            username="dbusername"
            password="dbpassword"
            driverClassName="org.hsql.jdbcDriver"
            url="jdbc:HypersonicSQL:database"
            maxTotal="8"
            maxIdle="4"/>
  ...
</Context>

```  


注意上述代码中的资源名（这里是 `jdbc/EmployeeDB`）必须跟 Web 应用部署描述符文件中指定的值相同。   

该例假定使用的是 HypersonicSQL 数据库 JDBC 驱动。可自定义 `driverClassName` 和 `driverName` 参数，使其匹配实际数据库的 JDBC 驱动与连接 URL。   

Tomcat 标准数据源资源工厂（`org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory`）的配置属性如下：  

- **driverClassName** 所用的 JDBC 驱动的完全合格的类名。  
- **username** JDBC 驱动所要接受的数据库用户名。  
- **password** JDBC 驱动所要接受的数据库密码。   
- **url** 传入 JDBC 驱动的连接 URL（为了向后兼容性考虑，也同样认可 `driverName` 属性，即与之等同）。  
- **initialSize** 连接池初始化过程中所创建的初始连接的数目。默认为 0。  
- **maxTotal** 连接池同时所能分配的最大连接数。默认为 8。   
- **minIdle** 连接池中同时空闲的最少连接数。默认为 0。   
- **maxIdle** 连接池中同时空闲的最多连接数。默认为 8。     
- **maxWaitMillis** 在抛出异常前，连接池等待（没有可用的连接）连接返回的最长等待毫秒数。默认为 -1（无限长时间）。  

还有一些额外的用来验证连接的属性，如下所示：   

- **validationQuery** 在连接返回应用之前，连接池用于验证连接的 SQL 查询。如果指定了该属性值，查询必须是一个至少能返回一行的 SQL SELECT 语句。  
- **validationQueryTimeout** 验证查询返回的超时时间。默认为 -1（无限长时间）。  
- **testOnBorrow** 布尔值，true 或 false，针对每次从连接池中借出的连接，判断是否应用验证查询对其验证。默认：true。   
- **testOnReturn** 布尔值，true 或 false，针对每次归还给连接池的连接，判断是否应用验证查询对其验证。默认：false。   

可选的 evictor thread 会清除空闲较长时间的连接，从而缩小连接池。evictor thread 不受 `minIdle` 属性值的空闲。注意，如果你只想通过配置的 `minIdle` 属性来缩小连接池，那么不需要使用 evictor thread。     


默认 evictor 是禁用的，另外，可以使用下列属性来配置它：      

- `timeBetweenEvictionRunsMillis` evictor 线程连续运行之间的毫秒数。默认为 -1（禁止）  
- `numTestsPerEvictionRun` evictor 每次运行中，evictor 实施的用来检测空闲与否的连接数目。默认为 3。
- `minEvictableIdleTimeMillis` evictor 从连接池中清除某连接后的空闲时间，以毫秒计，默认为 30 \* 60 \* 1000（30分钟）。
- `testWhileIdle` 布尔值，true 或 false。对于在连接池中处于空闲状态的连接，是否应被 evictor 线程通过验证查询来验证。默认为false。  


另一个可选特性是对废弃连接的移除。如果应用很久都不把某个连接返回给连接池，那么该连接就被称为废弃连接。连接池就会自动关闭这样的连接，并将其从池中移除。这么做是为了防止应用泄露连接。    

默认是禁止废弃连接的，可以通过下列属性来配置：   

- **removeAbandoned** 布尔值，true 或 false。确定是否去除连接池中的废弃连接。默认为 false。  
- **removeAbandonedTimeout** 经过多少秒后，借用的连接可以认为被废弃。默认为 300。
- **logAbandoned** 布尔值，true 或 false。确定是否需要针对废弃了语句或连接的应用代码来记录堆栈跟踪。如果记录的话，将会带来很大的开销。默认为 false。    

最后再介绍一些可以对连接池行为进行进一步微调的属性：   

- **defaultAutoCommit** 布尔值，`true` 或 `false`。由连接池所创建的连接的默认自动提交状态。默认为 true。  
- **defaultReadOnly** 布尔值，`true` 或 `false`。由连接池所创建的连接的默认只读状态。默认为 false。  
- **defaultTransactionIsolation** 设定默认的事务隔离级别。可取值为：`NONE`、`READ_COMMITTED`、`READ_UNCOMMITTED`、`REPEATABLE_READ` 与 `SERIALIZABLE`。没有默认设置。   
- **poolPreparedStatements** 布尔值，true 或 false。 是否池化 PreparedStatements 和 CallableStatements。默认为 `false`。
- **maxOpenPreparedStatements** 同时能被语句池分配的开放语句的最大数目。默认为 -1（无限）   
- **defaultCatalog**  catalog 默认值。默认值：未设定。
- **connectionInitSqls** 连接建立后运行的一系列 SQL 语句。各个语句间用分号（`;`）进行分隔。默认为：没有语句。  
- **connectionInitSqls** 传入驱动用于创建连接的驱动特定属性。每一属性都以 `name = value` 的形式给出，多个属性用分号（`;`）进行分隔。默认：没有属性。   
- **accessToUnderlyingConnectionAllowed** 布尔值，`true` 或 `false`。 是否可访问底层连接。默认为 `false`。   


要想更详细地了解这些属性，请参阅 commons-dbcp 文档。   





### <a name ="AddCustomResFactories">添加自定义资源工厂</a>    

如果标准资源工厂无法满足你的需求，你还可以自己编写资源工厂，然后将其集成到 Tomcat 中，在 Web 应用的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素中配置该工厂的使用方式。在下面的范例中，我们将创建一个资源工厂，只懂得如何 `com.mycompany.MyBean` bean


#### 1. 编写资源工厂类    

你必须编写一个类来实现 JNDI 服务提供者 `javax.naming.spi.ObjectFactory` 接口。每次 Web 应用在绑定到该工厂（假设该工厂配置中，`singleton = "false"`）的上下文项上调用 `lookup()` 时，就会调用 `getObjectInstance()` 方法，该方法有如下这些参数：   

- `Object obj`   


创建一个能够生成 `MyBean` 实例的资源工厂，需要像下面这样来创建类：   

```
package com.mycompany;

import java.util.Enumeration;
import java.util.Hashtable;
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.NamingException;
import javax.naming.RefAddr;
import javax.naming.Reference;
import javax.naming.spi.ObjectFactory;

public class MyBeanFactory implements ObjectFactory {

  public Object getObjectInstance(Object obj,
      Name name, Context nameCtx, Hashtable environment)
      throws NamingException {

      // Acquire an instance of our specified bean class
      MyBean bean = new MyBean();

      // Customize the bean properties from our attributes
      Reference ref = (Reference) obj;
      Enumeration addrs = ref.getAll();
      while (addrs.hasMoreElements()) {
          RefAddr addr = (RefAddr) addrs.nextElement();
          String name = addr.getType();
          String value = (String) addr.getContent();
          if (name.equals("foo")) {
              bean.setFoo(value);
          } else if (name.equals("bar")) {
              try {
                  bean.setBar(Integer.parseInt(value));
              } catch (NumberFormatException e) {
                  throw new NamingException("Invalid 'bar' value " + value);
              }
          }
      }

      // Return the customized instance
      return (bean);

  }

}

```

> // Acquire an instance of our specified bean class 需要我们所指定的bean 类的一个实例
> // Customize the bean properties from our attributes 从属性中自定义 bean 属性。  
> // Return the customized instance 返回自定义实例 

在上例中，无条件地创建了 `com.mycompany.MyBean` 类的一个新实例， 并根据工厂配置中的 `<ResourceParams>` 元素（下文详述）包括的参数来填充这一实例。你应该记住，必须忽略任何名为 `factory` 的参数——参数应该用来指定工厂类自身的名字（`com.mycompany.MyBeanFactory`），而不是配置的 bean 属性。  

关于 `ObjectFactory` 的更多信息，可参见 [JNDI 服务提供者接口（SPI）规范](http://docs.oracle.com/javase/7/docs/technotes/guides/jndi/index.html)。   

首先参照一个 `$CATALINA_HOME/lib` 目录中包含所有 JAR 文件的类路径来编译该类。完成之后，将这个工厂类以及相应的 Bean 类解压缩到 `$CATALINA_HOME/lib`，或者 `$CATALINA_HOME/lib` 内的一个 JAR 文件中。这样，所需的类文件就能被 Catalina 内部资源与 Web 应用看到了。

#### 2. 声明资源需求  

下一步，修改 Web 应用的部署描述符文件（`/WEB-INF/web.xml`），声明 JNDI 名称以便借此请求该 bean 的新实例。最简单的方法是使用 `<resource-env-ref>` 元素，如下所示：  

```  
<resource-env-ref>
  <description>
    Object factory for MyBean instances.
  </description>
  <resource-env-ref-name>
    bean/MyBeanFactory
  </resource-env-ref-name>
  <resource-env-ref-type>
    com.mycompany.MyBean
  </resource-env-ref-type>
<resource-env-ref>


```

**警告**：一定要遵从 Web 应用部署描述符文件中 DTD 所需要的元素顺序。关于这点，可参看[Servlet 规范](http://wiki.apache.org/tomcat/Specifications)中的解释。   

#### 3. 使用资源      

资源引用的典型用例如下所示：     

``` 
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");
MyBean bean = (MyBean) envCtx.lookup("bean/MyBeanFactory");

writer.println("foo = " + bean.getFoo() + ", bar = " +
               bean.getBar());

```




#### 4. 配置 Tomcat 资源工厂  

为了配置 Tomcat 的资源工厂，在 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素中添加以下元素：  

```
<Context ...>
  ...
  <Resource name="bean/MyBeanFactory" auth="Container"
            type="com.mycompany.MyBean"
            factory="com.mycompany.MyBeanFactory"
            singleton="false"
            bar="23"/>
  ...
</Context>

```  

注意上述代码中的资源名（这里是 `bean/MyBeanFactory`）必须跟 Web 应用部署描述符文件中指定的值相同。另外，我们还初始化了 `bar` 属性值，从而在返回新 bean 时，导致 `setBar(23)` 被调用。由于我们没有初始化 `foo` 属性（虽然完全可以这样做），所以 bean 将含有构造函数所定义的各种默认值。   

另外，你肯定能注意到，从应用开发者的角度来看，资源环境引用的声明，以及请求新实例的编程方式，都跟**通用 JavaBean 资源**（Generic JavaBean Resources）范例所用方式如出一辙。这揭示了使用 JNDI 资源封装功能的一个优点：只要维持兼容的 API，无需修改使用资源的应用，只需改变底层实现。   
  
