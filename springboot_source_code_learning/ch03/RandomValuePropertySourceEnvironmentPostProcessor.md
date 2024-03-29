# RandomValuePropertySourceEnvironmentPostProcessor向环境中添加属性源"random"：RandomValuePropertySource

1. postProcessEnvironment方法
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		RandomValuePropertySource.addToEnvironment(environment, this.logger);
	}
```

2. 