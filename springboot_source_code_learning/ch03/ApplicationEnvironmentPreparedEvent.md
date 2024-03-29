# 事件发布：ApplicationEnvironmentPreparedEvent

### 官方解读：
> 当SpringApplication启动并且环境首次可用于检查和修改时，会发布一个事件。

### 事件发布过程
1. 调用org.springframework.boot.SpringApplicationRunListeners#environmentPrepared方法
```java
    void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
    		doWithListeners("spring.boot.application.environment-prepared",
    				(listener) -> listener.environmentPrepared(bootstrapContext, environment));
    }
```
2. 遍历SpringApplicationRunListener集合，执行environmentPrepared方法
```java
	private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
			Consumer<StartupStep> stepAction) {
		StartupStep step = this.applicationStartup.start(stepName);
		this.listeners.forEach(listenerAction);
		if (stepAction != null) {
			stepAction.accept(step);
		}
		step.end();
	}
```
3. 调用org.springframework.boot.context.event.EventPublishingRunListener#environmentPrepared方法
```java
	@Override
	public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
			ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(
				new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
	}
```
4. 事件多播器SimpleApplicationEventMulticaster发布ApplicationEnvironmentPreparedEvent事件
```java
	@Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```
5. 获取与ApplicationEnvironmentPreparedEvent事件类型匹配的ApplicationListener监听器集合。
```java
	protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

		Object source = event.getSource();
		Class<?> sourceType = (source != null ? source.getClass() : null);
		ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

		// Potential new retriever to populate
		CachedListenerRetriever newRetriever = null;

		//先检查缓存中是否已经存在
		CachedListenerRetriever existingRetriever = this.retrieverCache.get(cacheKey);
		if (existingRetriever == null) {
			// Caching a new ListenerRetriever if possible
			if (this.beanClassLoader == null ||
					(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
							(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
				newRetriever = new CachedListenerRetriever();
				existingRetriever = this.retrieverCache.putIfAbsent(cacheKey, newRetriever);
				if (existingRetriever != null) {
					newRetriever = null;  // no need to populate it in retrieveApplicationListeners
				}
			}
		}
        //如果存在缓存且ApplicationListener集合不为null，则直接返回。
		if (existingRetriever != null) {
			Collection<ApplicationListener<?>> result = existingRetriever.getApplicationListeners();
			if (result != null) {
				return result;
			}
			//如果结果为null，则表示现有的检索器尚未被另一个线程完全填充。
        //在当前的本地尝试中，继续操作，就好像无法对此进行缓存。
		}
      //检索ApplicationListener
		return retrieveApplicationListeners(eventType, sourceType, newRetriever);
	}
```
6. 检索ApplicationEnvironmentPreparedEvent事件的ApplicationListener
```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
			ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable CachedListenerRetriever retriever) {

		List<ApplicationListener<?>> allListeners = new ArrayList<>();
		Set<ApplicationListener<?>> filteredListeners = (retriever != null ? new LinkedHashSet<>() : null);
		Set<String> filteredListenerBeans = (retriever != null ? new LinkedHashSet<>() : null);

		Set<ApplicationListener<?>> listeners;
		Set<String> listenerBeans;
		synchronized (this.defaultRetriever) {
			listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
			listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
		}

		//过滤已编程方式注册的监听器，包括来自ApplicationListenerDetector(单例bean和内部bean)的监听器
		for (ApplicationListener<?> listener : listeners) {
			if (supportsEvent(listener, eventType, sourceType)) {
				if (retriever != null) {
					filteredListeners.add(listener);
				}
				allListeners.add(listener);
			}
		}

		//过滤按照bean名称添加的监听器
		if (!listenerBeans.isEmpty()) {
			ConfigurableBeanFactory beanFactory = getBeanFactory();
			for (String listenerBeanName : listenerBeans) {
				try {
					if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
						ApplicationListener<?> listener =
								beanFactory.getBean(listenerBeanName, ApplicationListener.class);
						if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
							if (retriever != null) {
								if (beanFactory.isSingleton(listenerBeanName)) {
									filteredListeners.add(listener);
								}
								else {
									filteredListenerBeans.add(listenerBeanName);
								}
							}
							allListeners.add(listener);
						}
					}
					else {
						// Remove non-matching listeners that originally came from
						// ApplicationListenerDetector, possibly ruled out by additional
						// BeanDefinition metadata (e.g. factory method generics) above.
						Object listener = beanFactory.getSingleton(listenerBeanName);
						if (retriever != null) {
							filteredListeners.remove(listener);
						}
						allListeners.remove(listener);
					}
				}
				catch (NoSuchBeanDefinitionException ex) {
					// Singleton listener instance (without backing bean definition) disappeared -
					// probably in the middle of the destruction phase
				}
			}
		}
      //排序
		AnnotationAwareOrderComparator.sort(allListeners);
		if (retriever != null) {
			if (filteredListenerBeans.isEmpty()) {
				retriever.applicationListeners = new LinkedHashSet<>(allListeners);
				retriever.applicationListenerBeans = filteredListenerBeans;
			}
			else {
				retriever.applicationListeners = filteredListeners;
				retriever.applicationListenerBeans = filteredListenerBeans;
			}
		}
		return allListeners;
	}
```
7. 我们来看一下具体的过滤方法supportsEvent
```java
protected boolean supportsEvent(
		ApplicationListener<?> listener, ResolvableType eventType, @Nullable Class<?> sourceType) {

	GenericApplicationListener smartListener = (listener instanceof GenericApplicationListener ?
			(GenericApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
	return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
}
```
首先将ApplicaitonListener转化为标准的监听器接口GenericApplicationListener
调用监听器的supportsEventType方法（校验的是ApplicationEvent）和supportsSourceType方法（校验的是事件源ApplicationEvent中的source）

