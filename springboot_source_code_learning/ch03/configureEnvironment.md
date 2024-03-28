# 配置ApplicationServletEnvironment

```java
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    
	protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    	if (this.addConversionService) {
    		environment.setConversionService(new ApplicationConversionService());
    	}
    	configurePropertySources(environment, args);
    	configureProfiles(environment, args);
    }
```
### 一、配置转化服务ConfigurableConversionService
创建ApplicationConversionService，用于在属性上执行类型转换时使用。
```java
    public ApplicationConversionService() {
		this(null);
	}
	public ApplicationConversionService(StringValueResolver embeddedValueResolver) {
		this(embeddedValueResolver, false);
	}
	private ApplicationConversionService(StringValueResolver embeddedValueResolver, boolean unmodifiable) {
		if (embeddedValueResolver != null) {
			setEmbeddedValueResolver(embeddedValueResolver);
		}
		configure(this);
		this.unmodifiable = unmodifiable;
	}
```
> 这里着重看一下configure(this)方法，
> 主要作用是使用适合大多数Spring Boot应用程序的格式化程序和转换器配置给定到ApplicationConversionService
```java
    public static void configure(FormatterRegistry registry) {
		DefaultConversionService.addDefaultConverters(registry);
		DefaultFormattingConversionService.addDefaultFormatters(registry);
		addApplicationFormatters(registry);
		addApplicationConverters(registry);
	}
```

### 二、在此应用程序的环境中添加、删除或重新排序任何属性源。
```java
    protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
		MutablePropertySources sources = environment.getPropertySources();
		if (!CollectionUtils.isEmpty(this.defaultProperties)) {
			DefaultPropertiesPropertySource.addOrMerge(this.defaultProperties, sources);
		}
		if (this.addCommandLineProperties && args.length > 0) {
			String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
			if (sources.contains(name)) {
				PropertySource<?> source = sources.get(name);
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(
						new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
				composite.addPropertySource(source);
				sources.replace(name, composite);
			}
			else {
				sources.addFirst(new SimpleCommandLinePropertySource(args));
			}
		}
	}
```
> 如果设置了defaultProperties，检测属性源PropertySources中是否包含"defalutProperties"，包含就替换掉，否则就添加。
> 如果添加的命令行参数，检测属性源PropertySources中是否包含"commandLineArgs"，包含就替换，否则就添加。

### 三、配置应用程序环境中激活（或默认激活）的配置文件
```java
    protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
	}
```