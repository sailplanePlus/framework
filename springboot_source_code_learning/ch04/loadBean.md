# SpringApplication.load()方法，加载bean到ApplicationContext

一、 加载源文件
```java
    Set<Object> sources = getAllSources();

	public Set<Object> getAllSources() {
		Set<Object> allSources = new LinkedHashSet<>();
		if (!CollectionUtils.isEmpty(this.primarySources)) {
			allSources.addAll(this.primarySources);
		}
		if (!CollectionUtils.isEmpty(this.sources)) {
			allSources.addAll(this.sources);
		}
		return Collections.unmodifiableSet(allSources);
	}
```
> 官方解读：
> SpringApplications可以从各种不同的来源读取bean。通常建议使用一个@Configuration类来引导你的应用程序，但是，你也可以设置源:
· 使用AnnotatedBeanDefinitionReader加载的完全限定类名
· 使用XmlBeanDefinitionReader加载XML资源
· 使用GroovyBeanDefinitionReader加载groovy脚本
· 使用ClassPathBeanDefinitionScanner扫码包

二、加载bean
```java
load(context, sources.toArray(new Object[0]));

protected void load(ApplicationContext context, Object[] sources) {
	if (logger.isDebugEnabled()) {
		logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
	}
	BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
	if (this.beanNameGenerator != null) {
		loader.setBeanNameGenerator(this.beanNameGenerator);
	}
	if (this.resourceLoader != null) {
		loader.setResourceLoader(this.resourceLoader);
	}
	if (this.environment != null) {
		loader.setEnvironment(this.environment);
	}
	loader.load();
}
```
把源加载到reader
```java
void load() {
	for (Object source : this.sources) {
		load(source);
	}
}
//根据源类型进行加载
private void load(Object source) {
	Assert.notNull(source, "Source must not be null");
	if (source instanceof Class<?>) {
		load((Class<?>) source);
		return;
	}
	if (source instanceof Resource) {
		load((Resource) source);
		return;
	}
	if (source instanceof Package) {
		load((Package) source);
		return;
	}
	if (source instanceof CharSequence) {
		load((CharSequence) source);
		return;
	}
	throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

注册BeanDefinition，将类的元数据BeanDefinition注册到beanDefinitionMap
```java
private void load(Class<?> source) {
	if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
		// Any GroovyLoaders added in beans{} DSL can contribute beans here
		GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
		((GroovyBeanDefinitionReader) this.groovyReader).beans(loader.getBeans());
	}
	//检查bean是否符合注册条件
	if (isEligible(source)) {
	   //注册BeanDefinition
		this.annotatedReader.register(source);
	}
}
```