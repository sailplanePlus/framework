# SystemEnvironmentPropertySourceEnvionmentPostProcessor替换属性源"systemEnvironment"

> 这是一个EnvironmentPostProcessor，它用SystemEnvironmentPropertySourceEnvironmentPostProcessor.OriginAwareSystemEnvironmentPropertySource替换了systemEnvironment SystemEnvironmentPropertySource，后者可以跟踪每个系统环境属性的SystemEnvironmentOrigin。


1. 首先来看postProcessorEnvironment方法，如果环境属性源存在"systemEnvironment"则替换其对应的SystemEnvironmentPropertySource
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		String sourceName = StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME;
		PropertySource<?> propertySource = environment.getPropertySources().get(sourceName);
		if (propertySource != null) {
			replacePropertySource(environment, sourceName, propertySource, application.getEnvironmentPrefix());
		}
	}
```

2. 执行替换方法replacePropertySource()
```java
	private void replacePropertySource(ConfigurableEnvironment environment, String sourceName,
			PropertySource<?> propertySource, String environmentPrefix) {
		Map<String, Object> originalSource = (Map<String, Object>) propertySource.getSource();
		SystemEnvironmentPropertySource source = new OriginAwareSystemEnvironmentPropertySource(sourceName,
				originalSource, environmentPrefix);
		environment.getPropertySources().replace(sourceName, source);
	}
```
将SystemEnvironmentPropertySource替换为OriginAwareSystemEnvironmentPropertySource