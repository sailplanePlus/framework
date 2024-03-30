# Environment 准备的后续逻辑

1. 移动"defaultProperties" 属性源至最后
```java
	public static void moveToEnd(ConfigurableEnvironment environment) {
		moveToEnd(environment.getPropertySources());
	}
	
	public static void moveToEnd(MutablePropertySources propertySources) {
		PropertySource<?> propertySource = propertySources.remove(NAME);
		if (propertySource != null) {
			propertySources.addLast(propertySource);
		}
	}
```

2. 绑定Envrionmnet到SpringApplication
```java
bindToSpringApplication(environment);
```

3. 将给定环境转换为给定的StandardEnvironment类型。
```java
	if (!this.isCustomEnvironment) {
		EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
		environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
	}
```

4. 将‘configurationProperties’属性源移动值最前方，将环境管理的每个PropertySource适配为一个ConfigurationPropertySource，并允许经典的PropertySourcesPropertyResolver调用使用配置属性名进行解析。
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

5. 设置系统属性spring.beaninfo.ignore
```java
	private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
		if (System.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
			Boolean ignore = environment.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME,
					Boolean.class, Boolean.TRUE);
			System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME, ignore.toString());
		}
	}
```