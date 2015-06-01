# 29 Windows 服务  

## 目录  

## Tomcat 服务应用  

**Tomcat8** 是一个服务应用，能使 Tomcat 8 以 Windows 服务的形式运行。  

## Tomcat 监控应用  

**Tomcat8w** 是一个监控与配置 Tomcat 服务的 GUI 应用。  

可用的命令行选项为：  

<table >
<tr>
　<td><b>//ES//</b></td>
　<td>编辑服务配置</td>
　<td>这是默认操作。如果没有提供其他选项，则调用它。但是可执行未见被重命名为<b>servicenamew.exe。</b></td>
</tr>
<tr>
　<td><b>//MS//</b></td>
　<td>监控服务</td>
　<td>将图标放到系统托盘中。</td>
</tr>
　
</table>  
　
　
　
　
## 命令行实参    

命令行指令格式为：**//XX//ServiceName**。   

可用的命令行选项为：   


<table >
<tr>
　<td><b>//TS//</b></td>
　<td>以控制台应用的方式运行服务</td>
　<td>默认操作。如果没有其他选项，则调用它。ServiceName 是可执行文件没有后缀 exe 的名称，即 Tomcat8。</td>
</tr>
<tr>
　<td><b>//RS//</b></td>
　<td>运行服务</td>
　<td>只能被服务管理器调用</td>
</tr>  

<tr>
　<td><b>//SS//</b></td>
　<td>停止服务</td>
　<td></td>
</tr>

<tr>
　<td><b>//US//</b></td>
　<td>更新服务参数</td>
　<td></td>
</tr>

<tr>
　<td><b>//IS//</b></td>
　<td>安装服务</td>
　<td></td>
</tr>

<tr>
　<td><b>//DS//</b></td>
　<td>删除服务</td>
　<td>如果服务运行，则停止服务</td>
</tr>

</table>  


## 命令行形参  

每一个命令形参都有一个前缀 `--`。如果命令行前缀为 `++`，则该值会附加到已有选项中。如果环境变量和命令行形参相同，但是前缀是 `PR_`，则它要优先处理。比如：  

`set PR_CLASSPATH=xx.jar`  

它等同于把以下作为命令行形参的》》：  

`--Classpath=xx.jar`   

|形参名称|默认|描述|
|---|---|---|  
|`--Description`|-|服务名描述（最大 1024 字符）|
|`--DisplayName`|服务名|服务显示名|
|`--Install`| procrun.exe //RS//ServiceName |安装映像|
|`--Starup`|manual|服务启动模式有两种：**auto** 或 **manual**|  
|`++DependsOn`|-|该服务所依赖的一组其他服务。用 `#` 或 `;` 字符来分隔依赖服务|
|`++Environment`|-|利用 **键 = 值** 形式提供给服务的一组环境变量。用 `#` 或 `;` 字符来分隔依赖这些环境变量。如果需要在一个值中使用 `#` 或 `;` 字符，那么整个值必须以单引号闭合。|
|`--User`|-|用于运行可执行文件的用户账户。只用于 `StarMode` 取 **java** 或 **exe** 这两种值时，并且能使应用作为一种服务，运行在没有 LogonAsService 特权下的账户下。|
|`--Password`|-|通过 `--User` 形参设定的用户账户密码。|
|`--JavaHome`|JAVA_HOME|设定一个与同名环境变量所不同的 JAVA_HOME|
|`--Jvm`|auto|可以使用 **auto**（意即从 Windows 注册表中寻找 JVM），或者指定指向 **jvm.dll** 的完整路径。可以在此使用环境变量扩展。|
|`++JvmOptions`|-Xrs|传入 JVM 的一组选项，格式为 **-D** 或 **-X**。通过`#` 或 `;` 字符来分隔依赖这些选项（不能用于 **exe** 模式）。|
|`--Classpath`|-|设定 Java 类路径（不能用于 **exe** 模式）| 
|`--JvmMs`|-|初始内存池容量，以 MB 计。不能用于 **exe** 模式|
|`--JvmMx`|-|内存池最大容量，以 MB 计。不能用于 **exe** 模式|
|`--JvmSs`|-|线程堆栈容量，以 KB 计。不能用于 **exe** 模式|
|`--StartMode`|-|取值为 **jvm**、**java**、**exe** 其中之一。这些模式的含义为：<br/><li>jvm——进程内启动 Java。依赖 jvm.dll，参看 **--Jvm** 形参相关描述</li><li> Java——与 exe 类似，但会自动使用默认的 java 可执行文件。也即 %JAVA_HOME%\bin\java.exe。确保正确设定 JAVA_HOME，或使用 --JavaHome 来提供正确的位置。如果都未设定，procrun 会从注册表中寻找默认的 JDK（不是 JRE）</li><li>exe——以独立进程方式运行映像</li>|
|`--StartImage`|-|运行的可执行文件。只适用于 **exe** 模式|
|`--StartPath`|-|start 映像可执行文件的工作路径|
|`--StartClass`|Main|包含启动方法的类。适用于 **jvm** 与 **java** 模式，不适用于 **exe** 模式|
|`--StartMethod`|main|方法名如果不同，则使用 main|
|`++StartParams`|-|传入 `StartImage` 或 `StartClass` 的一组形参。用 `#` 或 `;` 字符来分隔形参。|
|`--StopMode`|-|取值为 **jvm**、**java**、**exe** 其中之一。更多详细描述参看 **--StartMode** |
|`--StopImage`|-|运行于 Stop 服务信号下的可执行文件。只适用于 **exe** 模式| 
|`--StopPath`|-|停止映像的可执行文件的工作路径。不适用于 **jvm** 模式|
|`--StopClass`|Main|用于 Stop 服务信号的类。适用于 **jvm** 和 **java** 类|
|`--StopMethod`|main|方法名如果不同，则使用 main|
|`++StopParams`|-|传入 StopImage 或 StopClass 的一组形参。用 `#` 或 `;` 字符来分隔形参|
|`--StopTimeout`|没有超时|用于 procrun 等待服务优雅地结束时的超时秒数。|
|`--LogPath`|%SystemRoot%\System32\LogFiles\Apache|定义日志路径。如有必要则创建路径。|
|`--LogPrefix`|commons-daemon|定义服务日志文件名前缀。日志文件被创建在 LogPath 所定义的路径处，带有后缀 `.YEAR-MONTH-DAY.log`|
|`--LogLevel`|Info|定义日志级别。取值为以下这些值的其中之一：**Error**、**Info**、**Warn** 或 **Debug**。（区分大小写） |
|`--StdOutput`|-|重定向的标准输出文件名。如果指定为 **auto**，则文件创建在 LogPath 所定义的路径处，文件名形式为：**service-stdout.YEAR-MONTH-DAY.log**|
|`--StdError`|-|重定向的标准错误文件名。如果指定为 **auto**，则文件创建在 LogPath 所定义的路径处，文件名形式为：**service-stderr.YEAR-MONTH-DAY.log**|
|`--PidFile`|-|定义运行中的进程 id 的文件名。实际文件创建在 **LogPath** 目录中。|  


