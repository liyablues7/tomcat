## 目录  

## 简介  

Tomcat 使用 JMX MBean 来实现自身的性能管理。  

每个包里的 mbeans-descriptor.xml 是针对 Catalina 的 JMX MBean 描述。  

为了避免出现 “ManagedBean is not found” 异常，你需要为自定义组件添加 MBean 描述。


## 添加 Mbean 描述  

在 mbeans-descriptor.xml 文件中，你可以为自定义组件添加 Mbean 描述。这个 xml 文件跟它所描述的类文件同在一个包内。  

```
  <mbean         name="LDAPRealm"
            className="org.apache.catalina.mbeans.ClassNameMBean"
          description="Custom LDAPRealm"
               domain="Catalina"
                group="Realm"
                 type="com.myfirm.mypackage.LDAPRealm">

    <attribute   name="className"
          description="Fully qualified class name of the managed object"
                 type="java.lang.String"
            writeable="false"/>

    <attribute   name="debug"
          description="The debugging detail level for this component"
                 type="int"/>
    .
    .
    .

  </mbean>

```