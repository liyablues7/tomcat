# Realm 配置

## 目录  
- 快速入门
- 概述
	1. 什么是 Realm  
	2. 配置 Realm
- 常用特性  
	1. 摘要式密码
	2. 范例应用    
	3. Manager 应用
	4. Realm 日志
- 标准 Realm 实现
	1. JDBCRealm  
	2. DataSourceRealm
	3. JNDIRealm
	4. UserDatabaseRealm
	5. MemoryRealm
	6. JAASRealm
	7. CombinedRealm
	8. LockOutRealm


## 快速入门  
 
本文档介绍了如何借助一个“数据库”来配置 Tomcat ，从而实现**容器管理安全性**。所要连接的这种数据库含有用户名、密码以及用户角色。你只需知道的是，如果使用的 Web 应用含有一个或多个 `<security-constraint>` 元素，`<login-config>` 元素定义了用户验证的必需细节信息。如果你不打算使用这些功能，则可以忽略这篇文档。  

关于容器管理安全性的基础知识，可参考 [Servlet Specification (Version 2.4)](http://wiki.apache.org/tomcat/Specifications) 中的第 12 节内容。  

关于如何使用 Tomcat 中的**单点登录**（用户只需验证一次，就能够登录一个虚拟主机的所有 Web 应用）功能，请参看[该文档](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html#Single_Sign_On)。



## 概述    
### 什么是 Realm   

**Realm**（安全域）其实就是一个存储用户名和密码的“数据库”再加上一个枚举列表。“数据库”中的用户名和密码是用来验证 Web 应用（或 Web 应用集合）用户合法性的，而每一合法用户所对应的**角色**存储在枚举列表中。可以把这些角色看成是类似 UNIX 系统中的 group（分组），因为只有能够拥有特定角色的用户才能访问特定的 Web 应用资源（而不是通过对用户名列表进行枚举适配）。特定用户的用户名下可以配置多个角色。

虽然 Servlet 规范描述了一个可移植机制，使应用可以在 `web.xml` 部署描述符中声明它们的安全需求，但却没有提供一种可移植 API 来定义出 Servlet 容器与相应用户及角色信息的接口。然而，在很多情况下，非常适于将 Servlet 容器与一些已有的验证数据库或者生产环境中已存在的机制“连接”起来。因此，Tomcat 定义了一个 Java 接口（`org.apache.catalina.Realm`），通过“插入”组件来建立连接。提供了 6 种标准插件，支持与各种验证信息源的连接：   

- <a href = "#JDBCRealm">JDBCRealm</a>——通过 JDBC 驱动器来访问保存在关系型数据库中的验证信息。  
- <a href = "#DataSourceRealm">DataSourceRealm</a>——访问保存在关系型数据库中的验证信息。    
- <a href = "#JNDIRealm">JNDIRealm</a>——访问保存在 LDAP 目录服务器中的验证信息。  
- <a href = "#UserDatabaseRealm">UserDatabaseRealm</a>——访问存储在一个 UserDatabase JNDI 数据源中的认证信息，通常依赖一个 XML 文档（`conf/tomcat-users.xml`）。  
- <a href = "#MemoryRealm">MemoryRealm</a>——访问存储在一个内存中对象集合中的认证信息，通过 XML 文档初始化（`conf/tomcat-users.xml`）。  
- <a href = "#JAASRealm">JAASRealm</a>——通过 Java 认证与授权服务（JAAS）架构来获取认证信息。    


另外，还可以编写自定义 `Realm` 实现，将其整合到 Tomcat 中，只需这样做：  

- 实现 `org.apache.catalina.Realm` 接口。  
- 将编译好的 realm 放到 $CATALINA_HOME/lib 中。  
- 声明自定义realm，具体方法详见<a href = "#configRealm">“配置Realm”一节</a>。  
- 在 [MBeans 描述符文件](》》需要连接相关文档)中声明自定义realm。        


### <a name = "configRealm">配置 Realm</a>       


在详细介绍标准 Realm 实现之前，简要了解 Realm 的配置方式是很关键的一步。大体来说，就是需要在`conf/server.xml` 配置文件中添加一个 XML 元素，如下所示：  

```
<Realm className="... class name for this implementation"
       ... other attributes for this implementation .../>  
```

`<Realm>` 可以嵌入以下任何一种 `Container` 元素中。Realm 元素的位置至关重要，它会对 Realm 的“范围”（比如说哪个 Web 应用能够共享同一验证信息）有直接的影响。   

- **`<Engine>` 元素** 内嵌入该元素中的这种 Realm 元素可以被**所有**虚拟主机上的**所有** Web 应用所共享，除非该 Realm 元素被内嵌入下属 `<Host>` 或 `<Context>` 元素的 Realm 元素所覆盖。  
- **`<Host>` 元素** 内嵌入该元素中的这种 Realm 元素可以被**这一**虚拟主机上的**所有** Web 应用所共享。除非该 Realm 元素被内嵌入下属 `<Context>` 元素的 Realm 元素所覆盖。    
- **`<Context>` 元素** 内嵌入该元素中的这种 Realm 元素**只能**被这一 Web 应用所使用。   

## 常用特性  

### <a name = "digestedPass">摘要式密码</a>  

对于每种标准 `Realm` 实现来说，用户的密码默认都是以明文方式保存的。在很多情况下，这种方式都非常糟糕，即使是一般的用户也能收集到足够的验证信息，从而以其他用户的信息成功登录。为了避免这种情况的发生，标准 `Realm` 实现支持一种对用户密码进行**摘要式处理**的机制，它能以无法轻易破解的形式对存储的密码进行加密处理，同时保证`Realm` 实现仍能使用这种加密后的密码进行验证。  

在标准的 Realm 验证时，会将存储的密码与用户所提供的密码进行比对，这时，我们可以通过指定 `<Realm>` 元素中的 `digest` 属性来选择摘要式密码。该属性值必须是一种`java.security.MessageDigest` 类所支持的摘要式算法（SHA、MD2、或 MD5）。当你选择该属性值时，存储在 Realm 中的密码内容必须是明文格式，随后它将被你所指定的算法进行摘要式加密。  

在调用 Realm 的 `authenticate()` 方法后，用户所提供的明文密码同样也会利用上述你所指定的加密算法进行加密，加密结果与 Realm 的返回值相比较。如果两者相等，则表明原始密码的明文形式更用户所提供的密码完全等同，因此该用户身份验证成功。  

可以采用以下两种比较便利的方法来计算明文密码的摘要值：   

- 如果应用需要动态计算摘要式密码，调用 `org.apache.catalina.realm.RealmBase` 类的静态 `Digest()` 方法，传入明文密码和摘要式算法名称及字符编码方案。该方法返回摘要式密码。
- 如果想执行命令行工具来计算摘要式密码，只需执行：   
	`CATALINA_HOME/bin/digest.[bat|sh] -a {algorithm} {cleartext-password}`  
	标准输出将返回明文密码的摘要式形式。   
	
如果使用 DIGEST 验证的摘要式密码，用来生成摘要密码的明文密码则将有所不同，而且必须使用一次不加盐的 MD5 算法。对应到上面的范例，那就是必须把 `{cleartext-password}` 替换成 `{username}:{realm}:{cleartext-password}`。再比如说，在一个开发环境中，可能采用这种形式：`testUser:Authentication required:testPassword`。`{realm}` 的值取自 Web 应用 `<login-config>` 的 `<realm-name>` 元素。如果没有在 web.xml 中指定，则使用默认的`Authentication required`。     

若要使用非平台默认编码的用户名和（或）密码，则命令如下：     

`CATALINA_HOME/bin/digest.[bat|sh] -a {algorithm} -e {encoding} {input}`

但需要注意的是，一定要确保输入正确地传入摘要。摘要返回 `{input}:{digest}`。如果输入在返回时出现损坏，摘要则将无效。   

摘要的输出格式为 `{salt}${iterations}${digest}`。如果盐的长度为 0，迭代次数为 1，则输出将简化为 `{digest}`。   

`CATALINA_HOME/bin/digest.[bat|sh]` 的完整格式如下：  

```
CATALINA_HOME/bin/digest.[bat|sh] [-a <algorithm>] [-e <encoding>]
        [-i <iterations>] [-s <salt-length>] [-k <key-length>]
        [-h <handler-class-name>] <credentials>  
```  

- `-a` 用来生成存储凭证的算法。如未指定，将使用凭证处理器（CredentialHandler）的默认值，如果认证处理器和算法均未指定，则使用默认值 `SHA-512`。   
- `-e` 指定用于任何必要的字节与字符间转换的字符编码方案。如未指定，使用系统默认的字符编码方案 `Charset#defaultCharset()`。
- `-i` 生成存储的凭证时所使用的迭代次数。如未指定，使用 CredentialHandler 的默认值。 
- `-s` 生成并存储到认证中的盐的长度（字节）。如未指定，使用 CredentialHandler 的默认值。 
- `-k` （生成凭证时，如果随带创建了）键的长度（位）。如未指定，则使用 CredentialHandler 的默认值 
- `-h` CredentialHandler 使用的完整类名。如未指定，则轮流使用内建的凭证处理器（`MessageDigestCredentialHandler`，然后是 `SecretKeyCredentialHandler`），将使用第一个接受指定算法的凭证处理器。


### 范例应用    

Tomcat 自带的范例应用中包含一个受到安全限制保护的区域，使用表单式登录方式。为了访问它，在你的浏览器地址栏中输入 `http://localhost:8080/examples/jsp/security/protected/`，并使用 [UserDatabaseRealm](http://tomcat.apache.org/tomcat-8.0-doc/realm-howto.html#UserDatabaseRealm) 默认的用户名和密码进行登录。  


### Manager 应用    

如果你希望使用 [Manager 应用](》链接至Manager章节》)在一个运行的 Tomcat 安装上来部署或取消部署 Web 应用，那么**必须**在一个选定的 Realm 实现上，将 **manager-gui** 角色添加到至少一个用户名上。这是因为 Manager 自身使用一个安全限制，要想在该应用的 HTML 界面中访问请求 URI，就必须要有 manager-gui 角色。  

出于安全性考虑，默认情况下，Realm 中的用户名（比如使用 `conf/tomcat-users.xml`）没有被分配 manager-gui 角色。因此，用户起初无法使用这个功能，除非 Tomcat 管理员特意将这一角色分配给他们。   


### Realm 日志    

Realm 的容器（Context、Host 及 Engine）所对应的日志配置文件将记录 `Realm` 所记录下的调试和异常信息。   




## 标准 Realm 实现    

### <a name ="JDBCRealm">JDBCRealm</a> 

#### 简介   

**JDBCRealm**  是 Tomcat `Realm` 接口的一种实现，它通过 JDBC 驱动程序在关系型数据库中查找用户。只要数据库结构符合下列要求，你可以通过大量的配置来灵活地修改现有的表与列名。  

- 必须有一张**用户表**（*users table*）。它包含着一个由 `Realm` 所能识别的所有合法用户所构成的行。  
- **用户表**必须至少包含两列（当然，如果现有应用确实需要，则同样也可以包含更多的列）：  
	- 用户名。当用户登录时，能被 Tomcat 识别的用户名。   
	- 密码。当用户登录时，能被 Tomcat 所识别的密码。该列中的值可能是明文，也可能是摘要式密码，稍后详述。   
- 必须有一张**用户角色表**（*user roles table*）。该表包含一个角色行，包含着可能指定给特定用户的每个合法角色。一个用户可以没有角色，也可以有一个或多个角色，这都是合法的。     
- **用户角色表** 至少应包含两列（如果现有应用确实需要，则也可以包含更多的列）：   
	- 用户名。Tomcat 所能识别的用户名（与用户表中指定的值相同）。  
	- 用户所对应的合法角色名。     

#### 快速入门    

为了设置 Tomcat 从而使用 JDBCRealm，需要执行以下步骤：  

1. 在数据库中创建符合上述规范的表与列。  
2. 配置一个 Tomcat 使用的数据库用户名与密码，并且至少有只读权限（Tomcat 永远都不会去修改那些表中的数据）。  
3. 将用到的 JDBC 驱动程序复制到 `$CATALINA_HOME/lib` 目录中。注意**只能**识别 JAR 文件！  
4. 在 `$CATALINA_BASE/conf/server.xml` 目录中设置一个 `<Realm>` 元素。这一点下文将会详细叙述。  
5. 如果 Tomcat 处于运行状态，则重启它。   


#### Realm 元素属性    

如上所述，为了配置 JDBCRealm，需要创建一个 `Realm` 元素，并把它放在 `$CATALINA_BASE/conf/server.xml` 文件中。JDBCRealm 的属性都定义在 [Realm](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html) 配置文档中。


#### 范例    

下面这个 SQL 脚本范例创建了我们所需的表（根据你所用的数据库，可以相应修改其中的语法）。  

```
create table users (
  user_name         varchar(15) not null primary key,
  user_pass         varchar(15) not null
);

create table user_roles (
  user_name         varchar(15) not null,
  role_name         varchar(15) not null,
  primary key (user_name, role_name)
);

``` 

`Realm` 元素包含在默认的 `$CATALINA_BASE/conf/server.xml` 文件中（被注释掉了）。在下面的范例中，有一个名为 authority 的数据库，它包含上述创建的表，通过用户名“dbuser”和密码“dbpass”进行访问。  

```
<Realm className="org.apache.catalina.realm.JDBCRealm"
      driverName="org.gjt.mm.mysql.Driver"
   connectionURL="jdbc:mysql://localhost/authority?user=dbuser&amp;password=dbpass"
       userTable="users" userNameCol="user_name" userCredCol="user_pass"
   userRoleTable="user_roles" roleNameCol="role_name"/>


```

#### 特别注意事项  

JDBCRealm 必须遵循以下规则：     

- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法，从而使任何对数据库的即时修改（新用户、密码或角色改变，等等）都能立即生效。  
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    
- 应用负责管理**users**（用户表）和**user roles**（用户角色表）中的信息。Tomcat 没有提供任何内置功能来维护这两种表。     


### <a name ="DataSourceRealm">DataSourceRealm</a>     

#### 简介   
 
**DataSourceRealm** 是 Tomcat `Realm` 接口的一种实现，它通过一个 JNDI 命名的 JDBC 数据源在关系型数据库中查找用户。只要数据库结构符合下列要求，你可以通过大量的配置来灵活地修改现有的表与列名。    

- 必须有一张**用户表**（*users table*）。它包含着一个由 `Realm` 所能识别的所有合法用户所构成的行。  
- **用户表**必须至少包含两列（当然，如果现有应用确实需要，则同样也可以包含更多的列）：  
	- 用户名。当用户登录时，能被 Tomcat 识别的用户名。   
	- 密码。当用户登录时，能被 Tomcat 所识别的密码。该列中的值可能是明文，也可能是摘要式密码，稍后详述。   
- 必须有一张**用户角色表**（*user roles table*）。该表包含一个角色行，包含着可能指定给特定用户的每个合法角色。一个用户可以没有角色，也可以有一个或多个角色，这都是合法的。     
- **用户角色表** 至少应包含两列（如果现有应用确实需要，则也可以包含更多的列）：   
	- 用户名。Tomcat 所能识别的用户名（与用户表中指定的值相同）。  
	- 用户所对应的合法角色名。     

#### 快速入门  

为了设置 Tomcat 从而使用 DataSourceRealm，需要执行以下步骤：  

1. 在数据库中创建符合上述规范的表与列。  
2. 配置一个 Tomcat 使用的数据库用户名与密码，并且至少有只读权限（Tomcat 永远都不会去修改那些表中的数据）。  
3. 为数据库配置一个 JNDI 命名的 JDBC DataSource。详情可参考[JNDI DataSource Example HOW-TO》应该链接至相应中文页面》](http://tomcat.apache.org/tomcat-8.0-doc/jndi-datasource-examples-howto.html)。
4. 在 `$CATALINA_BASE/conf/server.xml` 目录中设置一个 `<Realm>` 元素。这一点下文将会详细叙述。  
5. 如果 Tomcat 处于运行状态，则重启它。     

#### 范例  

下面这个 SQL 脚本范例创建了我们所需的表（根据你所用的数据库，可以相应修改其中的语法）。  

```
create table users (
  user_name         varchar(15) not null primary key,
  user_pass         varchar(15) not null
);

create table user_roles (
  user_name         varchar(15) not null,
  role_name         varchar(15) not null,
  primary key (user_name, role_name)
);
  
```  

在下面的范例中，有一个名为 authority 的 MySQL 数据库，它包含上述创建的表，通过名为 “java:/comp/env/jdbc/authority” 的 JNDI  命名的 JDBC 数据源来访问。

```
<Realm className="org.apache.catalina.realm.DataSourceRealm"
   dataSourceName="jdbc/authority"
   userTable="users" userNameCol="user_name" userCredCol="user_pass"
   userRoleTable="user_roles" roleNameCol="role_name"/>

```  

#### 特别注意事项   

使用 DataSourceRealm 时必须遵守下列规则：  

- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法，从而使任何对数据库的即时修改（新用户、密码或角色改变，等等）都能立即生效。  
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    
- 应用负责管理**users**（用户表）和**user roles**（用户角色表）中的信息。Tomcat 没有提供任何内置功能来维护这两种表。       


### <a name ="JNDIRealm">JNDIRealm</a>     

#### 简介  

**JNDIRealm** 是 Tomcat `Realm` 接口的一种实现，通过一个 JNDI 提供者<sup>1</sup>在 LDAP 目录服务器中查找用户。realm 支持大量的方法来使用认证目录。     

<sup>通常是可以使用 JNDI API 类的标准 LDAP 提供者。</sup>  

##### a. 连接目录  

realm 与目录服务器的连接是通过 **connectionURL** 配置属性来定义的。这个 URL 的格式是通过 JNDI 提供者来定义的。它通常是个 LDAP URL，指定了所要连接的目录服务器的域名，另外还（可选择）指定了所需的根命名上下文的端口和唯一名称（DN）。   

如果有多个提供者，则可以配置 **alternateURL**。如果一个套接字连接无法传递给提供者 **connectionURL**，则会换用 **alternateURL**。   

当通过创建连接来搜索目录，获取用户及角色信息时，realm 会利用 **connectionName** 和 **connectionPassword** 这两个属性所指定的用户名和密码在目录上进行自我认证。如果未指定这两个属性，则创立的连接是匿名连接，这种连接适用于大多数情况。    


##### b. 选择用户目录项  

在目录中，每个可被认证的用户都必须表示为独立的项，这种独立项对应着由属性 **connectionURL** 定义的初始 `DirContext` 中的元素。这种用户项必须有一个包含认证所需用户名的属性。  

每个用户项的唯一性名称（DN）通常含有用于认证的用户名。在这种情况下，**userPattern** 属性可以用来指定 DN，其中的 `{0}` 代表用户名应该被替换的位置。  

realm 必须搜索目录来寻找一个包含用户名的唯一项，可用下列属性来配置搜索：  

- **userBase** 用户子树的基准项。如果未指定，则搜索基准为顶级元素。  
- **userSubtree** 用户子树，也就是搜索范围。如果希望搜索以**userBase** 项为基准的整个子树，则使用 `true`；默认值为 `false`，只对顶级元素进行搜索。  
- **userSearch** 指定替代用户名之后所使用的 LDAP 搜索过滤器的模式。   

##### c. 用户认证  

- **绑定模式**   
	
	默认情况下，realm 会利用用户项的 DN 与用户所提供的密码，将用户绑定到目录上。如果成功执行了这种简单的绑定，那么就可以认为用户认证成功。
	
	出于安全考虑，目录可能保存的是用户的摘要式密码，而非明文密码（参看<a href= "#digestedPass">摘要式密码</a>以获知详情）。在这种情况下，在绑定过程中，目录会自动将用户所提供的明文密码加密为正确的摘要式密码，以便后续和存储的摘要式密码进行比对。然而在绑定过程中，realm 并不参与处理摘要式密码。不会用到 **digest** 属性，如果设置了该属性，也会被自动忽略。  
	
- **对比模式**  

	另外一种方法是，realm 从目录中获取存储的密码，然后将其与用户所提供的值进行比对。配置方法是，在包含密码的用户项中，将 **userPassword** 属性设为目录属性名。  
	
	对比模式的缺点在于：首先，`connectionName` 和 `connectionPassword` 属性必须配置成允许 realm 读取目录中的用户密码。出于安全考虑，这是一种不可取的做法。事实上，很多目录实现甚至都不允许目录管理器读取密码。其次，realm 必须自己处理摘要式密码，包括要设置所使用的具体算法、在目录中表示密码散列值的方式。但是，realm 可能有时又要访问存储的密码，比如为了支持 HTTP 摘要式访问认证（HTTP Digest Access Authentication，RFC 2069）。（注意，HTTP 摘要式访问认证不同于之前讨论过的在库中存储密码摘要的方式。）  
	
##### d. 赋予用户角色  

Realm 支持两种方法来表示目录中的角色：  

- **将角色显式表示为目录项**  

	通过明确的目录项来表示角色。角色项通常是一个 LDAP 分组项，该分组项的一个属性包含角色名称，另一属性值则是拥有该角色的用户的 DN 名或用户名。下列属性配置了一个目录搜索来寻找与认证用户相关的角色名。  
	
	- **roleBase** 角色搜索的基准项。如未指定，则基准项为顶级目录上下文。  
	- **roleSubtree** 搜索范围。如果希望搜索以 `roleBase` 为基准项的整个子树，则设为 `true`。默认值为 `false`，请求一个只包含顶级元素的单一搜索。  
	- **roleSearch** 用于选择角色项的 LDAP 搜索过滤器。它还可以（可选择）包含用于唯一名称的模式替换 `{0}`、用户名的模式替换 `{1}`，以及用户目录项属性的模式替换 `{2}`。使用 **userRoleAttribute** 来指定提供 `{2}` 值的属性名。  
	- **roleName** 包含角色名称的角色项属性。  
	- **roleNested** 启用内嵌角色。如果希望在角色中内嵌角色，则设为 `true`。如果配置了该属性，每一个新近找到的 roleName 和 DN 都将用于递归式的新角色搜索。默认值为 `false`。   

- **将角色表示为用户项属性**   

	将角色名称保存为用户目录项中的一个属性值。使用 **userRoleName** 来指定该属性名称。  
	
当然，也可以综合使用这两种方法来表示角色。  

#### 快速入门  

为了配置 Tomcat 使用 JNDIRealm，需要下列步骤：  

1. 确保目录服务器配置中的模式符合上文所列出的要求。  
2. 必要时可以为 Tomcat 配置一个用户名和密码，对上文所提到的信息用于只读访问权限（Tomcat 永远不会修改该信息。）  
3. 按照下文的方法，在 `$CATALINA_BASE/conf/server.xml` 文件中设置一个 `<Realm>` 元素。   
4. 如果 Tomcat 已经运行，则重启它。  

#### Realm 元素属性   

如上所述，为了配置 JDBCRealm，需要创建一个 `Realm` 元素，并把它放在 `$CATALINA_BASE/conf/server.xml` 文件中。JDBCRealm 的属性都定义在 [Realm](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html) 配置文档中。  

#### 范例    

在目录服务器上创建适合的模式超出了本文档的讲解范围，因为这是跟每个目录服务器的实现密切相关的。在下面的实例中，我们将假定使用的是 OpenLDAP 目录服务器的一个分发版（2.0.11 版或更新版本，可从[http://www.openldap.org](http://www.openldap.org)处下载）。假设 `slapd.conf` 文件包含下列设置（除了其他设置之外）。   

```  
database ldbm
suffix dc="mycompany",dc="com"
rootdn "cn=Manager,dc=mycompany,dc=com"
rootpw secret

```


我们还假定 `connectionURL`，使目录服务器与 Tomcat 运行在同一台机器上。要想了解如何配置及使用 JNDI LDAP 提供者的详细信息，请参看 [http://docs.oracle.com/javase/7/docs/technotes/guides/jndi/index.html](http://docs.oracle.com/javase/7/docs/technotes/guides/jndi/index.html)。  

接下来，假定利用如下所示的元素（以 LDIF 格式）来填充目录服务器。  

```  
# Define top-level entry
dn: dc=mycompany,dc=com
objectClass: dcObject
dc:mycompany

# Define an entry to contain people
# searches for users are based on this entry
dn: ou=people,dc=mycompany,dc=com
objectClass: organizationalUnit
ou: people

# Define a user entry for Janet Jones
dn: uid=jjones,ou=people,dc=mycompany,dc=com
objectClass: inetOrgPerson
uid: jjones
sn: jones
cn: janet jones
mail: j.jones@mycompany.com
userPassword: janet

# Define a user entry for Fred Bloggs
dn: uid=fbloggs,ou=people,dc=mycompany,dc=com
objectClass: inetOrgPerson
uid: fbloggs
sn: bloggs
cn: fred bloggs
mail: f.bloggs@mycompany.com
userPassword: fred

# Define an entry to contain LDAP groups
# searches for roles are based on this entry
dn: ou=groups,dc=mycompany,dc=com
objectClass: organizationalUnit
ou: groups

# Define an entry for the "tomcat" role
dn: cn=tomcat,ou=groups,dc=mycompany,dc=com
objectClass: groupOfUniqueNames
cn: tomcat
uniqueMember: uid=jjones,ou=people,dc=mycompany,dc=com
uniqueMember: uid=fbloggs,ou=people,dc=mycompany,dc=com

# Define an entry for the "role1" role
dn: cn=role1,ou=groups,dc=mycompany,dc=com
objectClass: groupOfUniqueNames
cn: role1
uniqueMember: uid=fbloggs,ou=people,dc=mycompany,dc=com

```

OpenLDAP 服务器。假定用户使用他们的 uid（比如说 jjones）登录应用，匿名连接已经足够可以搜索目录并获取角色信息了：  

```  
<Realm   className="org.apache.catalina.realm.JNDIRealm"
     connectionURL="ldap://localhost:389"
       userPattern="uid={0},ou=people,dc=mycompany,dc=com"
          roleBase="ou=groups,dc=mycompany,dc=com"
          roleName="cn"
        roleSearch="(uniqueMember={0})"
/>

```


利用这种配置，通过在 `userPattern` 替换用户名，realm 能够确定用户的 DN，然后利用这个 DN 和取自用户的密码将用户绑定到目录中，从而验证用户身份，然后搜索整个目录服务器来找寻用户角色。  

现在假定希望用户输入电子邮件地址（而不是用户 id）。在这种情况下，realm 必须搜索目录找到用户项。当用户项被保存在多个子树中，而这些子树可能分别对应不同的组织单位或企业位置时，可能必须执行一个搜索。   

另外，假设除了分组项之外，你还想用用户项的属性来保存角色，那么在这种情况下，Janet Jones 对应的项可能如下所示：  

```
dn: uid=jjones,ou=people,dc=mycompany,dc=com
objectClass: inetOrgPerson
uid: jjones
sn: jones
cn: janet jones
mail: j.jones@mycompany.com
memberOf: role2
memberOf: role3
userPassword: janet
  
```

这个 realm 配置必须满足以下新要求：  

```  
<Realm   className="org.apache.catalina.realm.JNDIRealm"
     connectionURL="ldap://localhost:389"
          userBase="ou=people,dc=mycompany,dc=com"
        userSearch="(mail={0})"
      userRoleName="memberOf"
          roleBase="ou=groups,dc=mycompany,dc=com"
          roleName="cn"
        roleSearch="(uniqueMember={0})"
/>

```

当 Janet Jones 用她的电子邮件 j.jones@mycompany.com 登录时，realm 会搜索目录，寻找带有该电邮值的唯一项，并尝试利用给定密码来绑定到目录：`uid=jjones,ou=people,dc=mycompany,dc=com`。如果验证成功，该用户将被赋予以下三个角色："role2" 与 "role3"，她的目录项中的 `memberOf` 属性值；"tomcat"，她作为成员存在的唯一分组项中的 `cn` 属性值。  

最后，为了验证用户，我们必须从目录中获取密码并在 realm 中执行本地比对，将 realm 按照如下方式来配置：  

```  
<Realm   className="org.apache.catalina.realm.JNDIRealm"
    connectionName="cn=Manager,dc=mycompany,dc=com"
connectionPassword="secret"
     connectionURL="ldap://localhost:389"
      userPassword="userPassword"
       userPattern="uid={0},ou=people,dc=mycompany,dc=com"
          roleBase="ou=groups,dc=mycompany,dc=com"
          roleName="cn"
        roleSearch="(uniqueMember={0})"
/>

```

但是，正如之前所讨论的那样，往往应该优先考虑默认的绑定模式。   

#### 特别注意事项  

使用 JNDIRealm 需要遵循以下规则：  

- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法，从而使任何对数据库的即时修改（新用户、密码或角色改变，等等）都能立即生效。  
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    
- 应用负责管理**users**（用户表）和**user roles**（用户角色表）中的信息。Tomcat 没有提供任何内置功能来维护这两种表。       









### <a name = "UserDatabaseRealm">UserDatabaseRealm</a>   


**UserDatabaseRealm** 是 Tomcat **Realm** 接口的一种实现，使用 JNDI 资源来存储用户信息。默认，JNDI 资源是通过一个 XML 文件来提供支持的。它并不是针对大规模生产环境用途而设计的。在启动时，UserDatabaseRealm 会从一个 XML 文档中加载所有用户以及他们角色的信息（该 XML 文档默认位于 `$CATALINA_BASE/conf/tomcat-users.xml`。）用户、密码以及相应角色通常可利用 JMX 进行动态编辑，更改结果会加以保存并立刻反映在 XML 文档中。  


#### Realm 元素属性  

跟<a href = "#configRealm">之前讨论</a>的一样，为了配置 UserDatabaseRealm，需要在 `$CATALINA_BASE/conf/server.xml` 中创建 `<Realm>` 元素。关于 UserDatabaseRealm 中的属性定义可参看 [Realm 配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html)。  

#### 用户文件格式  

用户文件使用的格式与 <a href= "#userfileformatofMemoryRealm">MemoryRealm</a>所使用的相同。   

#### 范例

默认的 Tomcat 安装已经配置了内嵌在 `<Engine>` 元素中的 UserDatabaseRealm，因而可以将其应用于所有的虚拟主机和 Web 应用中。默认的 `conf/tomcat-users.xml` 文件内容为：  

```   
<tomcat-users>
  <user username="tomcat" password="tomcat" roles="tomcat" />
  <user username="role1"  password="tomcat" roles="role1"  />
  <user username="both"   password="tomcat" roles="tomcat,role1" />
</tomcat-users>

```

#### 特别注意事项  

使用 UserDatabaseRealm 需要遵循以下规则：  

- 当 Tomcat 首次启动时，它会从用户文件中加载所有已定义的用户及其相关信息。假如对该用户文件中的数据进行修改，则只有重启 Tomcat 后才能生效。这些修改并不是通过 UserDatabase 数据源来完成的，是由 Tomcat 所提供的通过 JMX 访问的 MBean 来实现的。  
- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法。  
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    



  

### <a name= "MemoryRealm">MemoryRealm</a>    

#### 简介  


**MemoryRealm** 是一种对 Tomcat 的 Realm 接口的简单演示实现，并不是针对生产环境而设计的。在启动时，MemoryRealm 会从 XML 文档中加载所有的用户信息及其相关的角色信息（默认该文档位于 `$CATALINA_BASE/conf/tomcat-users.xml`）。只有重启 Tomcat 才能使对该文件作出的修改生效。  

#### Realm 元素属性  

跟<a href = "#configRealm">之前讨论</a>的一样，为了配置 MemoryRealm，需要在 `$CATALINA_BASE/conf/server.xml` 中创建 `<Realm>` 元素。关于 MemoryRealm 中的属性定义可参看 [Realm 配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html)。  


#### <a name = "userfileformatofMemoryRealm">用户文件格式</a>  

用户文件包含下列属性。默认情况下，`conf/tomcat-users.xml` 必须是一个 XML 文件，并且带有一个根元素：`<tomcat-users>`。每一个有效用户都有一个内嵌在根元素中的 `<user>` 元素。  

- **name** 用户登录所用的用户名。  
- **password** 用户登录所用的密码。如果 `<Realm>` 元素中没有设置 `digest` 属性，则采用明文密码，否则就设置为摘要式密码，如<a href = "#digestedPass">之前讨论</a>的那样。  
- **roles** 以逗号分隔的用户角色名列表。  

#### 特别注意事项  

使用 MemoryRealm 需要注意以下规则：  

- 当 Tomcat 首次启动时，它会从用户文件中加载所有已定义的用户及其相关信息。假如对该用户文件中的数据进行修改，则只有重启 Tomcat 后才能生效。  
- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法。    
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    
- 应用负责管理**users**（用户表）和**user roles**（用户角色表）中的信息。Tomcat 没有提供任何内置功能来维护这两种表。       




### <a name = "JAASRealm">JAASRealm</a>  

#### 简介  

**JAASRealm** 是 Tomcat 的 `Realm` 接口的一种实现，通过 Java Authentication & Authorization Service（JAAS，Java身份验证与授权服务）架构来实现对用户身份的验证。JAAS 架构现已加入到标准的 Java SE API 中。   

通过 JAASRealm，开发者实际上可以将任何安全的 Realm 与 Tomcat 的 CMA 一起组合使用。 

JAASRealm 是 Tomcat 针对基于 JAAS 的 J2EE 1.4 的 J2EE 认证框架的原型实现，基于 [JCP Specification Request 196](https://www.jcp.org/en/jsr/detail?id=196)，从而能够增强容器管理安全性，并且能促进“可插拔的”认证机制，该认证机制能够实现与容器的无关性。


根据 JAAS 登录模块和准则（参见 `javax.security.auth.spi.LoginModule` 与 `javax.security.Principal` 的相关说明），你可以自定义安全机制，或者将第三方的安全机制与 Tomcat 所实现的 CMA 相集成。  


#### 快速入门  

为了利用自定义的 JAAS 登录模块使用 JAASRealm，需要执行如下步骤：  

1. 编写自己的 JAAS 登录模块。在开发自定义登录模块时，将通过 JAAS 登录上下文对基于 JAAS <sup>2</sup>的 User 和 Role 类管理。注意，JAASRealm 内建的 `CallbackHandler` 目前只能识别 `NameCallback` 和 `PasswordCallback`。  

	<sup>2. 详情请参看 [JAAS 认证教程](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jaas/tutorials/GeneralAcnOnly.html) 与 [JAAS 登录模块开发教程](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jaas/JAASLMDevGuide.html)。</sup>  
	
2. 尽管 JAAS 并未明确指定，但你也应该为用户和角色创建不同的类来加以区分，它们都应该扩展自 `javax.security.Principal`，从而使 Tomcat 明白从登录模块中返回的规则究竟是用户还是角色（参看 `org.apache.catalina.realm.JAASRealm` 相关描述）。不管怎样，第一个返回的规则**总**被认为是用户规则。  
3. 将编译好的类指定在 Tomcat 的类路径中。 
4. 为 Java 建立一个 login.config 文件（参见 [JAAS LoginConfig 文件](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jaas/tutorials/LoginConfigFile.html)）。将其位置指定给 JVM，从而便于 Tomcat 明确它的位置。例如，设置如下环境变量：  
	`JAVA_OPTS=$JAVA_OPTS -Djava.security.auth.login.config==$CATALINA_BASE/conf/jaas.config`  
	
5. 为了保护一些资源，在 web.xml 中配置安全限制。  
6. 在 server.xml 中配置 JAASRealm 模块。  
7. 重启 Tomcat（如果它正在运行）。  

#### Realm 元素属性  

在上述步骤中，为了配置步骤 6 以上的 JAASRealm，需要创建一个 `<Realm>` 元素，并将其内嵌在 `<Engine>` 元素中的 `$CATALINA_BASE/conf/server.xml` 文件内。关于 JAASRealm 中的属性定义可参看 [Realm 配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html)。    

#### 范例  

下例是 server.xml 中的一截代码段：  

```   
<Realm className="org.apache.catalina.realm.JAASRealm"
                appName="MyFooRealm"
    userClassNames="org.foobar.realm.FooUser"
     roleClassNames="org.foobar.realm.FooRole"/>

```

完全由登录模块负责创建并保存用于表示用户规则的 User 与 Role 对象（`javax.security.auth.Subject`）。如果登录模块不仅无法创建用户对象，而且也无法抛出登录异常，Tomcat CMA 就会失去作用，所在页面就会变成 `http://localhost:8080/myapp/j_security_check` 或其他未指明的页面。


JAAS 方法具有双重的灵活性：  

- 你可以在自定义的登录模块后台执行任何所需的进程。     
- 通过改变配置以及重启服务器，你可以插入一个完全不同的登录模块，不需要对应用做出任何改动。  


#### 特别注意事项  

- 当用户首次访问一个受保护资源时，Tomcat 会调用这一 `Realm` 的 `authenticate()` 方法。     
- 一旦用户认证成功，在登录后，该用户（及其相应角色）就将缓存在 Tomcat 中。（对于以表单形式的认证，这意味着直到会话超时或者无效才会过期；对于基本形式的验证，意味着直到用户关闭浏览器才会过期。）在会话序列化期间**不会**保存或重置缓存的用户。对已认证用户的数据库信息进行的任何改动都**不会**生效，直到该用户下次登录。    
- 和其他 Realm 实现一样，如果 `server.xml` 中的 `<Realm>` 元素包含一个 `digest` 属性，则支持摘要式密码。JAASRealm 的 `CallbackHandler` 将先于将密码传回 `LoginModule` 之前，对密码进行摘要式处理。   

### CombinedRealm    

#### 简介  

**CombinedRealm** 是一种 Tomcat 的 Realm 实现，通过一个或多个子 Realm 进行用户验证。  

通过 CombinedRealm，开发者能够将多个 Realm（同一或不同类型） 组合起来使用，从而用于验证多种数据源，而且万一当其中一个 Realm 失败，或其他一些操作需要多个 Realm时，它还能提供回滚处理。  

子 Realm 是通过在定义 CombineRealm 的 `Realm` 元素中内嵌 `Realm` 元素来实现的。验证操作会按照 Realm 元素的叠加顺序来逐个进行。对逐个 Realm 进行验证，从而就能充分证明用户的身份。  


#### Realm 元素属性  

为了配置 CombinedRealm，需要创建一个 `<Realm>` 元素，并将其内嵌在 `<Engine>` 或 `<Host>` 元素中的 `$CATALINA_BASE/conf/server.xml` 文件内。同样，你也可以将其内嵌到 `context.xml` 文件下的 `<Context>` 节点。   

#### 范例  

下面是 server.xml 中的一段代码，综合使用了 UserDatabaseRealm 和 DataSourceRealm：  

```  
<Realm className="org.apache.catalina.realm.CombinedRealm" >
   <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
             resourceName="UserDatabase"/>
   <Realm className="org.apache.catalina.realm.DataSourceRealm"
             dataSourceName="jdbc/authority"
             userTable="users" userNameCol="user_name" userCredCol="user_pass"
             userRoleTable="user_roles" roleNameCol="role_name"/>
</Realm>

```


### LockOutRealm  

#### 简介  

**LockOutRealm** 是一个 Tomcat 的 Realm 实现，它扩展了 CombinedRealm，假如在某一段时间内出现很多验证失败，则它能够提供锁定用户的功能。     

为了确保操作的正确性，该 Realm 允许出现较合理的同步。


该 Realm 并不需要对底层的 Realm 或与其相关的用户存储机制进行任何改动。它会记录失败的登录，包括那些因为用户不存在的登录。为了防止无效用户通过精心设计的请求而实施的 DOS 攻击（从而造成缓存增加），没有通过验证的用户所在列表的容量受到了严格的限制。  

子 Realm 是通过在定义 LockOutRealm 的 `Realm` 元素中内嵌 `Realm` 元素来实现的。验证操作会按照 Realm 元素的叠加顺序来逐个进行。对逐个 Realm 进行验证，从而就能充分证明用户的身份。  


#### Realm 元素属性

为了配置 CombinedRealm，需要创建一个 `<Realm>` 元素，并将其内嵌在 `<Engine>` 或 `<Host>` 元素中的 `$CATALINA_BASE/conf/server.xml` 文件内。同样，你也可以将其内嵌到 `context.xml` 文件下的 `<Context>` 节点。关于 LockOutRealm 中的属性定义可参看 [Realm 配置文档](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html)。    


#### 范例  

下面是 server.xml 中的一段代码，为 UserDatabaseRealm 添加了锁定功能。


```
<Realm className="org.apache.catalina.realm.LockOutRealm" >
   <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
             resourceName="UserDatabase"/>
</Realm>
   
```
 








   






