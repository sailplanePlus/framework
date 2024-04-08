# 事件发布：AvailabilityChangeEvent

### 官方解读：
> 当应用程序的可用状态发生变化时发送的ApplicationEvent。任何应用程序组件都可以发送这样的事件来更新应用程序的状态。

> 可用于将AvailabilityChangeEvent发布到给定应用程序上下文的便利方法。
```java
public static <S extends AvailabilityState> void publish(ApplicationContext context, S state) {
	Assert.notNull(context, "Context must not be null");
	publish(context, context, state);
}
```

### AvailabilityChangeEvent事件具体回调的ApplicationListener
1. ApplicationAvailabilityBean
```java
@Override
public void onApplicationEvent(AvailabilityChangeEvent<?> event) {
	Class<? extends AvailabilityState> type = getStateType(event.getState());
	if (this.logger.isDebugEnabled()) {
		this.logger.debug(getLogMessage(type, event));
	}
	this.events.put(type, event);
}
```