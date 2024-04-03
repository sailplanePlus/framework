# 事件发布： ApplicationPreparedEvent

### 官方解读：
> 应用程序上下文已经准备就绪，但尚未刷新。此阶段加载bean定义，并且环境已准备就绪以供使用。

将SpringApplication中的事件监听器ApplicationListener添加到ApplicationContext中，然后发布ApplicationPreparedEvent事件
```java
@Override
public void contextLoaded(ConfigurableApplicationContext context) {
	for (ApplicationListener<?> listener : this.application.getListeners()) {
		if (listener instanceof ApplicationContextAware) {
			((ApplicationContextAware) listener).setApplicationContext(context);
		}
		context.addApplicationListener(listener);
	}
	this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
}
```

### ApplicationPreparedEvent事件具体回调的ApplicationListener
1. EnvironmentPostProcessorApplicationListener，将所有延迟日志切换到它们提供的目的地。
```java
private void onApplicationPreparedEvent() {
	finish();
}

private void finish() {
	this.deferredLogs.switchOverAll();
}
```
2. LonggingApplicationListener，向一级缓存中，注册日志相关的单例对象
```java
private void onApplicationPreparedEvent(ApplicationPreparedEvent event) {
	ConfigurableApplicationContext applicationContext = event.getApplicationContext();
	ConfigurableListableBeanFactory beanFactory = applicationContext.getBeanFactory();
	if (!beanFactory.containsBean(LOGGING_SYSTEM_BEAN_NAME)) {
		beanFactory.registerSingleton(LOGGING_SYSTEM_BEAN_NAME, this.loggingSystem);
	}
	if (this.logFile != null && !beanFactory.containsBean(LOG_FILE_BEAN_NAME)) {
		beanFactory.registerSingleton(LOG_FILE_BEAN_NAME, this.logFile);
	}
	if (this.loggerGroups != null && !beanFactory.containsBean(LOGGER_GROUPS_BEAN_NAME)) {
		beanFactory.registerSingleton(LOGGER_GROUPS_BEAN_NAME, this.loggerGroups);
	}
	if (!beanFactory.containsBean(LOGGING_LIFECYCLE_BEAN_NAME) && applicationContext.getParent() == null) {
		beanFactory.registerSingleton(LOGGING_LIFECYCLE_BEAN_NAME, new Lifecycle());
	}
}
```