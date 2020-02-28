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
最终调用的是SpringApplication中的方法
```
	// SpringApplication.java
	/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			// 创建环境
			context = createApplicationContext(); //1
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 准备环境
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			// 刷新环境
			refreshContext(context); // 2
			// 刷新环境后处理
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```
// 1  
创建的环境其实是AnnotationConfigServletWebServerApplicationContext,他继承了ServletWebServerApplicationContext => GenericWebApplicationContext => GenericApplicationContext => AbstractApplicationContext
所以最终是一个AbstractApplicationContext类  
//TODO 附上继承树
```
	// SpringApplication.java
	/**
	 * Strategy method used to create the {@link ApplicationContext}. By default this
	 * method will respect any explicitly set application context or application context
	 * class before falling back to a suitable default.
	 * @return the application context (not yet refreshed)
	 * @see #setApplicationContextClass(Class)
	 */
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				switch (this.webApplicationType) {
				case SERVLET:
					//org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}

```
// 2  
然后回去追刷新环境的方法
```
	protected void refresh(ApplicationContext applicationContext) {
		Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
		((AbstractApplicationContext) applicationContext).refresh();
	}
```
实际执行的是 AbstractApplicationContext(父类)的方法

```
	// org.springframework.context.support.AbstractApplicationContext#refresh
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 在这里调用，但是实际上是调用了子类的方法
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```
// TODO 这里有个问题，继承关系是 AnnotationConfigServletWebServerApplicationContext => ServletWebServerApplicationContext => GenericWebApplicationContext => GenericApplicationContext => AbstractApplicationContext
但是在代码上
AnnotationConfigServletWebServerApplicationContext 无
ServletWebServerApplicationContext 有
GenericWebApplicationContext 有 
GenericApplicationContext 无
AbstractApplicationContext 有(最终的父方法)

实际调用的是 的 onRefresh 方法
```
	// org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
			// 在这个地方创建了WebServer服务器
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```
在创建服务器的时候使用的是ServletWebServerFactory,调用getWebServer()方法
```
	private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}

```
