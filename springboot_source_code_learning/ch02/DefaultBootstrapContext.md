# 创建引导上下文 DefaultBootstrapContext

创建引导上下文DefaultBootstrapContext，在启动和环境后处理期间可用，直到ApplicationContext准备好为止。回调[org.springframework.boot.BootstrapRegistryInitializer](interface/BootstrapRegistryInitializer.md)#initialize方法，完成对DefaultBootstrapContext的设置。
```java
private DefaultBootstrapContext createBootstrapContext() {
		DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
		this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
		return bootstrapContext;
	}
```
DefaultBootstrapContext实现了BootstrapRegistry、BootstrapContext接口，具体作用请查阅接口详解。

所以可以在initialize方法中可以做几件事情：
> 向DefaultBootstrapContext注册指定的Class到注册表中
> 添加一个ApplicationListener，当BootstrapContext关闭且ApplicationContext已准备就绪时，将发布BootstrapContextClosedEvent事件调用它。