## 目录  

## 简介  

对于大多数用例来说，默认配置下的 Tomcat 都是相当安全的。有些环境可能需要更多（或更少）的安全配置。This page is to provide a single point of reference for configuration options that may impact security and to offer some commentary on the expected impact of changing those options. The intention is to provide a list of configuration options that should be considered when assessing the security of a Tomcat installation.本文提供的 》》可能影响安全性的配置选项，并提供了修改这些选项所带来的预期影响。目的是为了在评价 Tomcat 安装时，提供一些应值得考虑的配置选项。

**注意**：本章内容毕竟有所局限，你还需要对配置文档进行深入研究。在相关文档中有更完整的属性描述。       

## 非 Tomcat 设置  

Tomcat 配置不应成为唯一的防线，也应该保障系统（操作系统、网络及数据库，等等）中的其他组件的安全。    

不应该以根用户的身份来运行 Tomcat，应为 Tomcat 进程创建并分配一个专门的用户，并为该用户配置最少且必要的操作系统权限。比如，不允许使用 Tomcat 用户实现远程登录。    

文件权限同样也应适当限制。就拿 ASF 中的 Tomcat 实例为例说明吧（禁止自动部署，Web 应用被部署为扩张的目录。》》），标准配置规定所有的 Tomcat 文件》，然而拥有者具有读写特权，》》。   

对于网络层面，需要使用防火墙来限制进站与出站连接，只允许出现那些你希望的连接。   


## 默认的 Web 应用  

###  General  

### ROOT  

ROOT 应用带来安全风险的可能性非常小，但它确实含有正在使用的 Tomcat 的版本》。应该从可公开访问的 Tomcat 实例中清除 ROOT 应用，不是出于安全性原因，而是因为这样能给用户提供一个更适合的默认页面。   

### Documentation   

Documentation 带来安全风险的可能性非常小，但》它标识出了当前正使用的 Tomcat 版本。应该从可公开访问的 Tomcat 实例中清除该应用。   

### Examples   

应该从安全敏感性安装中移除 examples 应用。虽然 examples 应用并不包含任何已知的缺陷，



















