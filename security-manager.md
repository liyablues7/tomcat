# Security Manager HOW-TO

## 目录   

- 背景知识  
- 权限  
	1. 标准权限  
	2. Tomcat 自定义权限  
- 利用 SecurityManager 配置 Tomcat  
- 配置 Tomcat 中的包保护
- 疑难解答

## 背景知识  

Java 的 **SecurityManager** 能让 Web 浏览器在它自身的沙盒中运行小型应用（applet），从而具有防止不可信代码访问本地文件系统的文件以及防止其连接到主机，而不是加载该应用的位置，等等。如同 SecurityManager 能防止不可信的小型应用在你的浏览器上运行，运行 Tomcat 时，使用 SecurityManager 也能保护服务器，使其免受木马型的 applet、JSP、JSP Bean以及标签库的侵害，甚至也可以防止由于无意中的疏忽所造成的问题。  

假设网站有一位经授权可发布 JSP 的用户，他在无意中将下面这些代码加入了 JSP 中： 

`<% System.exit(1); %>`  

每当 Tomcat 执行这个 JSP 文件时，Tomcat 都会退出。Java 的 SecurityManager 构成了系统管理员保证服务器安全可靠的另一道防线。  

**警告**：使用 Tomcat 代码库时会执行一个安全审核。大多数关键包已受到保护，新的安全包保护机制已经实施。然而，在允许不可信用户发布 Web 应用、JSP、servlet、bean或标签库之前，你仍要反复确定自己配置的 SecurityManager 是否满足了要求。**但不管怎么说，利用 SecurityManager 来运行 Tomcat 肯定比没有它好得多**  


## Permissions    

权限类用于定义 Tomcat 加载的类所具有的权限。标准 JDK 中包含了很多标准权限类，你还可以针对自己的 Web 应用自定义权限类。Tomcat 支持这两种技术。

### 标准权限    

