#  重写 Valve
## 目录 

## 简介  

**重写 Valve**（Rewrite Valve） 实现 URL 重写功能的方式非常类似于 Apache HTTP Server 的 `mod_rewrite` 模块。  

## 配置  

重写 Valve 是通过使用 `org.apache.catalina.valves.rewrite.RewriteValve` 类名来配置成 Valve 的。  

经过配置，重写 Valve 可以做为一个 Valve 添加到 Host 中。参考[虚拟服务器文档](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html)来了解配置详情。该 Valve 使用包含重写指令的 `rewrite.config` 文件，且必须放在 Host 配置文件夹中。  

另外，重写 valve 也可以用在 Web 应用的 context.xml 中。该 Valve 使用包含重写指令的 `rewrite.config` 文件，且必须放在 Web 应用的 `WEB-INF` 文件夹中。    

## 指令  

`rewrite.config` 文件包含一系列指令，这些指令和 `mod_rewrite` 所用的指令很像，尤其是核心的 `RewriteRule` 与 `RewriteCond` 指令。  

**注意**：该部分内容修改自 `mod_rewrite` 文档，后者版权归属于 Apache 软件基金会（1995-2006），遵循 Apache 许可发布。  

### 1. RewriteCond  

格式：`RewriteCond TestString CondPattern`  

`RewriteCond` 指令定义了一个规则条件。一个或多个 `RewriteCond` 指令可以优先于 `RewriteRule` 指令执行。如果 URI 当前状态匹配它的模式，并且满足了这些条件，才会使用下列规则。  

*TestString* 是一种字符串，除了简单的文本之外，它还可以含有下列扩展结构。  

- `RewriteRule backreferences` 对形式 `$N`（0 <= N <= 9）的反向引用。提供对模式成组部分的访问（括号中的），从属于 `RewriteCond` 条件当前状态的 `RewriteRule`。  
- `RewriteCond backreferences`  
- `RewriteMap expansions`  
- `Server-Variables`  这些是形式 `%{ NAME_OF_VARIABLE }` 中的变量。`%{ NAME_OF_VARIABLE }` 中的 *NAME_OF_VARIABLE* 是一种取自下面列表的字符串：  

- **HTTP 报头**：  
HTTP\_USER\_AGENT  
HTTP\_REFERER  
HTTP\_COOKIE  
HTTP\_FORWARDED  
HTTP\_HOST  
HTTP\_PROXY\_CONNECTION  
HTTP\_ACCEPT    

- **连接与请求**：  
REMOTE\_ADDR  
REMOTE\_HOST  
REMOTE\_PORT  
REMOTE\_USER  
REMOTE\_IDENT  
REQUEST\_METHOD  
SCRIPT\_FILENAME  
REQUEST\_PATH  
CONTEXT\_PATH  
SERVLET\_PATH  
PATH\_INFO  
QUERY\_STRING  
AUTH\_TYPE    
- **服务器内部**：  
DOCUMENT\_ROOT    
SERVER\_NAME  
SERVER\_ADDR  
SERVER\_PORT  
SERVER\_PROTOCOL  
SERVER\_SOFTWARE  
- **日期与时间**：  
TIME\_YEAR  
TIME\_MON  
TIME\_DAY  
TIME\_HOUR  
TIME\_MIN  
TIME\_SEC  
TIME\_WDAY  
TIME  
- **特殊字符串**：   
THE\_REQUEST  
REQUEST\_URI  
REQUEST\_FILENAME  
HTTPS     

这些变量对应着相似名称的 HTTP MIME 报头和 Servlet API 方法。多数都记录在各种手册和 CGI 规范中。下面列出了重写 Valve 专有的那些变量：  

- `REQUEST_PATH`   
对应用于映射的完整路径。   
- `CONTEXT_PATH`    
对应映射的上下文的路径。    
- `SERVLET_PATH`      
对应 Servlet 路径。   
- `THE_REQUEST`  
由浏览器发送给服务器的完整 HTTP 请求代码行（比如，`GET /index.html HTTP/1.1`）。这并不包括任何由浏览器发送的额外报头。  
- `REQUEST_URI`   
HTTP 请求代码行中所请求的资源（在上例中，应为为 `/index.html`）。  
- `REQUEST_FILENAME`   
与请求相匹配的文件或脚本的完整本地文件系统路径。  
- `HTTPS`    
当连接使用 SSL/TLS 时，含有文本 `"on"`，否则含有 `"off"`。    

另外还需要注意的是：  

