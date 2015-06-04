# JDBC 数据源

## 目录  

- 概述    
- DriverManager，服务提供者机制以及内存泄露   
- 数据库连接池（DBCP 2）配置    
	1. 安装  
	2. 防止数据库连接池泄露   
	3. MySQL DBCP 范例    
	4. Oracle 8i、9i 与 10g    
	5. PostgreSQL  
- 非 DBCP 的解决方案   
- Oracle 8i 与 OCI 客户端  
	1. 简介  
	2. 连接起来  
- 常见问题   
	1. 数据库连接间歇性失败  
	2. 随机性的连接关闭异常  
	3. 上下文与全局命名资源    
	4. JNDI 资源命名和 Realm 交互     


## 概述  

JNDI 数据源配置的相关内容已经在 [JNDI 资源文档](连接至中文JNDI资源文档》》)中详细介绍过。但从 tomcat 用户的反馈意见来看，有些配置的细节问题非常棘手。   

针对常用的数据库，我们已经给 Tomcat 用户提供了一些配置范例，以及关于数据库使用的一些通用技巧。本章就将展示这些范例和技巧。   

另外，虽然有些注意事项来自于用户所提供的配置和反馈信息，但你可能也有不同的实践。如果经过试验，你发现某些配置可能具有广泛的助益作用，或者你觉得它们会使本章内容更加完善，请务必不吝赐教。  

