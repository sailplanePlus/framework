# 附加属性源"configurationProperties"
检查属性源PropertySources中是否存在"configurationProperties"源，并且是ConfigurationPropertySourcesPropertySource类型的，否则就创建一个新的ConfigurationPropertySourcesPropertySource，添加到属性源的首位。
```java
public static void attach(Environment environment) {
	Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
	MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
	PropertySource<?> attached = getAttached(sources);
	if (attached == null || !isUsingSources(attached, sources)) {
		attached = new ConfigurationPropertySourcesPropertySource(ATTACHED_PROPERTY_SOURCE_NAME,
				new SpringConfigurationPropertySources(sources));
	}
	sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
	sources.addFirst(attached);
}
```

> 附加ConfirgurationPropertySource支持到指定的环境中。这会将环境管理的每个PropertySource调整为ConfigurationPropertySource，并允许经典的PropertySourcesPropertyResolver调用以使用配置属性名称进行解析。