1. `SCRIPT_FILENAME` 和 `REQUEST_FILENAME` 含有同样的值：Apache 服务器内部结构 `request_rec` 的 `filename` 字段值。第一个名称常被称为 CGI 变量值，第二个名称相当于 `REQUEST_URI`（包含 `request_rec` 的 `uri` 字段值）。   
2. `%{ENV:variable}`。其中的 `variable` 可以是任何 Java 系统属性。目前可以使用。  
3. `%{SSL:variable}`。其中的 `variable` 是 SSL 环境变量名。目前还未实现。范例： `%{SSL:SSL_CIPHER_USEKEYSIZE}` 可能扩展到 `128`。  
4. `%{HTTP:header}`。其中的 `header` 可以是任意的 HTTP MIME 报头名称。该变量常用来获取发送到 HTTP 请求的报头值。范例： `%{HTTP:Proxy-Connection}` 是 HTTP 报头 `Proxy-Connection:` 的值。  

*CondPattern* 即条件模式，是一种应用于 *TestString* 当前实例的正则表达式。*TestString* 在匹配 *CondPattern* 之前，会首先求值。

**谨记**：*CondPattern* 是一种兼容 perl 并带有一些扩展的正则表达式。  

1. 可以在模式字符串前加上 `!` 字符作为前缀，来指定**非**匹配模式。  
2. *CondPattern* 有一些特殊变体。除了真正的正则表达式字符串外，还可以使用下列组合形式之一：  
- `<CondPattern` (字母顺序高于)  
将 *CondPattern* 当成一种纯粹字符串，按照字母顺序将其与 *TestString* 进行对比。如果在字母顺序上，*TestString* 高于 *CondPattern* ，则为 true。  
- `>CondPattern`（字母顺序低于）
将 *CondPattern* 当成一种纯粹字符串，按照字母顺序将其与 *TestString* 进行对比。如果在字母顺序上，*TestString* 低于 *CondPattern* ，则为 true。  
- `=CondPattern'` (字母顺序等于)  
将 *CondPattern* 当成一种纯粹字符串，按照字母顺序将其与 *TestString* 进行对比。如果在字母顺序上，*TestString* 等于 *CondPattern* ，则为 true。 
- `-d`（目录）  
将 `TestString` 当成一种路径名，测试它是否存在并且是一个目录。  
- `-f`（常规文件） 
将 `TestString` 当成一种路径名，测试它是否存在并且是一个常规文件。  
- `-s`（常规文件，带有文件尺寸）  
将 `TestString` 当成一种路径名，测试它是否存在并且是一个文件大小大于 0 的普通文件。  

**注意**：所有这些测试都可以加上前缀 `!` 来使它们的含义反向。
3. 可以将 `[flag]` 做为 `RewriteCond` 指令的第三个参数，为 `CondPattern` 设定标记。这里的 flag 是一个包含下列任一标记，且标记间由逗号分隔的列表：  
- `nocase|NC`（不区分大小写）
无论是扩展的 *TestString* 还是在 *CondPattern* 中，都不区分大小写（A-Z 和 a-z 是等同的）。该标记只有在比较 *TestString* 和 *CondPattern* 时才是有效的。对于文件系统和子请求（HTTP 请求）是无效的。
- `ornext|OR`（或者下一个条件）
利用本地 OR（而不是隐式的 AND）来组合规则条件。典型范例为：

```  
RewriteCond %{REMOTE_HOST}  ^host1.*  [OR]  
RewriteCond %{REMOTE_HOST}  ^host2.*  [OR]  
RewriteCond %{REMOTE_HOST}  ^host3.*  
RewriteRule ...some special stuff for any of these hosts...		  
```  
没有该标记，你必须写三遍条件/规则对。  

**范例**： 

假如想根据请求头的 `User-Agent:` 对网站主页进行重写，可以使用下列代码：  

```  
RewriteCond  %{HTTP_USER_AGENT}  ^Mozilla.*
RewriteRule  ^/$                 /homepage.max.html  [L]

RewriteCond  %{HTTP_USER_AGENT}  ^Lynx.*
RewriteRule  ^/$                 /homepage.min.html  [L]

RewriteRule  ^/$                 /homepage.std.html  [L]

```

说明：如果使用的浏览器将它自身标识为 'Mozilla'（包括 Netscape Navigator、Mozilla，等等），那么这就是内容最大化主页（max homepage。它可以包含框架或其他特性）；如果使用的是 Lynx 浏览器（基于终端的），那么获得的就是内容最小化的主页（min homepage）——专为便于浏览文本而设计的版本；如果这些条件都不适用（你使用的是其他浏览器，或者你的浏览器将其自身标识为非标准内容），那么得到的就是标准主页（std homepage）。  

### 2. RewriteMap   

格式：`RewriteMap name rewriteMapClassName optionalParameters`。  

