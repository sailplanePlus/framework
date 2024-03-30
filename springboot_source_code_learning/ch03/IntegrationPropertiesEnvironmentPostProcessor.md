# IntegrationPropertiesEnvironmentPostProcessor

> 用于将META-INF/spring.integration.properties的配置映射到环境中。

> 判断是否存在spring.integration.properties资源，如果存在加载配置文件，并添加到环境的属性源中。
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		Resource resource = new ClassPathResource("META-INF/spring.integration.properties");
		if (resource.exists()) {
			registerIntegrationPropertiesPropertySource(environment, resource);
		}
	}

	protected void registerIntegrationPropertiesPropertySource(ConfigurableEnvironment environment, Resource resource) {
		PropertiesPropertySourceLoader loader = new PropertiesPropertySourceLoader();
		try {
			OriginTrackedMapPropertySource propertyFileSource = (OriginTrackedMapPropertySource) loader
					.load("META-INF/spring.integration.properties", resource).get(0);
			environment.getPropertySources().addLast(new IntegrationPropertiesPropertySource(propertyFileSource));
		}
		catch (IOException ex) {
			throw new IllegalStateException("Failed to load integration properties from " + resource, ex);
		}
	}
```