# 触发spring.factories文件中注册的EnvironmentPostProcessor

1. 监听器EnvironmentPostProcessorApplicationListener处理ApplicationEnvironmentPreparedEvent事件
```java
	private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		SpringApplication application = event.getSpringApplication();
		for (EnvironmentPostProcessor postProcessor : getEnvironmentPostProcessors(application.getResourceLoader(),
				event.getBootstrapContext())) {
			postProcessor.postProcessEnvironment(environment, application);
		}
	}
```

2. 获取EnvironmentPostProcessor
```java
	List<EnvironmentPostProcessor> getEnvironmentPostProcessors(ResourceLoader resourceLoader,
			ConfigurableBootstrapContext bootstrapContext) {
		ClassLoader classLoader = (resourceLoader != null) ? resourceLoader.getClassLoader() : null;
		EnvironmentPostProcessorsFactory postProcessorsFactory = this.postProcessorsFactory.apply(classLoader);
		return postProcessorsFactory.getEnvironmentPostProcessors(this.deferredLogs, bootstrapContext);
	}
```

3. 创建EnvironmentPostProcessorsFactory，用来创建EnvironmentPostProcessor
```java
	static EnvironmentPostProcessorsFactory fromSpringFactories(ClassLoader classLoader) {
		return new ReflectionEnvironmentPostProcessorsFactory(classLoader,
				SpringFactoriesLoader.loadFactoryNames(EnvironmentPostProcessor.class, classLoader));
	}
```

4. 由EnvironmentPostProcessorsFactory对EnvironmentPostProcessor进行实例化
```java
	@Override
	public List<EnvironmentPostProcessor> getEnvironmentPostProcessors(DeferredLogFactory logFactory,
			ConfigurableBootstrapContext bootstrapContext) {
		Instantiator<EnvironmentPostProcessor> instantiator = new Instantiator<>(EnvironmentPostProcessor.class,
				(parameters) -> {
					parameters.add(DeferredLogFactory.class, logFactory);
					parameters.add(Log.class, logFactory::getLog);
					parameters.add(ConfigurableBootstrapContext.class, bootstrapContext);
					parameters.add(BootstrapContext.class, bootstrapContext);
					parameters.add(BootstrapRegistry.class, bootstrapContext);
				});
		return (this.classes != null) ? instantiator.instantiateTypes(this.classes)
				: instantiator.instantiate(this.classLoader, this.classNames);
	}
```
5.获取spring.factories中的EnvironmentPostProcessor，对给定的环境进行后处理
```java
postProcessor.postProcessEnvironment(environment, application);
```

> ### spring.factories文件中有配置了如下EnvironmentPostProcessor
> org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor,\
org.springframework.boot.env.RandomValuePropertySourceEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor

> 后续篇章将分别介绍每个EnvironmentPostProcessor的作用