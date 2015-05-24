## 目录  

## 简介   

**JDBC 连接池** `org.apache.tomcat.jdbc.pool` 是 [Apache Commons DBCP](http://commons.apache.org/proper/commons-dbcp/) 连接池的一种替换或备选方案。    

那究竟为何需要一个新的连接池？   

原因如下：   

1. Commons DBCP 1.x 是单线程。为了线程安全，在对象分配或对象返回的短期内，Commons 锁定了全部池。但注意这并不适用于 Commons DBCP 2.x。  
2. Commons DBCP 1.x 可能会变得很慢。当逻辑 CPU 数目增长，或者试图借出或归还对象的并发线程增加时，性能就会受到影响。高并发系统受到的影响会更为显著。注意这并不适用于 Commons DBCP 2.x。   
3. Commons DBCP 拥有 60 多个类。tomcat-jdbc-pool 核心只有 8 个类。因此为了未来需求变更着想，肯定需要更少的改动。我们真正需要的只是连接池本身，其余的只是附属。
4. Commons DBCP 使用静态接口，因此对于指定版本的 JRE，只能采用正确版本的 DBCP，否则就会出现 `NoSuchMethodException` 异常。   
5. 当DBCP 可以用其他更简便的实现来替代时，实在不值得重写那 60 个类。     
6. Tomcat JDBC 连接池无需为库本身添加额外线程，就能获取异步获取连接，》》。   
7. Tomcat JDBC 连接池是 Tomcat 的一个模块，依靠 Tomcat JULI 这个简化了的日志架构。   
8. 使用 `javax.sql.PooledConnection` 接口获取底层连接。   
9. 防止饥饿。如果池变空，线程将等待一个连接。当连接返回时，池就将唤醒正确的等待线程。大多数连接池只会一直维持饥饿状态。


Tomcat JDBC 连接池还具有一些其他连接池实现所没有的特点：     

1. 支持高并发环境与多核/CPU 系统。   
2. 接口的动态实现。支持 java.sql 与 java.sql 接口（只要 JDBC 驱动），甚至在利用低版本的 JDK 来编译时。   
3. 验证间隔时间。我们不必每次使用单个连接时都进行验证，可以在借出或归还连接时进行验证，只要不低于我们所设定的间隔时间就行。
4. 只执行一次查询。当与数据库建立起连接时，只执行一次的可配置查询。这项功能对会话设置非常有用，因为你可能会想在连接建立的整个时段内都保持会话。   
5. 能够配置自定义拦截器。通过自定义拦截器来增强功能。可以使用拦截器来采集查询统计，缓存会话状态，重新连接之前失败的连接，重新查询，缓存查询结果，等等。由于可以使用大量的选项，所以这种自定义拦截器也是没有限制的，跟 `java.sql`/`javax.sql` 接口的 JDK 版本没有任何关系。  
6. 高性能。后文将举例展示一些性能差异。   
7. 极其简单。它的实现非常简单，代码行数与源文件都非常少，这都有赖于从一开始研发它时，就把简洁当做重中之重。对比一下c3p0 ，它的源文件超过了 200 个（最近一次统计），而 Tomcat JDBC 核心只有 8 个文件，连接池本身则大约只有这个数目的一半，所以能够轻易地跟踪和修改可能出现的 Bug。  
8. 异步连接获取。可将连接请求队列化，系统返回 `Future<Connection>`。  
9. 更好地处理空闲连接。不再简单粗暴地直接把空闲连接关闭，而是仍然把连接保留在池中，通过更为巧妙的算法控制空闲连接池的规模。     
10. 可以控制连接应被废弃的时间：当池满了即废弃，或者指定一个池使用容差值，发生超时就进行废弃处理。   
11. 通过查询或语句来重置废弃连接计时器。允许一个使用了很长时间的连接不因为超时而被废弃。这一点是通过使用 `ResetAbandonedTimer` 来实现的。  
12. 经过指定时间后，关闭连接。与返回池的时间相类似。
13. 当连接要被释放时，获取 JMX 通知并记录所有日志。它类似于 `removeAbandonedTimeout`，但却不需要采取任何行为，只需要报告信息即可。通过 `suspectTimeout` 属性来实现。
14. 可以通过 `java.sql.Driver`、`javax.sql.DataSource` 或 `javax.sql.XADataSource` 获取连接。通过 `dataSource` 与 `dataSourceJNDI` 属性实现这一点。  
15. 支持 XA 连接。   

## 使用方法  

对于熟悉 Commons DBCP 的人来说，转而使用 Tomcat 连接池是非常简单的事。从其他连接池转换过来也非常容易。   

### 附加功能   

除了其他多数连接池能够提供的功能外，Tomcat 连接池还提供了一些附加功能：   

- `initSQL` 当连接创建后，能够执行一个 SQL 语句（只执行一次）。
- `validationInterval` 恰当地在连接上运行验证，同时又能避免太多频繁地执行验证。   
- `jdbcInterceptors` 灵活并且可插拔的拦截器，能够对池进行各种自定义，执行各种查询，处理结果集。下文将予以详述。     
- `fairQueue` 将 fair 标志设为 true，以达成线程公平性，或使用异步连接获取。       


### Apache Tomcat 容器内部   

Tomcat 连接池被配置为一个资源，》》》   在[》？Tomcat JDBC 文档](http://tomcat.apache.org/tomcat-8.0-doc/jndi-datasource-examples-howto.html)中有所说明，唯一的区别在于，你必须指定 `factory` 属性，并将其值设为  `org.apache.tomcat.jdbc.pool.DataSourceFactory`。  


### 独立性  

连接池只有另一个 tomcat-juli.jar。`org.apache.tomcat.jdbc.pool.DataSource`   

### JMX

连接池对象暴露了一个可以被注册的 MBean。为了让连接池对象创建 MBean，`jmxEnabled` 标志必须设为 true。这并不是说连接池》。在像 Tomcat 这样的容器中，Tomcat 本身注册》。`org.apache.tomcat.jdbc.pool.DataSource` 对象会注册实际的连接池 MBean。》》 


## 属性  

为了能够顺畅地在 Commons DBCP 与 Tomcat JDBC 连接池 之间转换，大多数属性名称及其含义都是相同的。   

### JNDI 工厂与类型    

|属性|描述|
|---|---|
|`factory`|必需的属性，其值应为 `org.apache.tomcat.jdbc.pool.DataSourceFactory` |
|`type`|类型应为 `javax.sql.DataSource` 或 `javax.sql.XADataSource`。<br/>根据类型，将创建`org.apache.tomcat.jdbc.pool.DataSource` 或 `org.apache.tomcat.jdbc.pool.XADataSource`。|  



### 系统属性    

系统属性作用于 JVM 范围，影响创建于 JVM 内的所有池。》资源池还是连接池？》   

|属性|描述|
|---|---|
|`org.apache.tomcat.jdbc.pool.onlyAttemptCurrentClassLoader`|布尔值，默认为 `false`。控制动态类（如JDBC 驱动、拦截器、验证器）的加载。如果采用默认值，池会首先利用当前类加载器（比如已经加载池类的类加载器）加载类；如果类加载失败，则尝试利用线程上下文加载器加载。取值为 `true` 时，会向后兼容 Apache Tomcat 8.0.8 及更早版本，只会采用当前类加载器。如果未设置，则取默认值。|

### 常用属性   

|属性|描述|
|---|---|
|`defaultAutoCommit`|（布尔值）连接池所创建的连接默认自动提交状态。如果未设置，则默认采用 JDBC 驱动的缺省值（如果未设置，则不会调用 `setAutoCommit` 方法）。|   
|`defaultReadOnly`|（布尔值）连接池所创建的连接默认只读状态。如果未设置，将不会调用 `setReadOnly` 方法。（有些驱动并不支持只读模式，比如：informix）|  
|`defaultTransactionIsolation`|（字符串）连接池所创建的连接的默认事务隔离状态。取值范围为：（参考 javadoc）<br/><li>`NONE`</li><li>`READ_COMMITTED`</li><li>`READ_UNCOMMITTED`</li><li>`REPEATABLE_READ`</li><li>`SERIALIZABLE`</li><br/>如果未设置该值，则不会调用任何方法，默认为 JDBC 驱动。|    
|`defaultCatalog`|（字符串）连接池所创建的连接的默认catalog。》数据字典？》|  
|`driverClassName`|（字符串）所要使用的 JDBC 驱动的完全限定的 Java 类名。该驱动必须能从同一个类加载器》》tomcat-jdbc.jar访问|  
|`username`|（字符串）传入 JDBC 驱动以便建立连接的连接用户名。注意，`DataSource.getConnection(username,password)` 方法默认不会使用传入该方法内的凭证，但会使用这里的配置信息。可参看 `alternateUsernameAllowed` 了解更多详情。|  
|`password`|（字符串）传入 JDBC 驱动以便建立连接的连接密码。注意，`DataSource.getConnection(username,password)` 方法默认不会使用传入该方法内的凭证，但会使用这里的配置信息。可参看 `alternateUsernameAllowed` 了解更多详情。|
|`maxActive`|（整形值）池同时能分配的活跃连接的最大数目。默认为 `100`。|
|`maxIdle`|（整型值）池始终都应保留的连接的最大数目。默认为 `maxActive:100`。会周期性检查空闲连接（如果启用该功能），留滞时间超过 `minEvictableIdleTimeMillis` 的空闲连接将会被释放。（请参考 `testWhileIdle`）  |    
|`minIdle`|（整型值）池始终都应保留的连接的最小数目。如果验证查询失败，则连接池会缩减该值。默认值取自 `initialSize:10`（请参考 `testWhileIdle`）。|  
|`initialSize`|（整型值）连接器启动时创建的初始连接数。默认为 `10`。|  
|`maxWait`|（整型值）在抛出异常之前，连接池等待（没有可用连接时）返回连接的最长时间，以毫秒计。默认为 `30000`（30 秒）|  
|`testOnBorrow`|（布尔值）默认值为 `false`。从池中借出对象之前，是否对其进行验证。如果对象验证失败，将其从池中清除，再接着去借下一个。注意：为了让 `true` 值生效，`validationQuery` 参数必须为非空字符串。为了实现更高效的验证，可以采用 `validationInterval`。|  
|`testOnReturn`|（布尔值）默认值为 `false`。将对象返回池之前，是否对齐进行验证。注意：为了让 `true` 值生效，`validationQuery` 参数必须为非空字符串。|  
|`testWhileIdle`|（布尔值）是否通过空闲对象清除者（如果存在的话）验证对象。如果对象验证失败，则将其从池中清除。注意：为了让 `true` 值生效，`validationQuery` 参数必须为非空字符串。该属性默认值为 `false`，为了运行池的清除/测试线程，必须设置该值。（另请参阅 `timeBetweenEvictionRunsMillis`）|  
|`validationQuery`|（字符串）在将池中连接返回给调用者之前，用于验证这些连接的 SQL 查询。如果指定该值，则该查询不必返回任何数据，只是不抛出 `SQLException` 异常。默认为 `null`。实例值为：`SELECT 1`（MySQL） `select 1 from dual`（Oracle） `SELECT 1`（MySQL Server）。|  
|`validationQueryTimeout`|（整型值）连接验证失败前的超时时间（以秒计）。通过在执行 `validationQuery` 的语句上调用 `java.sql.Statement.setQueryTimeout(seconds)` 来实现。池本身并不会让查询超时，完全是由 JDBC 来强制实现。若该值小于或等于 0，则禁用该功能。默认为 `-1`。|    
|`validatorClassName`|（字符串）实现 `org.apache.tomcat.jdbc.pool.Validator` 接口并提供了一个无参（可能是隐式的）构造函数的类名。如果指定该值，将通过该类来创建一个 Validator 实例来验证连接，代替任何验证查询。默认为 `null`，范例值为：`com.mycompany.project.SimpleValidator`。|  
|`timeBetweenEvictionRunsMillis`|（整型值）空闲连接验证/清除线程运行之间的休眠时间（以毫秒计）。不能低于 1 秒。该值决定了我们检查空闲连接、废弃连接的频率，以及验证空闲连接的频率。默认为 `5000`（5 秒）|  
|`numTestsPerEvictionRun`|（整型值）Tomcat JDBC 连接池没有用到这个属性。|  
|`minEvictableIdleTimeMillis`|（整型值）在被确定应被清除之前，对象在池中保持空闲状态的最短时间（以毫秒计）。默认为 `60000`（60 秒）|  
|`accessToUnderlyingConnectionAllowed`|（布尔值）没有用到的属性。可以在归入池内的连接上调用 `unwrap`来访问。参阅 `javax.sql.DataSource` 接口的相关介绍，或者通过反射调用 `getConnection`，或者将对象映射为 `javax.sql.PooledConnection`。|  
|`removeAbandoned`|（布尔值）该值为标志（Flag）值，表示如果连接时间超出了 `removeAbandonedTimeout`，则将清除废弃连接。如果该值被设置为 `true`，则如果连接时间大于 `removeAbandonedTimeout`，该连接会被认为是废弃连接，应予以清除。若应用关闭连接失败时，将该值设为 `true` 能够恢复该应用的数据库连接。另请参阅 `logAbandoned`。默认值为 `false`。|  
|`removeAbandonedTimeout`|（整型值）在废弃连接（仍在使用）可以被清除之前的超时秒数。默认为 `60`（60 秒）。应把该值设定为应用可能具有的运行时间最长的查询。|  
|`logAbandoned`|（布尔值）标志能够针对丢弃连接的应用代码，进行堆栈跟踪记录。由于生成堆栈跟踪，对废弃连接的日志记录会增加每一个借取连接的开销。默认为 `false`| 
|`connectionProperties`|（字符串）在建立新连接时，发送给 JDBC 驱动的连接属性。字符串格式必须为：[propertyName=property;]*。注意：user 与 password 属性会显式传入，因此这里并不需要包括它们。默认为 `null。`|
|`poolPreparedStatements`|（布尔值）未使用的属性|  
|`maxOpenPreparedStatements`|（整型值）未使用的属性|  


### Tomcat JDBC 增强属性     

|属性|描述|  
|---|---|  
|` initSQL `|字符串值。当连接第一次创建时，运行的自定义查询。默认值为 `null`。|  
|` jdbcInterceptors `|字符串。继承自类 `org.apache.tomcat.jdbc.pool.JdbcInterceptor` 的子类类名列表，由分号分隔。关于格式及范例，可参见下文的配置 JDBC 拦截器》》。<br/><br/> `java.sql.Connection` 对象》》 <br/><br/>预定义的拦截器有：<br/><li>`org.apache.tomcat.jdbc.pool.interceptor`</li><li>`ConnectionState`——记录自动提交、只读、catalog以及事务隔离级别等状态。</li><li>`org.apache.tomcat.jdbc.pool.interceptor`</li><li>`StatementFinalizer`——记录打开的语句，并当连接返回池后关闭它们。</li><br/><br/>有关更多预定义拦截器的详尽描述，可参阅[JDBC 拦截器》》] |  
|` validationInterval `|长整型值。为避免过度验证而设定的频率时间值（以秒计）。最多以这种频率运行验证。如果连接应该进行验证，但却没能在此间隔时间内得到验证，则会重新对其进行验证。默认为 `30000`（30 秒）。|  
|` jmxEnabled `|布尔值。是否利用 JMX 注册连接池。默认为 `true`。|  
|` fairQueue `|布尔值。假如想用真正的 FIFO 方式公平对待 `getConnection` 调用，则取值为 `true`。对空闲连接列表将采用 `org.apache.tomcat.jdbc.pool.FairBlockingQueue` 实现。默认值为 `true`。如果想使用异步连接获取功能，则必须使用该标志。<br/>设置该标志可保证线程能够按照连接抵达顺序来接收连接。<br/>在性能测试时，锁及锁等待的实现方式有很大差异。当 `fairQueue=true` 时，根据所运行的操作系统，存在一个决策过程。假如系统运行在 Linux 操作系统（属性 `os.name = linux`）上，为了禁止这个 Linux 专有行为，但仍想使用公平队列，那么只需在连接池类加载之前，将 `org.apache.tomcat.jdbc.pool.FairBlockingQueue.ignoreOS=true` 添加到系统属性上。|  
|` abandonWhenPercentageFull `|整型值。除非使用中连接的数目超过  `abandonWhenPercentageFull` 中定义的百分比，否则不会关闭并报告已废弃的连接（因为超时）。取值范围为 0-100。默认值为 0，意味着只要达到 `removeAbandonedTimeout`，就应关闭连接。|  
|` maxAge `|长整型值。连接保持时间（以毫秒计）。当连接要返回池中时，连接池会检查是否达到 `now - time-when-connected > maxAge` 的条件，如果条件达成，则关闭该连接，不再将其返回池中。默认值为 `0`，意味着连接将保持开放状态，在将连接返回池中时，不会执行任何年龄检查。|  
|` useEquals `|布尔值。如果想让 `ProxyConnection` 类使用 `String.equals`，则将该值设为 `true`；若想在对比方法名称时使用 `==`，则应将其设为 `false`。该属性不能用于任何已添加的拦截器中，因为那些拦截器都是分别配置的。默认值为 `true`。|  
|` suspectTimeout `|整型值。超时时间（以秒计）。默认值为 `0`。<br/>类似于 `removeAbandonedTimeout`，但不会把连接当做废弃连接从而有可能关闭连接。如果 `logAbandoned` 设为 `true`，它只会记录下警告。如果该值小于或等于 0，则不会执行任何怀疑式检查。如果超时值大于 0，而连接还没有被废弃，或者废弃检查被禁用时，才会执行怀疑式检查。如果某个连接被怀疑到，则记录下 WARN 信息并发送一个 JMX 通知。|   
|` rollbackOnReturn `|布尔值。如果 `autoCommit==false`，那么当连接返回池中时，池会在连接上调用回滚方法，从而终止事务。默认值为 `false`。|  
|` commitOnReturn `|布尔值。如果 `autoCommit==false`，那么当连接返回池中时，池会在连接上调用提交方法，从而完成事务；如果 `rollbackOnReturn==true`，则忽略该属性。默认值为 `false`。|  
|` alternateUsernameAllowed `|布尔值。出于性能考虑，JDBC 连接池默认会忽略 [`DataSource.getConnection(username,password)`](http://docs.oracle.com/javase/6/docs/api/javax/sql/DataSource.html#getConnection(java.lang.String,%20java.lang.String))调用，只返回之前池化的具有全局配置属性 `username ` 和 `password`的连接。<br/><br/>但经过配置，连接池还可以允许使用不同的凭证来请求每一个连接。为了启用这项在[`DataSource.getConnection(username,password)`](http://docs.oracle.com/javase/6/docs/api/javax/sql/DataSource.html#getConnection(java.lang.String,%20java.lang.String))调用中描述的功能，只需将 `alternateUsernameAllowed` 设为 `true`。<br/>》》》<br/>默认值为 `false`。<br/>该属性作为一个改进方案，被添加到了 [bug 50025](https://bz.apache.org/bugzilla/show_bug.cgi?id=50025) 中。 |  
|` dataSource `|（javax.sql.DataSource）将数据源注入连接池，从而使池利用数据源来获取连接，而不是利用 `java.sql.Driver` 接口来建立连接。它非常适于使用数据源（而非连接字符串）来池化 XA 连接或者已建立的连接时。默认值为 `null`。|  
|` dataSourceJNDI `|字符串。在 JNDI 中查找的数据源的 JNDI 名称，随后将用于建立数据库连接。参看 `datasource` 属性的介绍。默认值为 `null`。|  
|` useDisposableConnectionFacade `|布尔值。如果希望在连接上放上一个facade，从而使连接在关闭后无法重用，则要将值设为 `true`。》》》默认值为 `true`。|  
|` logValidationErrors `|布尔值。设为 `true` 时，能将验证阶段的错误记录到日志文件中，错误会被记录为 SEVERE。考虑到了向后兼容性，默认值为 `false`。|
|` propagateInterruptState `|布尔值。传播已中断的线程（还没有清除中断状态）的中断状态。考虑到了向后兼容性，默认值为 `false`。|  
|` ignoreExceptionOnPreLoad `|布尔值。在初始化池时，是否忽略连接创建错误。取值为 `true`时表示忽略；设为 `false` 时，抛出异常，从而宣告池初始化失败。默认值为 `false`。|  


## 高级用法    

### JDBC 拦截器    

要想看看拦截器使用方法的具体范例，可以看看 `org.apache.tomcat.jdbc.pool.interceptor.ConnectionState`。这个简单的拦截器缓存了三个属性：`autoCommit`、`readOnly`、`transactionIsolation`，为的是避免系统与数据库之间无用的往返。  

当需求增加时，姜维连接池核心增加更多的拦截器。欢迎贡献你的才智！

拦截器当然并不局限于 `java.sql.Connection`，当然也可以对方法调用的任何结果进行包装。你可以构建查询性能分析器，以便当查询运行时间超过预期时间时提供 JMX 通知。  



### 配置 JDBC 拦截器   

JDBC 拦截器是通过 **jdbcInterceptor** 属性来配置的。该属性值包含一列由分号分隔的类名。如果这些类名非完全限定，就会在它们的前面加上  `org.apache.tomcat.jdbc.pool.interceptor.` 前缀。

范例：  
`jdbcInterceptors="org.apache.tomcat.jdbc.pool.interceptor.ConnectionState; org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer"`  
它实际上等同于：  
`jdbcInterceptors="ConnectionState;StatementFinalizer"`   

拦截器也同样有属性。拦截器的属性指定在类名后的括号里，如果设置多个属性，则用逗号分隔开。  

范例：  
`jdbcInterceptors="ConnectionState;StatementFinalizer(useEquals=true)"`  

系统会自动忽略属性名称、属性值以及类名前后多余的空格字符。


### org.apache.tomcat.jdbc.pool.JdbcInterceptor     

所有拦截器的抽象基类，无法实例化。  

|属性|描述|  
|---|---|  
|useEquals|（布尔值）如果希望 `ProxyConnection` 类使用 `String.equals`，则设为 true；当希望在对比方法名时使用 `==`，则设为 `false`。默认为 `true`。|  

### org.apache.tomcat.jdbc.pool.interceptor.ConnectionState  

它能为下列属性缓存连接：`autoCommit`、`readOnly`、`transactionIsolation` 及 `catalog`。这是一个性能增强》》》，当利用已设定的值来调用 getter 与 setter 时，它能够避免往返数据库。

### org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer

跟踪所有使用 `createStatement`、`prepareStatement` 或 `prepareCall` 的语句，当连接返回池后，关闭这些语句。  

|属性|描述|
|---|---|
|trace|（以字符串形式表示的布尔值）对未关闭语句进行跟踪。当启用跟踪且连接被关闭时，如果相关语句没有关闭，则拦截器会记录所有的堆栈跟踪。默认值为 `false`。|  

### org.apache.tomcat.jdbc.pool.interceptor.StatementCache

缓存连接中的 `PreparedStatement` 或 `CallableStatement` 实例。  

它会针对每个连接对这些语句进行缓存，然后计算池中所有连接的整体缓存数，如果缓存数超过了限制 `max`，就不再对随后的语句进行缓存，而是直接关闭它们。


|属性|描述|
|---|---|
|`prepared`|（以字符串形式表示的布尔值）对使用 `prepareStatement` 调用创建的 `PreparedStatement` 实例进行缓存。默认为 `true` |
|`callable`|（以字符串形式表示的布尔值）对使用 `prepareCall` 调用创建的 `CallableStatement` 实例进行缓存。默认为 `false`|
|`max`|（以字符串形式表示的整型值）连接池中的缓存语句的数量限制。默认为 `50`|    


### org.apache.tomcat.jdbc.pool.interceptor.StatementDecoratorInterceptor   

请参看 [48392](https://bz.apache.org/bugzilla/show_bug.cgi?id=48392)。拦截器会包装语句和结果集，从而防止对使用了 `ResultSet.getStatement().getConnection()` 和 `Statement.getConnection()` 方法的实际连接进行访问。  

### org.apache.tomcat.jdbc.pool.interceptor.QueryTimeoutInterceptor

当新语句创建时，自动调用 `java.sql.Statement.setQueryTimeout(seconds)`。池本身并不会让查询超时，完全是依靠 JDBC 驱动来强制查询超时。  

|属性|描述|
|---|---|
|`queryTimeout`|（以字符串形式表示的整型值）查询超时的毫秒数。默认为 `1000` 毫秒。|  

### org.apache.tomcat.jdbc.pool.interceptor.SlowQueryReport  

当查询超过失败容差值时，记录查询性能并发布日志项目。使用的日志级别为 `WARN`。

|属性|描述|
|---|---|
| `threshold` |（以字符串形式表示的整型值）查询应超时多少毫秒才发布日志警告。默认为 `1000` 毫秒|
| `maxQueries` |（以字符串形式表示的整型值）为保留内存空间，所能记录的最大查询数量。默认为 `1000`|
| `logSlow` |（以字符串形式表示的布尔值）如果想记录较慢的查询，设为 `true`。默认为 `true`|
| `logFailed` |（以字符串形式表示的布尔值）如果想记录失败查询，设为 `true`。默认为 `true`|  


### org.apache.tomcat.jdbc.pool.interceptor.SlowQueryReportJmx  

这是对 `SlowQueryReport` 的扩展，除了发布日志项目外，它还发布 JMX 通知，以便监视工具作出相关反应。该类从其父类继承了所有属性。它使用了 Tomcat 的 JMX 引擎，所以在 Tomcat 容器外部是无效的。使用该类时，默认情况下，是通过 `ConnectionPool` MBean 来发送 JMX 通知。如果 `notifyPool=false`，则 `SlowQueryReportJmx` 也可以注册一个 MBean。  


|属性|描述|
|---|---|
|` notifyPool `|（以字符串形式表示的布尔值）如果希望用 `SlowQueryReportJmx` MBean 发送 JMX 通知，则设为 `false`。默认为 `true`|
|` objectName `|字符串。定义一个有效的 `javax.management.ObjectName` 字符串，用于注册》》。默认值为 `null`。可以使用 `tomcat.jdbc:type=org.apache.tomcat.jdbc.pool.interceptor.SlowQueryReportJmx,name=the-name-of-the-pool` 来注册对象。 |
 

### org.apache.tomcat.jdbc.pool.interceptor.ResetAbandonedTimer   

》》。这意味着如果超时为 30 秒，而你使用连接运行了 10 个 10秒的查询，那么它》就会被标为废弃，并可能依靠 `abandonWhenPercentageFull` 属性重新声明》。每次成功地在连接上执行操作或执行查询时，该拦截器就会重设签出计时器。  


## 代码范例  

其他 JDBC 用途的 Tomcat 配置范例可以参考 相关的 [Tomcat 文档](http://tomcat.apache.org/tomcat-8.0-doc/jndi-datasource-examples-howto.html)。  


### 简单的 Java

下面这个简单的范例展示了如何创建并使用数据源：  

```

```  

### 作为资源使用  

下例展示了如何为 JNDI 查找配置资源。  

```

```  


### 异步连接获取  

Tomcat JDBC 连接池支持异步连接获取，无需为池库添加任何额外线程。这是通过在数据源上添加一个方法 `Future<Connection> getConnectionAsync()` 来实现的。为了使用异步获取，必须满足两个条件：  

1. 必须把 `failQueue` 属性设为 `true`。 
2. 必须把数据源转换为 `org.apache.tomcat.jdbc.pool.DataSource`。   

下例就使用了异步获取功能：    

```

```  

### 拦截器  

对于启用、禁止或修改特定连接或其组件的功能而言，使用拦截器无疑是一种非常强大的方式。**There are many different use cases for when interceptors are useful**。默认情况下，基于性能方面的考虑，连接池是无状态的。连接池本身所插入的状态是 `defaultAutoCommit`、`defaultReadOnly`、`defaultTransactionIsolation`，或 `defaultCatalog`（如果设置了这些状态）。这 4 个状态只有在连接创建时才设置。无论这些属性是否在连接使用期间被修改，池本身都不能重置它们。  

拦截器必须扩展自 `org.apache.tomcat.jdbc.pool.JdbcInterceptor` 类。该类相当简单，你必须利用一个无参数构造函数。  

```
  public JdbcInterceptor() {
  }  
```  

当从连接池借出一个连接时，拦截器能够通过实现以下方法，初始化这一事件或以一些其他形式来响应该事件。

`public abstract void reset(ConnectionPool parent, PooledConnection con);`

上面这个方法有两个参数，一个是连接池本身的引用 `ConnectionPool parent`，一个是底层连接的引用 `PooledConnection con`。     

当调用 `java.sql.Connection` 对象上的方法时，会导致以下方法被调用：  

`public Object invoke(Object proxy, Method method, Object[] args) throws Throwable`  

`Method method` 是被调用的实际方法，`Object[] args` 是参数。通过观察下面这个非常简单的例子，我们可以解释如果当连接已经关闭时，如何让 `java.sql.Connection.close()` 的调用变得 》noop》。   

```   
  if (CLOSE_VAL==method.getName()) {
      if (isClosed()) return null; //noop for already closed.
  }
  return super.invoke(proxy,method,args);  

```

#### 池启动与停止  

当连接池开启或关闭时，你可以得到相关通知。可能每个拦截器类只通知一次，即使它是一个实例方法。也可能使用当前未连接到池中的拦截器》》》

```   
  public void poolStarted(ConnectionPool pool) {
  }

  public void poolClosed(ConnectionPool pool) {
  }

```  

当重写这些方法时，如果你扩展自 `JdbcInterceptor` 之外的类，不要忘记调用超类。

#### 配置拦截器  

拦截器可以通过 `jdbcInterceptors` 属性或 `setJdbcInterceptors` 方法来配置。拦截器也可以有属性，可以通过如下方式来配置：   

```   
  String jdbcInterceptors=
    "org.apache.tomcat.jdbc.pool.interceptor.ConnectionState(useEquals=true,fast=yes)"

```

#### 拦截器属性  

既然拦截器也有属性，那么你也可以读取其中的属性值。你可以重写 `setProperties` 方法。   

```  
  public void setProperties(Map<String, InterceptorProperty> properties) {
     super.setProperties(properties);
     final String myprop = "myprop";
     InterceptorProperty p1 = properties.get(myprop);
     if (p1!=null) {
         setMyprop(Long.parseLong(p1.getValue()));
     }
  }

```

### 获取实际的 JDBC 连接  

连接池围绕实际的连接创建包装器，为的是能够正确地》。同样，为了执行特定的功能，我们也可以在这些包装器中创建拦截器。如果不需要获取实际的连接，可以使用 `javax.sql.PooledConnection` 接口。   

```  
  Connection con = datasource.getConnection();
  Connection actual = ((javax.sql.PooledConnection)con).getConnection();

```  




## 构建  

下面利用 1.6 来构建 JDBC 连接池代码，但它也可以向后兼容到 1.5 运行时环境。为了单元测试，使用 1.6 或更高版本。  

更多的关于 JDBC 用途的 Tomcat 配置范例可参看 [Tomcat 文档]()。  

### 从源代码构建   

构建非常简单。   

构建文件位于 Tomcat 的[源代码仓库](http://svn.apache.org/viewvc/tomcat/trunk/modules/jdbc-pool/)中。  

```  

```  





