通过用户必须实现的一个接口来实现映射。类名为 `org.apache.catalina.valves.rewrite.RewriteMap`，代码为：  

```  
package org.apache.catalina.valves.rewrite;

public interface RewriteMap {
public String setParameters(String params);
public String lookup(String key);
}

```

### 3. RewriteRule  

格式：`RewriteRule Pattern Substitution`  

`RewriteRule` 指令是重写机制的核心。此指令可以多次使用，每个实例都定义一个单独的重写规则。这些规则的定义顺序尤为重要，因为这是在运行时应用它们的顺序。  

模式是一个作用于当前 URL 的兼容 perl 的正则表达式，这里的“当前”是指该规则生效时的 URL，它可能与被请求的 URL 不同，因为其他规则可能在此之前已经发生匹配并对它做了改动。   

下面是关于正则表达式格式的一些提示：    


> **文本**： 

> - `.`——匹配任何单个字符
> - `[`chars`]`——匹配当前字符
> - `[^`chars`]`——不匹配当前字符
> - text1`|`text2——匹配 text1 或 text2

---  

> **量词**:
> - `?`——零个或者 1 个 `?` 号前的字符
> - `*`——零个或者 N 个 `*` 号前的字符（N > 0）
> - `+`——零个或 N 个 `+` 号前的字符（N > 1）    

---  

> **分组**：
> `(`text`)`——文本分组（设定第 N 组可以被引用为 RewriteRule）

---  

> **行锚**  
> 
> `^`——匹配一行起始处的空字符串。     
> `$`——匹配一行结束处的空字符串。   

---  

> **转义**  
> 
> `\char` ——将指定字符转义（比如将`.`、`[]`、`()` 等字符转义）




