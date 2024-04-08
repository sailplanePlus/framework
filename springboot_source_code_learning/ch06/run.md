# 应用上下文刷完后续处理

```java
afterRefresh(context, applicationArguments);
Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
if (this.logStartupInfo) {
	new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
}
listeners.started(context, timeTakenToStartup);
callRunners(context, applicationArguments);

listeners.ready(context, timeTakenToReady);
```


一、执行afterRefresh()方法
```java
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
}
```
> 这一步什么都没做，留作扩展使用。

二、