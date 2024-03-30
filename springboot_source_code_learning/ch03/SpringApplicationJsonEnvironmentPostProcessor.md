# SpringApplicationJsonEnvironmentPostProcessor 添加spring.applicaiton.json属性源

> 它从spring.application.json 或这SPRING.APPLICATION.JSON中解析JSON，并将其作为一个Map属性源添加到Environment中。新属性被添加的优先级高于系统属性。

1. postProcessorEnvironment()方法
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		MutablePropertySources propertySources = environment.getPropertySources();
		propertySources.stream().map(JsonPropertyValue::get).filter(Objects::nonNull).findFirst()
				.ifPresent((v) -> processJson(environment, v));
	}
```

2. 遍历环境的属性源中是否存在spring.application.json属性，如果存在则创建JsonPropertyValue
```java
	static JsonPropertyValue get(PropertySource<?> propertySource) {
		for (String candidate : CANDIDATES) {
			Object value = propertySource.getProperty(candidate);
			if (value instanceof String && StringUtils.hasLength((String) value)) {
				return new JsonPropertyValue(propertySource, candidate, (String) value);
			}
		}
		return null;
	}
```

3. 将JsonPropertyValue 添加到环境的属性源中
```java
	private void addJsonPropertySource(ConfigurableEnvironment environment, PropertySource<?> source) {
		MutablePropertySources sources = environment.getPropertySources();
		String name = findPropertySource(sources);
		if (sources.contains(name)) {
			sources.addBefore(name, source);
		}
		else {
			sources.addFirst(source);
		}
	}
```