## 安装服务  

最安全的手动安装服务的方式是利用提供的 **service.bat** 脚本。需要有管理员特权才能运行该脚本。为了安装服务，必要时可以采用 `/user` 指定一个用户。  

**注意**：在 Windows Vista 或其他版本更新的 Windows 操作系统上，如果开启了用户账户控制功能（UAC，User Account Control），当脚本启动 Tomcat8.exe 时，系统会要求提供额外的特权。如果你想为服务安装程序传入附加选项，如 `PR_*` 环境变量，则必须在系统对它们进行全局配置，或者启动相关程序，利用更高级的特权来设置它们，比如：右键点击 cmd.exe 然后选择 “以管理员身份运行”；在 Windows 8（或更新版本）或 Windows Server 2012（或更新版本）系统中，还可以在文件资源管理器中点击“文件”菜单，为当前目录打开一个高级命令提示符（elevated command prompt）。详情参看[问题 56143](https://bz.apache.org/bugzilla/show_bug.cgi?id=56143)。  

```  

```

还有第 2 个参数，可以让你指定服务名，》》  

```  

```

如果使用 tomcat8.exe，你需要使用 **//IS//** 参数。  

```  

```

## 更新服务  

要想更新服务参数，需要使用 **//US//** 参数。  

```  

```  

如果想为服务指定可选名，需要以如下方式进行：  

```

```

## 删除服务   

如要删除服务，需使用 **//DS//** 参数。  
如果服务正在运行，则会先停止然后再删除。  

```   

```   

为服务指定可选名的方式如下：  

## 调试服务  

想要在控制台模式下运行服务，需使用 **//TS//** 参数。   

## 多个实例  

Tomcat 支持安装多个实例。

每个实例文件夹都需要具有如下结构：  

- conf  
- logs
- temp
- webapps
- work

》，conf 应该包含 CATALINA_HOME\conf\ 中的下列文件的副本。  》没有复制或编辑的文件，直接从 CATALINA_HOME\conf 

- server.xml  
- web.xml   

必须编辑 CATALINA_BASE\conf\server.xml，指定一个唯一的 IP/端口用于实例侦听。找到包含 `<Connector port="8080" ...` 的代码行，添加一个地址属性，并且（或者）更新端口号，以便指定一个唯一的 IP/端口组合。   

要想安装一个实例，首先将 CATALINA_HOME 环境变量设置为 Tomcat 安装目录名称。然后创建一个第二个环境变量 CATALINA_BASE，并将其指向实例文件夹。最后运行 `service install` 命令指定服务名称。  

```  

```  

修改服务设置，需要运行 **tomcat8w //ES//instance1**。    

对于附加实例，创建附加实例文件夹，更新 CATALINA_BASE 环境变量，然后再次安装服务。  

```

```  












