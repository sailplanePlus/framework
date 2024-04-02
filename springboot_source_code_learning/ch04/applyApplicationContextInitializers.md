# ApplicationContextInitializers 应用到上下文中

初始化给定的应用上下文
```java
	protected void applyInitializers(ConfigurableApplicationContext context) {
		for (ApplicationContextInitializer initializer : getInitializers()) {
			Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
					ApplicationContextInitializer.class);
			Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
			initializer.initialize(context);
		}
	}
```
1. DelegatingApplicationContextInitializer
> 获取环境属性"context.initializer.classes"下的ApplicationContextInitialzier，并实例化。执行initialize方法，初始化给定的应用上下文
```java
	@Override
	public void initialize(ConfigurableApplicationContext context) {
		ConfigurableEnvironment environment = context.getEnvironment();
		List<Class<?>> initializerClasses = getInitializerClasses(environment);
		if (!initializerClasses.isEmpty()) {
			applyInitializerClasses(context, initializerClasses);
		}
	}
```
> 获取环境属性"context.initializer.classes"下的ApplicationContextInitialzier
```java
	private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
		String classNames = env.getProperty(PROPERTY_NAME);
		List<Class<?>> classes = new ArrayList<>();
		if (StringUtils.hasLength(classNames)) {
			for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
				classes.add(getInitializerClass(className));
			}
		}
		return classes;
	}
```
> 实例化ApplicationContextInitializer
```java
	private void applyInitializerClasses(ConfigurableApplicationContext context, List<Class<?>> initializerClasses) {
		Class<?> contextClass = context.getClass();
		List<ApplicationContextInitializer<?>> initializers = new ArrayList<>();
		for (Class<?> initializerClass : initializerClasses) {
			initializers.add(instantiateInitializer(contextClass, initializerClass));
		}
		applyInitializers(context, initializers);
	}
```
> 初始化应用上下文
```java
	private void applyInitializers(ConfigurableApplicationContext context,
			List<ApplicationContextInitializer<?>> initializers) {
		initializers.sort(new AnnotationAwareOrderComparator());
		for (ApplicationContextInitializer initializer : initializers) {
			initializer.initialize(context);
		}
	}
```
2. SharedMetadataReaderFactoryContextInitializer
> 创建用于在ConfigurationClassPostProcessor和Spring Boot之间创建共享的CachingMetadataReaderFactory。
> 向beanFactoryPostProcessors注册表中注册CachingMetadataReaderFactoryPostProcessor
```java
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		BeanFactoryPostProcessor postProcessor = new CachingMetadataReaderFactoryPostProcessor(applicationContext);
		applicationContext.addBeanFactoryPostProcessor(postProcessor);
	}
```
3. ContextIdApplicationContextInitializer
> 设置Spring应用程序上下文的ID。使用spring.application.name属性来创建ID。如果未设置该属性，则使用application。并向容器一级缓存中注册ContextId单例对象
```java
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		ContextId contextId = getContextId(applicationContext);
		applicationContext.setId(contextId.getId());
		applicationContext.getBeanFactory().registerSingleton(ContextId.class.getName(), contextId);
	}
```
4. ConfigurationWarningsApplicationContextInitializer
> 向beanFactoryPostProcessors注册表中注册ConfigurationWarningsPostProcessor
```java
	@Override
	public void initialize(ConfigurableApplicationContext context) {
		context.addBeanFactoryPostProcessor(new ConfigurationWarningsPostProcessor(getChecks()));
	}
```
5. RSocketPortInfoApplicationContextInitializer
>  用于设置 RSocketServer 服务器实际监听的端口的环境属性。属性 "local.rsocket.server.port" 可以直接通过 @Value 注入到测试中，也可以通过 Environment 获取。
属性会自动传播到任何父上下文中。
向ApplicationContext中的applicationListeners中添加Listener
```java
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		applicationContext.addApplicationListener(new Listener(applicationContext));
	}
```
6. ServerPortInfoApplicationContextInitializer
> 用于设置 Web 服务器实际监听的端口的环境属性。属性 "local.server.port" 可以直接通过 @Value 注入到测试中，也可以通过 Environment 获取。
如果 WebServerInitializedEvent 具有服务器命名空间，它将用于构造属性名称。例如，"management" 执行器上下文将具有属性名称 "local.management.port"。
属性会自动传播到任何父上下文中。
向ApplicationContext中的applicationListeners中添加ServerPortInfoApplicationContextInitializer
```java
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		applicationContext.addApplicationListener(this);
	}
	
	//监听WebServerInitializedEvent事件，将端口号设置的属性源中
	@Override
	public void onApplicationEvent(WebServerInitializedEvent event) {
		String propertyName = "local." + getName(event.getApplicationContext()) + ".port";
		setPortProperty(event.getApplicationContext(), propertyName, event.getWebServer().getPort());
	}
```
7. ConditionEvaluationReportLoggingListener
> 将 ConditionEvaluationReport 写入日志。报告将以 DEBUG 级别记录。如果发生崩溃，将触发一个提示信息，建议用户使用启用调试的方式重新运行以显示报告。
此初始化程序不打算在多个应用程序上下文实例之间共享。
```java
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		applicationContext.addApplicationListener(new ConditionEvaluationReportListener());
		if (applicationContext instanceof GenericApplicationContext) {
			// Get the report early in case the context fails to load
			this.report = ConditionEvaluationReport.get(this.applicationContext.getBeanFactory());
		}
	}
```