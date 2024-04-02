# 初始化scanner属性 ClassPathBeanDefinitionScanner

AnnotationConfigServletWebServerApplicationContext构造方法中，创建ClassPathBeanDefinitionScanner对象
```java
	public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```
```java
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
		this(registry, true);
	}
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
	   this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
	}
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment) {

		this(registry, useDefaultFilters, environment,
				(registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
	}
```

> 为给定的 bean 工厂创建一个新的 ClassPathBeanDefinitionScanner。
如果传入的 bean 工厂不仅实现了 BeanDefinitionRegistry 接口，还实现了 ResourceLoader 接口，它将被用作默认的 ResourceLoader。这通常适用于 org.springframework.context.ApplicationContext 实现。
如果给定的是普通的 BeanDefinitionRegistry，则默认的 ResourceLoader 将是 org.springframework.core.io.support.PathMatchingResourcePatternResolver。
如果传入的 bean 工厂还实现了 EnvironmentCapable 接口，它的环境将被此读取器使用。否则，读取器将初始化并使用一个 StandardEnvironment。所有的 ApplicationContext 实现都是 EnvironmentCapable，而普通的 BeanFactory 实现则不是。
```java
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;
      //注册@Component默认的过滤器
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(resourceLoader);
	}
```

注册默认的过滤器
```java
	protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```