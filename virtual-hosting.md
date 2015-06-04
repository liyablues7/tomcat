# 24 虚拟主机与 Tomcat  


## 目录  

- 前提设定  
- server.xml
- Web 应用目录  
- 配置你的上下文  
	1. 一般配置方法  
	2. context.xml - 方法 1  
	3. context.xml - 方法 2  
	4. 更多配置信息     


## 前提设定

针对本教程，假设你有一个开发主机，并有两个主机名：`ren` 和 `stimpy`。再来假设一个 Tomcat 运行实例，`$CATALINA_HOME` 表示它的安装位置，可能是 `/usr/local/tomcat`。     

另外，本教程使用 UNIX 风格的分隔符及命令，如果你使用的是 Windows，则需要相应修改一下。   

## server.xml  

编辑 `server.xml` 文件的 [Engine](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html) 部分，如下所示：  

```   
<Engine name="Catalina" defaultHost="ren">
    <Host name="ren"    appBase="renapps"/>
    <Host name="stimpy" appBase="stimpyapps"/>
</Engine>

```        

注意：每个主机的 appBase 下的目录结构不能彼此重复。   

关于 [Engine](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html) 与 [Host](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html) 元素的其他属性，可参看相关的配置文档。     



## Web 应用目录  

创建每一个虚拟主机的目录：  

```   
mkdir $CATALINA_HOME/renapps
mkdir $CATALINA_HOME/stimpyapps

```        

## 配置你的上下文  

### 1. 一般配置方法  

上下文通常位于 `appBase` 目录下。比如，在 `ren` 主机上配置 war 文件形式的 `foobar` 上下文，使用 `$CATALINA_HOME/renapps/foobar.war`。注意，`ren` 主机的默认或 ROOT 上下文应配置成 `$CATALINA_HOME/renapps/ROOT.war`（WAR 文件形式） 或 `$CATALINA_HOME/renapps/ROOT`（目录形式）。   

**注意：对于同一主机而言，上下文的 `docBase` 不能和 `appBase` 相同。**  

### 2. context.xml - 方法 1

在上下文中，创建一个 `META-INF` 目录，将你的上下文定义文件（`context.xml`）放入其中，比如说：`$CATALINA_HOME/renapps/ROOT/META-INF/context.xml`。这能使部署更加容易，特别对于分配的是WAR 文件时。  

### 3. context.xml - 方法 2

在 `$CATALINA_HOME/conf/Catalina` 下创建一个结构：  


```   
mkdir $CATALINA_HOME/conf/Catalina/ren
mkdir $CATALINA_HOME/conf/Catalina/stimpy

```  

注意结尾那个名为“Catalina”的目录表示的是如前所示的 [Engine](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html) 元素的 `name` 属性。  

对于默认的 Web 应用，则按如下方式添加：    

```  
$CATALINA_HOME/conf/Catalina/ren/ROOT.xml
$CATALINA_HOME/conf/Catalina/stimpy/ROOT.xml

```   

如果想为每个主机都使用 Tomcat Manager 应用，则需要按下列方式来添加它：   

```    
cd $CATALINA_HOME/conf/Catalina
cp localhost/manager.xml ren/
cp localhost/manager.xml stimpy/  

```


### 4. 更多信息  

有关 Context 元素的其他属性，可以参阅相关的配置文档：[Context](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html)。  













