# 准备应用程序环境 Environment

```java
public ConfigurableApplicationContext run(String... args) {
		...
		
		try {
		   //创建ApplicationArguments，并解析命令行参数String[]args
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			//准备一个可配置的环境对象
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			//配置环境属性spring.beaninfo.ignore
			configureIgnoreBeanInfo(environment);
            ...
		}
		...
	}
```

创建并配置环境prepareEnvironment(listenrers, bootstrapContext, applicationArguments)：
```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
		listeners.environmentPrepared(bootstrapContext, environment);
		DefaultPropertiesPropertySource.moveToEnd(environment);
		Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
				"Environment prefix cannot be set via properties.");
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
			environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```