关于适用于 Tomcat 的标准系统 SecurityManager 权限类，以下仅是一个简短的总结。详情请查看[http://docs.oracle.com/javase/7/docs/technotes/guides/security/](http://docs.oracle.com/javase/7/docs/technotes/guides/security/)。  

- **java.util.PropertyPermission**——控制对 JVM 属性的读/写，比如说 `java.home`。
- **java.lang.RuntimePermission**——控制一些系统/运行时函数的使用，比如 `exit()` 和 `exec()`。 另外也控制包的访问/定义。
- **java.io.FilePermission**——控制对文件和目录的读/写/执行。
- **java.net.SocketPermission**——控制网络套接字的使用。
- **java.net.NetPermission**——控制组播网络连接的使用。
- **java.lang.reflect.ReflectPermission**——控制类反射的使用。
- **java.security.SecurityPermission**——控制对 Security 方法的访问。
- **java.security.AllPermission**——允许访问任何权限，仿佛没有 SecurityManager。


### Tomcat 自定义权限      

Tomcat 使用了一个自定义权限类 **org.apache.naming.JndiPermission**。该权限能够控制对 JNDI 命名的基于文件的资源的可读访问。权限名就是 JNDI 名，无任何行为。后面的 `*` 可以用来在授权时进行模糊匹配。比如，可以在策略文件中加入以下内容：   

`permission  org.apache.naming.JndiPermission  "jndi://localhost/examples/*";`  


从而为每个部署的 Web 应用动态生成这样的权限项，允许它们读取自己的静态资源，而不允许读取其他的文件（除非显式地赋予这些文件权限）。

另外，Tomcat 还能动态生成下面这样的文件权限。   

```
permission java.io.FilePermission "** your application context**", "read";

permission java.io.FilePermission
  "** application working directory**", "read,write";
permission java.io.FilePermission
  "** application working directory**/-", "read,write,delete";

```


***application working directory** 是部署应用所用的文件夹或 WAR 文件。 **application working directory** 是应 Servlet 规范需要而提供给应用的暂时性目录。  



## 利用 SecurityManager 配置 Tomcat   
### 策略文件格式    

Java SecurityManager 所实现的安全策略配置在 `$CATALINA_BASE/conf/catalina.policy` 文件中。该文件完全替代了 JDK
 系统目录中提供的 `java.policy` 文件。既可以手动编辑 `catalina.policy` 文件，也可以使用Java 1.2 或以后版本附带的 [policytool](http://docs.oracle.com/javase/6/docs/technotes/guides/security/PolicyGuide.html) 应用。   
 
`catalina.policy` 文件中的项使用标准的 `java.policy` 文件格式，如下所示：  

```  
// 策略文件项范例  

grant [signedBy <signer>,] [codeBase <code source>] {
  permission  <class>  [<name> [, <action list>]];
};

```

**signedBy** 和 **codeBase** 两项在授予权限时是可选项。注释行以 `//` 开始，在当前行结束。`codeBase` 以 URL 的形式。对于文件 URL，可以使用 `${java.home}` 与 `${catalina.home}`属性（这些属性代表的是使用 `JAVA_HOME`、`CATALINA_HOME` 和 `CATALINA_BASE` 环境变量为这些属性定义的目录路径）。  


### 默认策略文件    

默认的 `$CATALINA_BASE/conf/catalina.policy` 文件如下所示：  

```    

// Licensed to the Apache Software Foundation (ASF) under one or more
// contributor license agreements.  See the NOTICE file distributed with
// this work for additional information regarding copyright ownership.
// The ASF licenses this file to You under the Apache License, Version 2.0
// (the "License"); you may not use this file except in compliance with
// the License.  You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

// ============================================================================
// catalina.policy - Security Policy Permissions for Tomcat
//
// This file contains a default set of security policies to be enforced (by the
// JVM) when Catalina is executed with the "-security" option.  In addition
// to the permissions granted here, the following additional permissions are
// granted to each web application:
//
// * Read access to the web application's document root directory
// * Read, write and delete access to the web application's working directory
// ============================================================================


// ========== SYSTEM CODE PERMISSIONS =========================================


// These permissions apply to javac
grant codeBase "file:${java.home}/lib/-" {
        permission java.security.AllPermission;
};

// These permissions apply to all shared system extensions
grant codeBase "file:${java.home}/jre/lib/ext/-" {
        permission java.security.AllPermission;
};

// These permissions apply to javac when ${java.home] points at $JAVA_HOME/jre
grant codeBase "file:${java.home}/../lib/-" {
        permission java.security.AllPermission;
};

// These permissions apply to all shared system extensions when
// ${java.home} points at $JAVA_HOME/jre
grant codeBase "file:${java.home}/lib/ext/-" {
        permission java.security.AllPermission;
};


// ========== CATALINA CODE PERMISSIONS =======================================


// These permissions apply to the daemon code
grant codeBase "file:${catalina.home}/bin/commons-daemon.jar" {
        permission java.security.AllPermission;
};

// These permissions apply to the logging API
// Note: If tomcat-juli.jar is in ${catalina.base} and not in ${catalina.home},
// update this section accordingly.
//  grant codeBase "file:${catalina.base}/bin/tomcat-juli.jar" {..}
grant codeBase "file:${catalina.home}/bin/tomcat-juli.jar" {
        permission java.io.FilePermission
         "${java.home}${file.separator}lib${file.separator}logging.properties", "read";

        permission java.io.FilePermission
         "${catalina.base}${file.separator}conf${file.separator}logging.properties", "read";
        permission java.io.FilePermission
         "${catalina.base}${file.separator}logs", "read, write";
        permission java.io.FilePermission
         "${catalina.base}${file.separator}logs${file.separator}*", "read, write";

        permission java.lang.RuntimePermission "shutdownHooks";
        permission java.lang.RuntimePermission "getClassLoader";
        permission java.lang.RuntimePermission "setContextClassLoader";

        permission java.lang.management.ManagementPermission "monitor";

        permission java.util.logging.LoggingPermission "control";

        permission java.util.PropertyPermission "java.util.logging.config.class", "read";
        permission java.util.PropertyPermission "java.util.logging.config.file", "read";
        permission java.util.PropertyPermission "org.apache.juli.AsyncLoggerPollInterval", "read";
        permission java.util.PropertyPermission "org.apache.juli.AsyncMaxRecordCount", "read";
        permission java.util.PropertyPermission "org.apache.juli.AsyncOverflowDropType", "read";
        permission java.util.PropertyPermission "org.apache.juli.ClassLoaderLogManager.debug", "read";
        permission java.util.PropertyPermission "catalina.base", "read";

        // Note: To enable per context logging configuration, permit read access to
        // the appropriate file. Be sure that the logging configuration is
        // secure before enabling such access.
        // E.g. for the examples web application (uncomment and unwrap
        // the following to be on a single line):
        // permission java.io.FilePermission "${catalina.base}${file.separator}
        //  webapps${file.separator}examples${file.separator}WEB-INF
        //  ${file.separator}classes${file.separator}logging.properties", "read";
};

// These permissions apply to the server startup code
grant codeBase "file:${catalina.home}/bin/bootstrap.jar" {
        permission java.security.AllPermission;
};

// These permissions apply to the servlet API classes
// and those that are shared across all class loaders
// located in the "lib" directory
grant codeBase "file:${catalina.home}/lib/-" {
        permission java.security.AllPermission;
};


// If using a per instance lib directory, i.e. ${catalina.base}/lib,
// then the following permission will need to be uncommented
// grant codeBase "file:${catalina.base}/lib/-" {
//         permission java.security.AllPermission;
// };


// ========== WEB APPLICATION PERMISSIONS =====================================


// These permissions are granted by default to all web applications
// In addition, a web application will be given a read FilePermission
// for all files and directories in its document root.
grant {
    // Required for JNDI lookup of named JDBC DataSource's and
    // javamail named MimePart DataSource used to send mail
    permission java.util.PropertyPermission "java.home", "read";
    permission java.util.PropertyPermission "java.naming.*", "read";
    permission java.util.PropertyPermission "javax.sql.*", "read";

    // OS Specific properties to allow read access
    permission java.util.PropertyPermission "os.name", "read";
    permission java.util.PropertyPermission "os.version", "read";
    permission java.util.PropertyPermission "os.arch", "read";
    permission java.util.PropertyPermission "file.separator", "read";
    permission java.util.PropertyPermission "path.separator", "read";
    permission java.util.PropertyPermission "line.separator", "read";

    // JVM properties to allow read access
    permission java.util.PropertyPermission "java.version", "read";
    permission java.util.PropertyPermission "java.vendor", "read";
    permission java.util.PropertyPermission "java.vendor.url", "read";
    permission java.util.PropertyPermission "java.class.version", "read";
    permission java.util.PropertyPermission "java.specification.version", "read";
    permission java.util.PropertyPermission "java.specification.vendor", "read";
    permission java.util.PropertyPermission "java.specification.name", "read";

    permission java.util.PropertyPermission "java.vm.specification.version", "read";
    permission java.util.PropertyPermission "java.vm.specification.vendor", "read";
    permission java.util.PropertyPermission "java.vm.specification.name", "read";
    permission java.util.PropertyPermission "java.vm.version", "read";
    permission java.util.PropertyPermission "java.vm.vendor", "read";
    permission java.util.PropertyPermission "java.vm.name", "read";

    // Required for OpenJMX
    permission java.lang.RuntimePermission "getAttribute";

    // Allow read of JAXP compliant XML parser debug
    permission java.util.PropertyPermission "jaxp.debug", "read";

    // All JSPs need to be able to read this package
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.tomcat";

    // Precompiled JSPs need access to these packages.
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.jasper.el";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.jasper.runtime";
    permission java.lang.RuntimePermission
     "accessClassInPackage.org.apache.jasper.runtime.*";

    // Precompiled JSPs need access to these system properties.
    permission java.util.PropertyPermission
     "org.apache.jasper.runtime.BodyContentImpl.LIMIT_BUFFER", "read";
    permission java.util.PropertyPermission
     "org.apache.el.parser.COERCE_TO_ZERO", "read";

    // The cookie code needs these.
    permission java.util.PropertyPermission
     "org.apache.catalina.STRICT_SERVLET_COMPLIANCE", "read";
    permission java.util.PropertyPermission
     "org.apache.tomcat.util.http.ServerCookie.STRICT_NAMING", "read";
    permission java.util.PropertyPermission
     "org.apache.tomcat.util.http.ServerCookie.FWD_SLASH_IS_SEPARATOR", "read";

    // Applications using Comet need to be able to access this package
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.comet";

    // Applications using WebSocket need to be able to access these packages
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.tomcat.websocket";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.tomcat.websocket.server";
};


// The Manager application needs access to the following packages to support the
// session display functionality. These settings support the following
// configurations:
// - default CATALINA_HOME == CATALINA_BASE
// - CATALINA_HOME != CATALINA_BASE, per instance Manager in CATALINA_BASE
// - CATALINA_HOME != CATALINA_BASE, shared Manager in CATALINA_HOME
grant codeBase "file:${catalina.base}/webapps/manager/-" {
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.ha.session";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.manager";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.manager.util";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.util";
};
grant codeBase "file:${catalina.home}/webapps/manager/-" {
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.ha.session";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.manager";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.manager.util";
    permission java.lang.RuntimePermission "accessClassInPackage.org.apache.catalina.util";
};

// You can assign additional permissions to particular web applications by
// adding additional "grant" entries here, based on the code base for that
// application, /WEB-INF/classes/, or /WEB-INF/lib/ jar files.
//
// Different permissions can be granted to JSP pages, classes loaded from
// the /WEB-INF/classes/ directory, all jar files in the /WEB-INF/lib/
// directory, or even to individual jar files in the /WEB-INF/lib/ directory.
//
// For instance, assume that the standard "examples" application
// included a JDBC driver that needed to establish a network connection to the
// corresponding database and used the scrape taglib to get the weather from
// the NOAA web server.  You might create a "grant" entries like this:
//
// The permissions granted to the context root directory apply to JSP pages.
// grant codeBase "file:${catalina.base}/webapps/examples/-" {
//      permission java.net.SocketPermission "dbhost.mycompany.com:5432", "connect";
//      permission java.net.SocketPermission "*.noaa.gov:80", "connect";
// };
//
// The permissions granted to the context WEB-INF/classes directory
// grant codeBase "file:${catalina.base}/webapps/examples/WEB-INF/classes/-" {
// };
//
// The permission granted to your JDBC driver
// grant codeBase "jar:file:${catalina.base}/webapps/examples/WEB-INF/lib/driver.jar!/-" {
//      permission java.net.SocketPermission "dbhost.mycompany.com:5432", "connect";
// };
// The permission granted to the scrape taglib
// grant codeBase "jar:file:${catalina.base}/webapps/examples/WEB-INF/lib/scrape.jar!/-" {
//      permission java.net.SocketPermission "*.noaa.gov:80", "connect";
// };



```  


### 使用 SecurityManager 启动 Tomcat   

一旦配置好了用于 SecurityManager 的 `catalina.policy` 文件，就可以使用 `-security` 选项启动带有 SecurityManager 的Tomcat。  

```
$CATALINA_HOME/bin/catalina.sh start -security    (Unix)
%CATALINA_HOME%\bin\catalina start -security      (Windows)

```

## 配置 Tomcat 中的包保护    

从 Tomcat 5 开始，可以通过配置保护 Tomcat 内部包，使其免于被定义与访问。详情查看 [http://www.oracle.com/technetwork/java/seccodeguide-139067.html](http://www.oracle.com/technetwork/java/seccodeguide-139067.html)。  

**警告**：假如去除默认的包保护，可能会造成安全漏洞。  

### 默认属性文件    

默认的 `$CATALINA_BASE/conf/catalina.properties` 文件如下所示：  

```   
#
# List of comma-separated packages that start with or equal this string
# will cause a security exception to be thrown when
# passed to checkPackageAccess unless the
# corresponding RuntimePermission ("accessClassInPackage."+package) has
# been granted.
package.access=sun.,org.apache.catalina.,org.apache.coyote.,org.apache.tomcat.,
org.apache.jasper.
#
# List of comma-separated packages that start with or equal this string
# will cause a security exception to be thrown when
# passed to checkPackageDefinition unless the
# corresponding RuntimePermission ("defineClassInPackage."+package) has
# been granted.
#
# by default, no packages are restricted for definition, and none of
# the class loaders supplied with the JDK call checkPackageDefinition.
#
package.definition=sun.,java.,org.apache.catalina.,org.apache.coyote.,
org.apache.tomcat.,org.apache.jasper.


```  

一旦为 SecurityManager 配置了 `catalina.properties` 文件 ，记得重启 Tomcat。  

## 疑难解答  

假如应用执行一个由于缺乏所需权限而被禁止的操作，当 SecurityManager 侦测到这种违规时，就会抛出 `AccessControLException` 或 `SecurityException` 异常。虽然调试缺失的权限是很有难度的，但还是有一个办法，那就是将在执行中制定的所有安全决策的调试输出打开，这需要在启动 Tomcat 之前设置一个系统属性。最简单的方法就是通过 `CATALINA_OPTS` 环境变量来实现，命令如下所示：  

```   
export CATALINA_OPTS=-Djava.security.debug=all    (Unix)
set CATALINA_OPTS=-Djava.security.debug=all       (Windows)

```

记住，一定要在启动 Tomcat 之前去做。  

**警告**：这将生成很多兆的输出内容！但是，它能通过搜索关键字 `FAILED` 来锁定问题所在位置，确定需要检查的权限。此外，查阅 Java 安全文档可了解更多的可设置选项。   



 