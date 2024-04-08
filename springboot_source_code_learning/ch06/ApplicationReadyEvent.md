# 事件发布：ApplicationReadyEvent

### 官方解读：
> 尽可能晚地发布事件，以表明应用程序已准备好为请求提供服务。事件的源是SpringApplication本身，但要注意不要修改其内部状态，因为到那时所有初始化步骤都已完成。
```java
@Override
public void ready(ConfigurableApplicationContext context, Duration timeTaken) {
   //发布事件：ApplicationReadyEvent
	context.publishEvent(new ApplicationReadyEvent(this.application, this.args, context, timeTaken));
	//发布事件：AvailabilityChangeEvent
	AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
}
```

### ApplicationReadyEvent事件具体回调的ApplicationListener
1. SpringApplicationAdminMXBeanRegistrar
```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationReadyEvent) {
		onApplicationReadyEvent((ApplicationReadyEvent) event);
	}
	if (event instanceof WebServerInitializedEvent) {
		onWebServerInitializedEvent((WebServerInitializedEvent) event);
	}
}

void onApplicationReadyEvent(ApplicationReadyEvent event) {
	if (this.applicationContext.equals(event.getApplicationContext())) {
		this.ready = true;
	}
}
```

2. BackgroundPreinitializer
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

### 再次发布事件： AvailabilityChangeEvent
> 应用程序的“就绪”状态：ReadinessState.ACCEPTING_TRAFFIC
> 代表着：应用程序已准备好接收流量。