**请注意，对比 Tomcat 7.x 和 Tomcat 8.x，JNDI 资源配置多少有些不同，这是因为使用的 Apache Commons DBCP 库的版本不同所致。**所以，为了在 Tomcat 8 中使用，你最好修改老版本的 JNDI 资源配置，以便能够匹配下文范例中的格式。详情可参看[Tomcat 迁移文档](http://tomcat.apache.org/migration.html)。   

另外还要提示的是，一般来说（特别是对于本教程而言），JNDI 数据源配置会假定你已经理解了 [Context](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 与 [Host](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html) 的配置偏好，其中包括在后者配置偏好中的应用自动部署的相关内容。


## DriverManager，服务提供者机制以及内存泄露  

`java.sql.DriverManager` 支持[服务提供者](http://docs.oracle.com/javase/6/docs/api/index.html?java/sql/DriverManager.html)机制。这项功能的实际作用在于：对于所有可用的 JDBC 驱动，只要它们声明提供 `META-INF/services/java.sql.Driver` 文件，就会被自动发现、加载并注册，从而减轻了我们在创建 JDBC 连接之前还需要显式地加载数据库驱动的负担。但在 servlet 容器环境的所有 Java 版本中，却根本没法实现这种功能。问题在于 `java.sql.DriverManager` 只会扫描一次驱动。     


Tomcat 自带的[阻止 JRE 内存泄漏侦听器](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html)可以在一定程度上解决这个问题，它会在 Tomcat 启动时触发驱动扫描。该侦听器默认是启用的。只有可见于该侦听器的库（比如 `$CATALINA_BASE/lib` 中的库）才能被数据库驱动所扫描。如果你想禁用该功能，那么一定要记住：首先使用 JDBC 的 Web 应用会触发扫描，从而当该应用重新加载时会出错；对于其他依赖该功能的 Web 应用来说也会导致出错。   

所以，假如应用的 `WEB-INF/lib` 目录中存在数据库驱动，那么这些应用就不能依赖服务提供者机制，而应该显式地注册驱动。  

`java.sql.DriverManager` 中的驱动已经被认为是内存泄露之源。当 Web 应用停止运行时，它所注册的任何驱动都必须重新注册。当 Web 应用停止运行时，Tomcat 会尝试自动寻找并重新注册任何由 Web 应用类加载器所加载的 JDBC 驱动。但最好是由应用通过 `ServletContextListener` 来实现这一点。  





## 数据库连接池（DBCP 2）配置   

Apache Tomcat 的默认数据库连接池实现基于的是 [Apache Commons](http://commons.apache.org) 项目的库，具体来说是这两个库：   

- Commons DBCP 
- Commons Pool   

这两个库都位于一个 JAR 文件中：`$CATALINA_HOME/lib/tomcat-dbcp.jar`。但该文件只包括连接池所需要的类，包名也已经改变了，以避免与应用冲突。     

DBCP 2.0 支持 JDBC 4.1。  

### 安装  

可参阅 [DBCP 文档](http://commons.apache.org/proper/commons-dbcp/configuration.html)了解完整的配置参数。  

### 防止数据库连接池泄露  

数据库连接池创建并管理着一些与数据库的连接。与打开新的连接相比，回收或重用现有的数据库连接要更为高效一些。   

连接池化还存在一个问题。Web 应用必须明确地关闭 ResultSet、Statement，以及 Connection。假如 Web 应用无法关闭这些资源时，会导致这些资源再也无法被重用，从而造成了数据库连接池“泄露”。如果再也没有可用连接时，最终这将导致 Web 应用数据库连接失败。  

针对该问题，有一个解决办法：通过配置 Apache Commons DBCP，记录并恢复这些废弃的数据库连接。它不仅能恢复这些连接，而且还能针对打开这些连接而又永远不关闭它们的代码生成堆栈跟踪。   

为了配置 DBCP 数据源来移除并回收废弃的数据库连接，将下列属性（一个或全部）添加到你的 DBCP 数据源中的 `Resource` 配置中：   

`removeAbandonedOnBorrow=true`   

`removeAbandonedOnMaintenance=true`    

以上属性默认都为 `false`。注意，只有当 `timeBetweenEvictionRunsMillis` 为正值，从而启用池维护时，`removeAbandonedOnMaintenance` 才能生效。关于这些属性的详情，可查看 [DBCP 文档](http://commons.apache.org/proper/commons-dbcp/configuration.html) 。   

使用 `removeAbandonedTimeout` 属性设置某个数据库连接闲置的秒数，超过此时段，即被认为是废弃连接。      

`removeAbandonedTimeout="60"`    

默认的去除废弃连接的超时为 300 秒。   

将 `logAbandoned` 设为 `true`，可以让 DBCP 针对那些抛弃数据库连接资源的代码，记录堆栈跟踪信息。  

`logAbandoned="true"`  

默认为 `false`。


### MySQL DBCP 范例   

#### 0. 简介  

已报告的能够正常运作的 [MySQL](http://www.mysql.com/products/mysql/index.html) 与 JDBC 驱动的版本号为：   

- MySQL 3.23.47、使用 InnoDB 的 MySQL 3.23.47、MySQL 3.23.58 以及 MySQL 4.0.1 alpha    
- [Connector/J](http://www.mysql.com/products/connector/) 3.0.11-stable （JDBC 官方驱动）     
- [mm.mysql](http://mmmysql.sourceforge.net) 2.0.14 （一个较老的 JDBC 第三方驱动）  

在继续下一步的操作之前，千万不要忘了将 JDBC 驱动的 JAR 文件复制到 `$CATALINA_HOME/lib` 中。  

#### 1. MySQL 配置   

一定要按照下面的说明去操作，否则会出现问题。   

创建一个新的测试用户、一个新的数据库，以及一张新的测试表。**必须**为 MySQL 用户指定一个密码。如果密码为空，那么在连接时，就会无法正常驱动。   

```  
mysql> GRANT ALL PRIVILEGES ON *.* TO javauser@localhost
    ->   IDENTIFIED BY 'javadude' WITH GRANT OPTION;
mysql> create database javatest;
mysql> use javatest;
mysql> create table testdata (
    ->   id int not null auto_increment primary key,
    ->   foo varchar(25),
    ->   bar int);


```

> **注意**：一旦测试结束，就该把上例中的这个用户删除！   

下面在 `testdata` 表中插入一些测试数据：   

```
mysql> insert into testdata values(null, 'hello', 12345);
Query OK, 1 row affected (0.00 sec)

mysql> select * from testdata;
+----+-------+-------+
| ID | FOO   | BAR   |
+----+-------+-------+
|  1 | hello | 12345 |
+----+-------+-------+
1 row in set (0.00 sec)

mysql>


```

#### 2. 上下文配置  

在 [Context](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 中添加资源声明，以便在 Tomcat 中配置 JNDI 数据源。   

范例如下：   

```
<Context>

    <!-- maxTotal: Maximum number of database connections in pool. Make sure you
         configure your mysqld max_connections large enough to handle
         all of your db connections. Set to -1 for no limit.
         -->

    <!-- maxIdle: Maximum number of idle database connections to retain in pool.
         Set to -1 for no limit.  See also the DBCP documentation on this
         and the minEvictableIdleTimeMillis configuration parameter.
         -->

    <!-- maxWaitMillis: Maximum time to wait for a database connection to become available
         in ms, in this example 10 seconds. An Exception is thrown if
         this timeout is exceeded.  Set to -1 to wait indefinitely.
         -->

    <!-- username and password: MySQL username and password for database connections  -->

    <!-- driverClassName: Class name for the old mm.mysql JDBC driver is
         org.gjt.mm.mysql.Driver - we recommend using Connector/J though.
         Class name for the official MySQL Connector/J driver is com.mysql.jdbc.Driver.
         -->

    <!-- url: The JDBC connection url for connecting to your MySQL database.
         -->

  <Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
               maxTotal="100" maxIdle="30" maxWaitMillis="10000"
               username="javauser" password="javadude" driverClassName="com.mysql.jdbc.Driver"
               url="jdbc:mysql://localhost:3306/javatest"/>

</Context>


```  

#### 3. web.xml 配置  

为该测试应用创建一个 `WEB-INF/web.xml` 文件：   

```  

<web-app xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
    version="2.4">
  <description>MySQL Test App</description>
  <resource-ref>
      <description>DB Connection</description>
      <res-ref-name>jdbc/TestDB</res-ref-name>
      <res-type>javax.sql.DataSource</res-type>
      <res-auth>Container</res-auth>
  </resource-ref>
</web-app>


```

#### 4. 测试代码  

创建一个简单的 `test.jsp` 页面，稍后将用到它。   

```  

<%@ taglib uri="http://java.sun.com/jsp/jstl/sql" prefix="sql" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>

<sql:query var="rs" dataSource="jdbc/TestDB">
select id, foo, bar from testdata
</sql:query>

<html>
  <head>
    <title>DB Test</title>
  </head>
  <body>

  <h2>Results</h2>

<c:forEach var="row" items="${rs.rows}">
    Foo ${row.foo}<br/>
    Bar ${row.bar}<br/>
</c:forEach>

  </body>
</html>


```   

JSP 页面用到了 [JSTL](http://www.oracle.com/technetwork/java/index-jsp-135995.html) 的 SQL 和 Core taglibs。你可以从 [Apache Tomcat Taglibs - Standard Tag Library](http://tomcat.apache.org/taglibs/standard/) 项目中获取它，不过要注意应该是 1.1.x 或之后的版本。下载 JSTL 后，将 `jstl.jar` 和 `standard.jar` 复制到 Web 应用的 `WEB-INF/lib` 目录中。   


最后，将你的应用部署到 `$CATALINA_BASE/webapps`，可以采用两种方式：或者将应用以名叫 `DBTest.war` 的 WAR 文件形式部署；或者把应用放入一个叫 `DBTest` 的子目录中。

部署完毕后，就可以在浏览器输入 `http://localhost:8080/DBTest/test.jsp`，查看你的第一个劳动成果了。     


### Oracle 8i、9i 与 10g   

#### 0. 简介   

Oracle 需要的配置和 MySQL 差不多，只不过也存在一些常见问题。   

针对过去版本的 Oracle 的驱动可能以 *.zip 格式（而不是 *.jar 格式）进行分发的。Tomcat 只使用 `*.jar` 文件，而且它们还必须安装在 `$CATALINA_HOME/lib` 中。因此，`classes111.zip` 或 `classes12.zip` 这样的文件后缀应该改成 `.jar`。因为 jar 文件本来就是一种 zip 文件，因此不需要将原 zip 文件解压缩然后创建相应的 jar 文件，只需改换后缀名即可。   

对于 Oracle 9i 之后的版本，应该使用 `oracle.jdbc.OracleDriver` 而不是 `oracle.jdbc.driver.OracleDriver`，因为 Oracle 规定开始弃用 `oracle.jdbc.driver.OracleDriver`，下一个重大版本将不再支持这一驱动类。   


#### 1. 上下文配置  

跟前文 MySql 的配置一样，你也需要在 [Context](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 中定义数据源。下面定义一个叫做 myoracle 的数据源，使用上文说的短驱动来连接（用户名为 scott，密码为 tiger）到名为 mysid 的SID（Oracle 系统ID，标识一个数据库的唯一标示符）。 用户 scott 使用的 Schema 就是默认的 schema。

使用 OCI 驱动，只需在 URL 字符串中将 thin 变为 oci 即可。     

```
<Resource name="jdbc/myoracle" auth="Container"
              type="javax.sql.DataSource" driverClassName="oracle.jdbc.OracleDriver"
              url="jdbc:oracle:thin:@127.0.0.1:1521:mysid"
              username="scott" password="tiger" maxTotal="20" maxIdle="10"
              maxWaitMillis="-1"/>

```   




#### 2. web.xml 配置    

在创建 Web 应用的 web.xml 文件时，一定要遵从 Web 应用部署描述符文件中 DTD 所需要的元素顺序。  

```   
<resource-ref>
 <description>Oracle Datasource example</description>
 <res-ref-name>jdbc/myoracle</res-ref-name>
 <res-type>javax.sql.DataSource</res-type>
 <res-auth>Container</res-auth>
</resource-ref>

```   

#### 3. 代码范例    

可以使用上文所列的范例应用（假如你创建了所需的 DB 实例和表，等等），将数据源代码用下面的代码替换：   

```   
Context initContext = new InitialContext();
Context envContext  = (Context)initContext.lookup("java:/comp/env");
DataSource ds = (DataSource)envContext.lookup("jdbc/myoracle");
Connection conn = ds.getConnection();
//etc.

```   



### PostgreSQL  

#### 0. 简介  

PostgreSQL 配置与 Oracle 基本相似。  

#### 1. 所需文件   

将 Postgres 的 JDBC jar 文件复制到 `$CATALINA_HOME/lib` 中。和 Oracle 配置一样，jar 文件必须放在这个目录中，DBCP 类加载器才能找到它们。不管接下来如何配置，这是首先必须要做的。   

#### 2. 资源配置   

目前有两种选择：定义一个能够被 Tomcat 所有应用所共享的数据源，或者定义只能被单个应用所使用的数据源。   

##### 2a. 共享数据源配置  

如果想定义能够被多个 Tomcat 应用所共享的数据源，或者只想在文件中定义自己的数据源，则采用如下配置：       

**尽管有些用户反馈说这样可行，但本文档作者却没有成功，希望有人能阐述清楚。**   

```  
<Resource name="jdbc/postgres" auth="Container"
          type="javax.sql.DataSource" driverClassName="org.postgresql.Driver"
          url="jdbc:postgresql://127.0.0.1:5432/mydb"
          username="myuser" password="mypasswd" maxTotal="20" maxIdle="10" maxWaitMillis="-1"/>


```  

##### 2b. 应用专属的资源配置   

如果希望专门为某一应用定义数据源，其他 Tomcat 应用无法使用，可以使用如下配置。这种方法对 Tomcat 安装的损害性要小一些。  

在你的应用的 [Context](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 中创建一个资源定义，如下所示：   

```
<Context>

<Resource name="jdbc/postgres" auth="Container"
          type="javax.sql.DataSource" driverClassName="org.postgresql.Driver"
          url="jdbc:postgresql://127.0.0.1:5432/mydb"
          username="myuser" password="mypasswd" maxTotal="20" maxIdle="10"
maxWaitMillis="-1"/>
</Context>

   
```

#### 3. web.xml 配置   

```   
<resource-ref>
 <description>postgreSQL Datasource example</description>
 <res-ref-name>jdbc/postgres</res-ref-name>
 <res-type>javax.sql.DataSource</res-type>
 <res-auth>Container</res-auth>
</resource-ref>

```   

#### 4. 访问数据库   

在利用程序访问数据库时，记住把 `java:/comp/env` 放在你的 JNDI lookup 方法参数的前部，如下面这段代码所示。另外，可以用任何你想用的值来替换 `jdbc/postgres`，不过记得也要用同样的值来修改上面的资源定义文件。     

```
InitialContext cxt = new InitialContext();
if ( cxt == null ) {
   throw new Exception("Uh oh -- no context!");
}

DataSource ds = (DataSource) cxt.lookup( "java:/comp/env/jdbc/postgres" );

if ( ds == null ) {
   throw new Exception("Data source not found!");
}
   
```

## 非 DBCP 的解决方案  

这些方案或者使用一个单独的数据库连接（建议仅作测试用！），或者使用其他一些池化技术。  

## Oracle 8i 与 OCI 客户端  

### 简介  

虽然并不能严格地解决如何使用 OCI 客户端来创建 JNDI 数据源的问题，但这些注意事项却能和上文提到的 Oracle 与 DBCP 解决方案结合起来使用。  

为了使用 OCI 驱动，应该先安装一个 Oracle 客户。你应该已经通过光盘安装好了 Oracle 8i（8.1.7）客户端，并从 [otn.oracle.com](http://www.oracle.com/technetwork/index.html) 下载了适用的 JDBC/OCI 驱动（Oracle8i 8.1.7.1 JDBC/OCI 驱动）。  

将 `classes12.zip` 重命名为 `classes12.jar` 后，将其复制到 `$CATALINA_HOME/lib` 中。根据 Tomcat 的版本以及你所使用的 JDK，你可能还必须该文件中的删除 `javax.sql.*` 类。   


### 连接起来   

确保在 `$PATH` 或 `LD_LIBRARY_PATH`（可能在 `$ORAHOME\bin`）目录下存在 `ocijdbc8.dll` 或 `.so` 文件，另外还要确认能否使用 `System.loadLibrary("ocijdbc8");` 这样的简单测试程序加载本地库。   

下面你应该创建一个简单测试用 servlet 或 jsp，其中应该包含以下关键代码：  

```  
DriverManager.registerDriver(new
oracle.jdbc.driver.OracleDriver());
conn =
DriverManager.getConnection("jdbc:oracle:oci8:@database","username","password");

```   

目前数据库是 `host:port:SID` 形式，如果你试图访问测试用servlet/jsp，那么你会得到一个 `ServletException ` 异常，造成异常的根本原因在于 `java.lang.UnsatisfiedLinkError:get_env_handle`。  

分析一下，首先 `UnsatisfiedLinkError` 表明：   

- JDBC 类文件和 Oracle 客户端版本不匹配。消息中透露出的意思是没有找到需要的库文件。比如，你可能使用 Oracle 8.1.6 的 class12.zip 文件，而 Oracle 客户端版本则是 8.1.5。classeXXXs.zip 文件必须与 Oracle 客户端文件版本相一致。   

- 出现了一个 `$PATH, LD_LIBRARY_PATH` 问题。   

- 有报告称，忽略从 otn 网站下载的驱动，使用 `$ORAHOME\jdbc\lib` 目录中的 class12.zip 文件，同样能够正常运作。  

接下来，你可能还会遇到另一个错误消息：`ORA-06401 NETCMN: invalid driver designator`。   

Oracle 文档是这么说的：“异常原因：登录（连接）字符串包含一个不合法的驱动标识符。解决方法：修改字符串，重新提交。”所以，如下面这样来修改数据库（`host:port:SID`）连接字符串：   

`(description=(address=(host=myhost)(protocol=tcp)(port=1521))(connect_data=(sid=orcl)))`  



## 常见问题   

下面是一些 Web 应用在使用数据库时经常会遇到的问题，以及一些应对技巧。   

### 数据库连接间歇性失败    

Tomcat 运行在 JVM 中。JVM 周期性地会执行垃圾回收（GC），清除不再使用的 Java 对象。当 JVM 执行 GC 时，Tomcat 中的代码执行就会终止。如果配置好的数据库连接建立的最长时间小于垃圾回收的时间，数据库连接就会失败。   

在启动 Tomcat 时，将 `-verbose:gc` 参数添加到 `CATALINA_OPTS` 环境变量中，就能知道垃圾回收所占用的时间了。在启用 `verbose:gc` 后， `$CATALINA_BASE/logs/catalina.out` 日志文件就能包含每次垃圾回收的数据，其中也包括它所占用的时间。   

正确调整 JVM 后，垃圾回收可以做到在 99% 的情况下占用时间不超过 1 秒。剩余的情况则只占用几秒钟的时间，只有极少数情况下 GC 会占用超过 10 秒钟的时间。   

保证让数据库连接超时设定在 10~15 秒。对于 DBCP，可以使用 `maxWaitMillis` 参数来设置。   

### 随机性的连接关闭异常  

当某一请求从连接池中获取了一个数据库连接，然后关闭了它两次时，往往会出现这样的异常消息。使用连接池时，关闭连接，就会把它归还给连接池，以便之后其他的请求能够重用该连接，而并不会关闭连接。Tomcat 使用多个线程来处理并发请求。下面这个范例就演示了，在 Tomcat 中，一系列事件导致了这种错误。    

运行在**线程 1** 中的**请求 1** 获取了一个连接。

**请求 1** 关闭了数据库连接。

JVM 将运行的线程切换为**线程 2**。  

**线程 2** 中运行的**请求 2** 获取了一个数据库连接。  
（同一个数据库连接刚被**请求 1** 关闭）  

JVM 又将运行的线程切换回为**线程 1**。  

  
**请求 1** 第二次关闭了数据库连接。

JVM 将运行的线程切换回**线程 2**。  

**请求 2** 和**线程 2** 试图使用数据库连接，但却失败了。因为**请求 1** 已经关闭了它。



```     

  Connection conn = null;
  Statement stmt = null;  // Or PreparedStatement if needed
  ResultSet rs = null;
  try {
    conn = ... get connection from connection pool ...
    stmt = conn.createStatement("select ...");
    rs = stmt.executeQuery();
    ... iterate through the result set ...
    rs.close();
    rs = null;
    stmt.close();
    stmt = null;
    conn.close(); // Return to connection pool
    conn = null;  // Make sure we don't close it twice
  } catch (SQLException e) {
    ... deal with errors ...
  } finally {
    // Always make sure result sets and statements are closed,
    // and the connection is returned to the pool
    if (rs != null) {
      try { rs.close(); } catch (SQLException e) { ; }
      rs = null;
    }
    if (stmt != null) {
      try { stmt.close(); } catch (SQLException e) { ; }
      stmt = null;
    }
    if (conn != null) {
      try { conn.close(); } catch (SQLException e) { ; }
      conn = null;
    }
  }


```

### 上下文与全局命名资源   

注意，虽然在上面的说明中，把 JNDI 声明放在一个 Context 元素里面，但还是有可能（而且有时更需要）把这些声明放在服务器配置文件的 [GlobalNamingResources](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 区域。被放置在 GlobalNamingResources 区域的资源将会被服务器的各个上下文所共享。

### JNDI 资源命名和 Realm 交互   

为了让 Realm 能运作，realm 必须指向定义在 `<GlobalNamingResources>` 或 `<Context>` 区域中的数据源，而不是`<ResourceLink>` 重新命名的数据源。   

