# 事件发布：ApplicationStartedEvent

### 官方解读：
> 在刷新应用上下文之后，但在调用任何应用程序和命令行运行程序之前发布的事件。

```java
@Override
public void started(ConfigurableApplicationContext context, Duration timeTaken) {
    //发布事件ApplicationStartedEvent
	context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context, timeTaken));
	//发布事件AvailabilityChangeEvent
	AvailabilityChangeEvent.publish(context, LivenessState.CORRECT);
}
```

### ApplicationStartedEvent事件具体回调的ApplicationListener
1. BackgroundPreinitializer(啥都没做)
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

2. DelegatingApplicationListener(啥都没做)
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

### 发布事件：AvailabilityChangeEvent
> 应用程序的“活跃”状态： LivenessState.CORRECT
> 代表着：应用程序正在运行，其内部状态是正确的。