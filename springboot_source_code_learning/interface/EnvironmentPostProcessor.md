# 【SpringBoot 源码学习】EnvironmentPostProcessor 详解

1. EnvironmentPostProcessor允许应用在应用程序上下文刷新之前对应用程序的环境进行定制。EnvironmentPostProcessor实现必须在META-INF/spring.factories中注册，使用该类的完全限定名作为键。如果实现希望按特定顺序调用，可以实现Ordered接口或使用@Order注解。
```java
@FunctionalInterface
public interface EnvironmentPostProcessor {

	/**
	 * Post-process the given {@code environment}.
	 * @param environment the environment to post-process
	 * @param application the application to which the environment belongs
	 */
	void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application);

}
```

2. EnvironmentPostProcessor实现可以选择接受以下构造函数参数：
* DeferredLogFactory：一个工厂，可用于创建日志记录器，其输出延迟到应用程序准备好为止（允许环境本身配置日志级别）
* Log：将输出延迟到应用程序完全准备好之后到日志
* ConfigurableBootstrapContext：一个引导上下文，可用于存储创建成本较高或需要共享的对象（BootstrapContext或BootstrapRegistry也可以使用）

3. 这里你可能会有些疑惑，为什么还限制我EnvironmentPostProcessor的具体参数呢？不得不说spring的单一职责原则（SRP）应用的非常好。spring专门提供了EnvironmentPostProcessorsFactory，专门由EnvironmentPostProcessorApplicationListener用来创建EnvironmentPostProcessor实例的工厂接口。我们来看一下时如何创建的：
```java
	static EnvironmentPostProcessorsFactory fromSpringFactories(ClassLoader classLoader) {
		return new ReflectionEnvironmentPostProcessorsFactory(classLoader,
				SpringFactoriesLoader.loadFactoryNames(EnvironmentPostProcessor.class, classLoader));
	}
```
我们看到创建了实例工厂ReflectionEnvironmentPostProcessorsFactory，通过org.springframework.boot.env.ReflectionEnvironmentPostProcessorsFactory#getEnvironmentPostProcessors方法，我们就可以理解为什么限制EnvironmentPostProcessor的具体参数了。
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
这个方法是负责实例化EnvironmentPostProcessor的，我们来看具体的实例化过程：

首先根据EnvironmentPostProcessor的完全限定类名，创建提供类型的Supplier
```java
	public List<T> instantiate(ClassLoader classLoader, Collection<String> names) {
		Assert.notNull(names, "Names must not be null");
		return instantiate(names.stream().map((name) -> TypeSupplier.forName(classLoader, name)));
	}
	
	static TypeSupplier forName(ClassLoader classLoader, String name) {
	   return new TypeSupplier() {
    
    		@Override
    		public String getName() {
    			return name;
    		}
    
    		@Override
    		public Class<?> get() throws ClassNotFoundException {
    			return ClassUtils.forName(name, classLoader);
    		}
	   };
	}
```

然后遍历TypeSupplier执行instantiate方法完成EnvironmentPostProcessor的实例化
```java
	private List<T> instantiate(Stream<TypeSupplier> typeSuppliers) {
		List<T> instances = typeSuppliers.map(this::instantiate).collect(Collectors.toList());
		AnnotationAwareOrderComparator.sort(instances);
		return Collections.unmodifiableList(instances);
	}
	private T instantiate(TypeSupplier typeSupplier) {
		try {
			Class<?> type = typeSupplier.get();
			Assert.isAssignable(this.type, type);
			return instantiate(type);
		}
		catch (Throwable ex) {
			this.failureHandler.handleFailure(this.type, typeSupplier.getName(), ex);
			return null;
		}
	}
```

获取EnvironmentPostProcessor的构造函数，解析构造器的参数
```java
private T instantiate(Class<?> type) throws Exception {
		Constructor<?>[] constructors = type.getDeclaredConstructors();
		Arrays.sort(constructors, CONSTRUCTOR_COMPARATOR);
		for (Constructor<?> constructor : constructors) {
			Object[] args = getArgs(constructor.getParameterTypes());
			if (args != null) {
				ReflectionUtils.makeAccessible(constructor);
				return (T) constructor.newInstance(args);
			}
		}
		throw new IllegalAccessException("Class [" + type.getName() + "] has no suitable constructor");
	}
```

来看一下getArgs方法中的getAvailableParameter方法：
```java
	private Object[] getArgs(Class<?>[] parameterTypes) {
		Object[] args = new Object[parameterTypes.length];
		for (int i = 0; i < parameterTypes.length; i++) {
			Function<Class<?>, Object> parameter = getAvailableParameter(parameterTypes[i]);
			if (parameter == null) {
				return null;
			}
			args[i] = parameter.apply(this.type);
		}
		return args;
	}
```
然后继续深入查看getAvailableParameter方法：
```java
	private Function<Class<?>, Object> getAvailableParameter(Class<?> parameterType) {
		for (Map.Entry<Class<?>, Function<Class<?>, Object>> entry : this.availableParameters.entrySet()) {
			if (entry.getKey().isAssignableFrom(parameterType)) {
				return entry.getValue();
			}
		}
		return null;
	}
```
我们可以看到遍历availableParameters集合，校验构造器参数的类型是否与之前设定好的类型匹配，如果匹配，获取创建构造参数的Function<Class<?>, Object>，执行apply方法，完成构造参数的赋值
```java
args[i] = parameter.apply(this.type);
```