8. 回调org.springframework.context.ApplicationListener#onApplicationEvent方法
```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}
```

### ApplicationEnvironmentPreparedEvent事件具体回调的ApplicationListener
1. **EnvironmentPostProcessorApplicationListener**，用于触发在spring.factories文件中注册的EnvironmentPostProcessor。
```java
	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationEnvironmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
		}
		if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent();
		}
		if (event instanceof ApplicationFailedEvent) {
			onApplicationFailedEvent();
		}
	}

	private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		SpringApplication application = event.getSpringApplication();
		for (EnvironmentPostProcessor postProcessor : getEnvironmentPostProcessors(application.getResourceLoader(),
				event.getBootstrapContext())) {
			postProcessor.postProcessEnvironment(environment, application);
		}
	}
```
> 这个监听器很重要，完成了配置文件的加载工作，后续专门针对其进行分析。

2. AnsiOutputApplicationListener，根据属性spring.output.ansi.enabled的值配置AnsiOutput
```java
	@Override
	public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		Binder.get(environment).bind("spring.output.ansi.enabled", AnsiOutput.Enabled.class)
				.ifBound(AnsiOutput::setEnabled);
		AnsiOutput.setConsoleAvailable(environment.getProperty("spring.output.ansi.console-available", Boolean.class));
	}
```

3. LoggingApplicationListener
```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationStartingEvent) {
			onApplicationStartingEvent((ApplicationStartingEvent) event);
		}
		else if (event instanceof ApplicationEnvironmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
		}
		else if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent((ApplicationPreparedEvent) event);
		}
		else if (event instanceof ContextClosedEvent) {
			onContextClosedEvent((ContextClosedEvent) event);
		}
		else if (event instanceof ApplicationFailedEvent) {
			onApplicationFailedEvent();
		}
	}
	
		private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		SpringApplication springApplication = event.getSpringApplication();
		if (this.loggingSystem == null) {
			this.loggingSystem = LoggingSystem.get(springApplication.getClassLoader());
		}
		initialize(event.getEnvironment(), springApplication.getClassLoader());
	}
```

4. BackgroundPreinitializer，用于在后台线程中触发耗时任务的早期初始化。
将IGNORE_BACKGROUNDPREINITIALIZER_PROPERTY_NAME系统属性设置为true以禁用此机制，并让此类初始化在前台中进行。
```java
@Override
	public void onApplicationEvent(SpringApplicationEvent event) {
		if (!ENABLED) {
			return;
		}
		if (event instanceof ApplicationEnvironmentPreparedEvent
				&& preinitializationStarted.compareAndSet(false, true)) {
			performPreinitialization();
		}
		if ((event instanceof ApplicationReadyEvent || event instanceof ApplicationFailedEvent)
				&& preinitializationStarted.get()) {
			try {
				preinitializationComplete.await();
			}
			catch (InterruptedException ex) {
				Thread.currentThread().interrupt();
			}
		}
	}
```
5. DelegatingApplicationListener，委托给在context.listener.classes环境属性下指定的其他侦听器。
> 判断是否配置环境属性context.listener.classes，如果配置了，实例化它们。如果配置了其他监听器ApplicationListener，则创建一个SimpleApplicationEventMulticaster事件多播器，将这些ApplicationListener添加到事件检索集合中。然后发布ApplicationEnvironmentPreparedEvent事件
```java
@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationEnvironmentPreparedEvent) {
			List<ApplicationListener<ApplicationEvent>> delegates = getListeners(
					((ApplicationEnvironmentPreparedEvent) event).getEnvironment());
			if (delegates.isEmpty()) {
				return;
			}
			this.multicaster = new SimpleApplicationEventMulticaster();
			for (ApplicationListener<ApplicationEvent> listener : delegates) {
				this.multicaster.addApplicationListener(listener);
			}
		}
		if (this.multicaster != null) {
			this.multicaster.multicastEvent(event);
		}
	}
```
6. FileEncodingApplicationListener
> 这是一个ApplicationListener，如果系统文件编码与环境中设置的预期值不匹配，则会阻止应用程序启动。默认情况下，它没有任何效果，但如果您将spring.mandatory_file_encoding（或该名称的一些驼峰或大写变体）设置为字符编码的名称（例如"UTF-8"），则此初始化程序在file.encoding系统属性不等于它时会引发异常。
系统属性file.encoding通常由JVM根据LANG或LC_ALL环境变量设置。它用于（与其他依赖于这些环境变量的平台相关变量一起）对JVM参数以及文件名和路径进行编码。在大多数情况下，您可以在命令行上（使用标准的JVM功能）覆盖文件编码系统属性，但也考虑将LANG环境变量设置为显式的字符编码值（例如"en_GB.UTF-8"）。
```java
@Override
	public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		String desired = environment.getProperty("spring.mandatory-file-encoding");
		if (desired == null) {
			return;
		}
		String encoding = System.getProperty("file.encoding");
		if (encoding != null && !desired.equalsIgnoreCase(encoding)) {
			if (logger.isErrorEnabled()) {
				logger.error("System property 'file.encoding' is currently '" + encoding + "'. It should be '" + desired
						+ "' (as defined in 'spring.mandatoryFileEncoding').");
				logger.error("Environment variable LANG is '" + System.getenv("LANG")
						+ "'. You could use a locale setting that matches encoding='" + desired + "'.");
				logger.error("Environment variable LC_ALL is '" + System.getenv("LC_ALL")
						+ "'. You could use a locale setting that matches encoding='" + desired + "'.");
			}
			throw new IllegalStateException("The Java Virtual Machine has not been configured to use the "
					+ "desired default character encoding (" + desired + ").");
		}
	}
```