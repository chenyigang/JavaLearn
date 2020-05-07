- 第二章

1. ### 不使用spring-boot-starter-parent

springboot一般性是添加对应的parent的父pom，作用：

1. 统一Java版本号
2. 统一编码格式
3. 提供Dependency Management进行项目以来的版本管理

~~~
//如果不适用可以手动进行Dependency Management配置
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.2.6.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    <dependencies>
</dependencyManagement>
~~~

4. 默认的资源过滤与插件配置//TODO
