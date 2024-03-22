# 创建SpringApplication

## 调用SpringApplication中的静态run方法
```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
}
```
## 创建一个新的SpringApplication实例。应用程序上下文将从指定的主源加载bean。实例可以是调用run(String...)之前自定义。
```java
public SpringApplication(ResourceLoader resourceLoader, 
    Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new
		  LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
      setInitializers((Collection)
        getSpringFactoriesInstances(ApplicationContextInitializer.class));
      setListeners((Collection)
        getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
}
```
1. 设置资源加载器resourceLoader
    ```java
    this.resourceLoader = resourceLoader;
    ```
2. 将启动类放入primarySources
    ```java
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    ```
3. 推测web应用类型webApplicationType (NONE、REACTIVE、SERVLET)
    ```java
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    ```
4. 从spring.factories中获取[BootStrapRegistryInitializer](/interface/BootstrapRegistryInitializer.md)
    ```java
    this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    ```
5. 从spring.factories中获取ApplicationContextInitializer（在应用程序上下文初始化之前执行一些自定义逻辑）
    ```java
    setInitializers((Collection)
        getSpringFactoriesInstances(ApplicationContextInitializer.class));
    ```
6. 从spring.factories中获取ApplicationListener
    ```java
    setListeners((Collection)
        getSpringFactoriesInstances(ApplicationListener.class));
    ```
7. 推测出main类（main()方法所在的类）
    ```java
    this.mainApplicationClass = deduceMainApplicationClass();
    ```

### 关于方法 getSpringFactoriesInstances(Class<T> type) 的解释
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, 
    Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(
	       SpringFactoriesLoader.loadFactoryNames(type,classLoader));
		List<T> instances = createSpringFactoriesInstances
		  (type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```
#### 使用给定的类加载器，从"META-INF/spring.factories"加载给定类型的工厂实现的完全限定类名
```java
    Set<String> names = new LinkedHashSet<>(
        SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //获取给定类型的完全限定类名
    public static List<String> loadFactoryNames(Class<?>factoryType,
        @Nullable ClassLoader classLoader) {
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
		String factoryTypeName = factoryType.getName();
		return loadSpringFactories(classLoaderToUse)
		  .getOrDefault(factoryTypeName, Collections.emptyList());
	}
	//首先从cache中获取，如果没有则遍历spring.factories配置，获取给定类型的完全限定类名，并添加到cache
	private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
		Map<String, List<String>> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		result = new HashMap<>();
		try {
			Enumeration<URL> urls = classLoader.
			 getResources(FACTORIES_RESOURCE_LOCATION);
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils
				    .loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryTypeName = ((String) entry.getKey()).trim();
					String[] factoryImplementationNames = StringUtils
							.commaDelimitedListToStringArray((String) entry.getValue());
					for (String factoryImplementationName : factoryImplementationNames) {
						result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
								.add(factoryImplementationName.trim());
					}
				}
			}

			// Replace all lists with unmodifiable lists containing unique elements
			result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
					.collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
			cache.put(classLoader, result);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
		return result;
	}
```
#### 获取到完全限定类名后，使用反射创建实例
```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
			ClassLoader classLoader, Object[] args, Set<String> names) {
		List<T> instances = new ArrayList<>(names.size());
		for (String name : names) {
			try {
				Class<?> instanceClass = ClassUtils.forName(name, classLoader);
				Assert.isAssignable(type, instanceClass);
				Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
				T instance = (T) BeanUtils.instantiateClass(constructor, args);
				instances.add(instance);
			}
			catch (Throwable ex) {
				throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
			}
		}
		return instances;
	}
```
#### 将创建的实例进行排序
```java
AnnotationAwareOrderComparator.sort(instances);
```