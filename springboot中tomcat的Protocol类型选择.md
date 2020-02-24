# SpringBoot中的tomcat的Protocal类型选择  

最近在工作中收到领导的通知，要求检查生产服务器使用的tomcat是哪个版本，询问才知道原来是收到通知，一些旧版本的AJP是存在问题的，所以要排查升级，并关闭AJP功能  
在实际生产中，打包为war包的应用是自己安装tomcat部署的，所以可以进行手动升级和关闭对应功能，但是SpringBoot框架下的又是如何处理呢？  
 
两个问题  
+  怎么确定SpringBoot内置tomcat版本号  
这个相对比较简单，都知道，SpringBoot使用父pom来管理一些常用包的版本号，除非手动再次指定版本号，否则，使用内置依赖中的版本号。
``` 
    //父pom
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.4.7.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
    //父pom的父pom  
    <parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-dependencies</artifactId>
		<version>1.4.7.RELEASE</version>
		<relativePath>../../spring-boot-dependencies</relativePath>
	</parent>
    // 在内部可以找到
    <properties>
        <tomcat.version>8.5.15</tomcat.version>
	</properties>
```  
+ 怎么确定SpringBoot的内置tomcat的Protocal配置和如何修改
### SpringApplication的run方法开始
```
	public static void main(String[] args) {
		SpringApplication.run(SpringApplication.class, args);
	}
```