要想了解更多关于正则表达式的信息，请参见 perl 正则表达式在线联机手册（[perldoc perlre](http://www.perldoc.com/perl5.6.1/pod/perlre.html)）。关于正则表达式及其变体（POSIX 正则表达式）的详情，可看看这本书：  


*Mastering Regular Expressions, 2nd Edition* （目前该书为第 3 版）
Jeffrey E.F. Friedl 
O'Reilly & Associates, Inc. 2002
ISBN 978-0-596-00289-3  


在规则中，NOT 字符（`!`）可作为模式的前缀，实现逆向模式。比如“如果当前 URL 并**不**匹配该模式”。这可以用在易于匹配的是逆向模式这种异常情况下，或者也可以作为最后一个默认规则来使用。   

注意：在使用 NOT 字符反转模式时，不能在模式中包含分组的通配成分。这是因为，如果模式不匹配（比如反匹配），分组内将没有内容。如果使用了逆向模式，就不能在替代字符串中使用 `$N`。  

重写规则中的替代字符串（substitution string）是一种用来取代模式匹配的原始 URL 的字符串。除了纯文本之外，它还包括：  

1. 反向引用（`$N`）RewriteRule 模式。  
2. 反向引用（`%N`）最后匹配的 RewriteCond 模式。  
3. 规则条件测试字符串中（`%{VARNAME}`）的服务器变量。  
4. 映射函数调用（`${mapname:key|default}`）。  

反向引用表示的是形式 `$N`（N 的范围为 0-9），它是指用模式所匹配的第 **N** 组的内容去替换 URL。服务器变量 `RewriteCond` 指令的 *TestString* 所用的相同。映射函数来自 `RewriteMap` 指令。这 3 类变量都按照上述顺序展开。  

如前所述，所有的重写规则都应用于替代字符串（按照配置文件中所定义的顺序）。URL 完全由替代字符串所替换，直到所有规则都应用完毕，重写过程才结束（或利用 `L`  标记来终止）。  

还有一个特殊的替代字符串：`-`，意思是**不替代**，当需要让重写规则只匹配 URL 而不替换时，就用得上它了。这一字符串通常与 **C**（chain）标记配合使用，为的是在替换发生前应用多个模式。  

另外，还可以将 `[`flags`]` 做为 RewriteRule 的第三个参数，从而为替代字符串设置特殊标记。*flags*是一个包含下列标记且标记间以逗号分隔的列表：  

- `chain|C` （与下一个规则相链接）

此标记使当前规则与下一个规则（它又可以与其后规则相链接，如此反复）相链接。 它产生如下效果：如果某个规则被匹配，通常会继续处理其后继规则，这个标记就不起作用；如果某规则不被匹配，则其后继链接规将会被忽略。比如，在执行一个外部重定向时， 对一个目录级规则集，你可能需要删除 `.www`（因为此处不应该出现 `.www`）。  

- `cookie|CO` = ***NAME:VAL:domain[:lifetime[:path]]*** （设置 cookie）

在客户端浏览器上设置一个 cookie。cookie 的名称是 ***NAME***，其值是 ***VAL***。 *domain* 字段是该 cookie 的域，比如 `.apache.org`，可选的*lifetime* 是 cookie 生命周期（以分钟计），可选的 *path* 是 cookie 的路径。    

- `env|E` = *VAR:VAL* （设置环境变量）

强制一个名为 *VAR* 的请求变量值为 *VAL*，VAL可以包含可扩展的反向引用的正则表达式 `$N` 和 `%N`。可以多次使用该标记，以便设置多个变量。   

- `forbidden|F`（强制禁止访问该 URL ）

强制禁止访问当前的 URL——立即反馈一个 HTTP 响应代码 403。使用这个标记，可以链接若干 `RewriteConds` 以阻断某些 URL。  

- `gone|G`（强制 URL 为已失效）

强制当前 URL 为已失效——立即反馈一个 HTTP 响应代码 410（请求资源已被删除）。使用这个标记，可以标明页面已被删除而不存在.  

- `host|H=Host`（重写虚拟主机）  
重写虚拟主机，而不是重写 URL。

- `last|L`（最后一个规则）

立即停止重写操作，并不再应用其他重写规则。它对应于 Perl 中的 `last` 命令或 C 语言中的 `break` 命令。使用该标记可以防止当前已被重写的 URL 被后续规则所继续重写。比如，用它可以将根路径的 URL（`/`）重写为实际存在的 URL，比如：`/e/www/`。    

- `next|N`（重新执行）  

重新执行重写操作（从第一个重写规则重新开始）。这时，要匹配的 URL 已不是原始 URL 了，而是经最后一个重写规则处理过的 URL。它对应于 Perl 中的 `next` 命令或 C 语言中的 `continue` 命令。此标记可以重新开始重写操作，立即回到循环的头部。  

**但是要小心，不要制造死循环！**  

- `nocase|NC`（不区分大小写）

使模式不区分大小写。当模式与当前 URL 匹配时，A-Z 和 a-z 没有区别。   

- `noescape|NE`（在输出中不对 URI 进行转义）   

此标记阻止重写 Valve 对重写结果应用常规的 URI 转义规则。一般情况下，特殊字符（如`%`、`$`、`;`等）都会被转义为十六进制值。此标记可以阻止这种转义，允许百分号等符号出现在输出中，如：

`RewriteRule /foo/(.*) /bar?arg=P1\%3d$1 [R,NE]`  

将 `/foo/zed` 转化为 `/bar?arg=P1=zed` 的安全请求。  

- `qsappend|QSA`（附加查询串）

该标记会强制重写引擎将替代字符串中的一个查询字符串部分添加到已有字符串上，而不是简单地替换已有字符串。当你想要通过重写规则为查询字符串添加更多数据时，可以使用该标记。

- `redirect|R`**[=code]**（强制重定向）  

将 `http://thishost[:thisport]/`（使新的 URL 成为一个 URI）作为替代字符串的前缀，从而强制执行一个外部重定向。如果没有指定 code，则产生一个HTTP 响应代码 302（暂时移动）。如果需要使用在 300-400 范围内的其他响应代码，只需在此指定这个数值即可。另外，还可以使用下列符号名称：`temp`（默认），`permanent`、`seeother`。用它可以把规范化的 URL 反馈给客户端，如重写 `/~` 为 `/u/`，或对`/u/user` 加上斜杠，等等。  

**注意**：在使用这个标记时，必须确保替换字段是一个有效的 URL！否则它会指向一个无效的位置！并且要记住，此标记本身只是对 URL 加上 `http://thishost[:thisport]/` 前缀而已，不会妨碍重写操作。通常，你会希望停止重写，然后立即重定向。要想停止重写，你还需要添加 `L` 标记。   


- `skip|S`**=num**（忽略后续规则） 

如果当前规则匹配，此标记会强制重写引擎跳过当前匹配规则后面的 num 个规则。它可以实现一个类似 if-then-else 的构造：then 子句的最后一个规则是 `skip = N`，其中 N 代表 else 子句中的规则数目。（该标记不同于`chain|C` 标记！）

- `type|T`**=MIME-type**（强制指定 MIME 类型）

强制指定目标文件的 MIME 类型为 **MIME-type**，可基于一些条件设置内容类型。比如在下面的代码段中，如果利用 `.phps` 扩展调用 `.php` 文件，它们就能被 `mod_php` 模块显示。  

`RewriteRule ^(.+\.php)s$ $1 [T=application/x-httpd-php-source]`  






















