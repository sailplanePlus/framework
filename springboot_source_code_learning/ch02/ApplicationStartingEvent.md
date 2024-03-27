# 事件发布：ApplicationStartingEvent

### 官方解读：
> 事件在SpringApplication启动后，在Environment或ApplicationContext可用之前，尽可能早的发布，但在ApplicationListeners注册之后。
> 事件的源是SpringApplication本身，但是要注意在早期阶段不要过多的使用它的内部状态，因为可能在生命周期的后期被修改。

### 事件发布的调用过程
1. 调用org.springframework.boot.SpringApplicationRunListeners#starting方法
```java
void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {
		doWithListeners("spring.boot.application.starting", (listener) -> listener.starting(bootstrapContext),
				(step) -> {
					if (mainApplicationClass != null) {
						step.tag("mainApplicationClass", mainApplicationClass.getName());
					}
				});
	}
```
2. 遍历SpringApplicationRunListener的集合，执行starting方法
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
3. 调用org.springframework.boot.context.event.EventPublishingRunListener#starting方法
```java
    @Override
	public void starting(ConfigurableBootstrapContext bootstrapContext) {
		this.initialMulticaster
				.multicastEvent(new ApplicationStartingEvent(bootstrapContext, this.application, this.args));
	}
```
4. 事件多播器SimpleApplicationEventMulticaster发布ApplicationStartingEvent事件
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
5. 获取与SpringStartingEvent事件类型匹配的ApplicationListener监听器集合。
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
			// If result is null, the existing retriever is not fully populated yet by another thread.
			// Proceed like caching wasn't possible for this current local attempt.
		}
      //检索ApplicationListener
		return retrieveApplicationListeners(eventType, sourceType, newRetriever);
	}
```
6. 检索ApplicationStartingEvent事件的ApplicationListener监听器
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

### ApplicationStartingEvent事件具体回调的ApplicationListener
> LoggingApplicationListener 检测并返回真该使用的日志系统

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

	private void onApplicationStartingEvent(ApplicationStartingEvent event) {
		this.loggingSystem = LoggingSystem.get(event.getSpringApplication().getClassLoader());
		this.loggingSystem.beforeInitialize();
	}
```