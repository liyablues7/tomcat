# 安装 Tomcat 

## 目录  

- 本章概述  
- Windows 系统下的安装  
- UNIX 守护进程   


## 本章概述  

可利用多种方法把 Tomcat 安装到不同的平台上。关于 Tomcat 安装方面的重要文档是 [RUNNING.txt](http://tomcat.apache.org/tomcat-8.0-doc/RUNNING.txt)。如果本节内容尚未能解决你的某些困惑，建议查阅该文档获取帮助。   


## Windows 系统下的安装     

利用 Windows 安装程序可以轻松地在 Windows 系统下安装 Tomcat。无论是在界面还是在功能上，Windows 安装程序都很类似于其他向导式安装程序，只需在以下几个方面稍加注意：     

- **以 Windows 服务的形式进行安装** 利用多种配置，Tomcat 可以安装为 Windows 服务。在组件页面勾选复选框，将服务设置为“自动”启动，这样当 Windows 启动时，Tomcat 也随即启动。为了获取最佳的安全性，可以把该服务作为单独用户来运行，并降低权限（详情参看 Windows 服务管理工具及其相关文档）   
- **Java 位置** 为了运行服务，安装程序通常会提供默认的 JRE。安装程序使用注册表来确认 JRE 的基础路径，这可能是 Java 7 或 更新的版本，还可能包括安装在完整 JDK 中作为其一个部分存在的 JRE。在 64 位操作系统下运行时，安装程序会优先查找 64 位 JRE，只有当无法找到时，才去查找32位的 JRE。并非强制性规定必须使用安装程序所侦测到的默认 JRE，可以使用任何已经安装的 Java 7 或 更新的 JRE（32 位或 64 位）。   
- **托盘图标** 当 Tomcat 作为一种服务运行时，不会显示托盘图标。只有当选择在安装完后立即运行 Tomcat 时，不管此时 Tomcat 是否以服务形式运行，托盘图标都会显现。    
- 要想更好地了解如何管理以 Windows 服务形式运行的 Tomcat 的信息，可查看 [Window 服务指南](http://tomcat.apache.org/tomcat-8.0-doc/windows-service-howto.html)。   

针对启动与配置 Tomcat，安装程序会创建相关的快捷方式。另外，需要特别注意的是，只有当 Tomcat 运行时，Tomcat 的 管理 Web 应用（administration web application）工具才能使用。   

### UNIX 守护进程   

利用 commons-daemon 工程的 jsvc 工具，可以将 Tomcat 作为一个守护进程来运行。Tomcat 的二进制发行版中包含着 jsvc 的源代码包，它需要编译。构建 jsvc 需要一个 C ANSI 编译器（比如 GCC）、GNU Autoconf，以及一个 JDK。   

在运行脚本之前，先将环境变量 `JAVA_HOME` 设置为 JDK 的基础路径。在调用 `./configure` 脚本时，需要使用 `--with-java` 参数来指定 JDK 路径，比如：`./configure --with-java=/usr/java`。  

使用下列命令应该就能返回一个编译好的 jsvc 二进制可执行文件，位于 `$CATALINA_HOME/bin` 目录中——这需要的前提条件是：使用了 GNU TAR，并且将环境变量 `CATALINA_HOME` 指向 Tomcat 安装基本路径。  

请注意，应该使用 GNU make（gmake）而不是 FreeBSD 系统下的原生 BSD make。  

```
cd $CATALINA_HOME/bin
tar xvfz commons-daemon-native.tar.gz
cd commons-daemon-1.0.x-native-src/unix
./configure
make
cp jsvc ../..
cd ../..

```


使用下列命令，Tomcat 就可以作为一个守护进程来运行了。  

```
CATALINA_BASE=$CATALINA_HOME
cd $CATALINA_HOME
./bin/jsvc \
    -classpath $CATALINA_HOME/bin/bootstrap.jar:$CATALINA_HOME/bin/tomcat-juli.jar \
    -outfile $CATALINA_BASE/logs/catalina.out \
    -errfile $CATALINA_BASE/logs/catalina.err \
    -Dcatalina.home=$CATALINA_HOME \
    -Dcatalina.base=$CATALINA_BASE \
    -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager \
    -Djava.util.logging.config.file=$CATALINA_BASE/conf/logging.properties \
    org.apache.catalina.startup.Bootstrap

```  

如果 JVM 默认使用的是服务器 VM，而不是客户端 VM，则可能还需要指定 `-jvm server`。这一点已经在 OS X 系统下得到证实。  

jsvc 还有其他一些有用的参数。比如：`-user` 就能让守护进程初始化完成后切换到另一个用户，从而能以非特权用户来运行 Tomcat，同时又能使用特权端口。不过要注意的是，如果使用这个选项来以根用户运行 Tomcat，需要禁用 `org.apache.catalina.security.SecurityListener` 检查，这个检查是用来防止以根用户来运行 Tomcat 的。  

`jsvc --help` 参数会提供完整的 jsvc 用途信息。尤其是 `-debug` 参数，它对于调试 jsvc 运行中出现的问题是非常有用的一个工具。   

`$CATALINA_HOME/bin/daemon.sh` 可以作为一个模板，利用 jsvc `/etc/init.d/` 在启动时自动开启 Tomcat。    


注意，要想以上述方式运行 Tomcat，Commons-Daemon JAR 文件必须位于运行时的类路径上。Commons-Daemon JAR 文件在bootstrap.jar 清单的类路径项中。如果某个 Commons-Daemon 类出现了 ClassNotFoundException（无法找到类） 或 NoClassDefFoundError（无法找到类定义） 这样的错误，那么在加载 jsvc 时将 Commons-Daemon JAR 添加到 `-cp` 参数中。     

