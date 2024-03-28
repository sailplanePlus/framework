# 获取或者创建环境对象ConfigurableEnvironment

```java
    ConfigurableEnvironment environment = getOrCreateEnvironment();
```

```java
    private ConfigurableEnvironment getOrCreateEnvironment() {
    	if (this.environment != null) {
    		return this.environment;
    	}
    	switch (this.webApplicationType) {
    		case SERVLET:
    			return new ApplicationServletEnvironment();
    		case REACTIVE:
    			return new ApplicationReactiveWebEnvironment();
    		default:
    			return new ApplicationEnvironment();
    	}
    }
```
### 一、通过我们的webApplicationType判断，通过无参构造方法创建ApplicationServletEnvironment
```java
	public AbstractEnvironment() {
		this(new MutablePropertySources());
	}
```
### 二、构造过程中回调customizePropertySources(propertySouces)，以允许子类根据需要操作PropertySources实例。
```java
    protected AbstractEnvironment(MutablePropertySources propertySources) {
    		this.propertySources = propertySources;
    		this.propertyResolver = createPropertyResolver(propertySources);
    		customizePropertySources(propertySources);
	}
```
1. 创建属性解析器ConfigurationPropertySourcesPropertyResolver
```java
    @Override
    protected ConfigurablePropertyResolver createPropertyResolver(MutablePropertySources propertySources) {
    	return ConfigurationPropertySources.createPropertyResolver(propertySources);
    }
```
2.回调customizePropertySources(propertySources)方法
```java
	@Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
		propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
		if (jndiPresent && JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
			propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
		}
		super.customizePropertySources(propertySources);
	}
```
设置propertySouce
* servletContextInitParams
* servletConfigInitParams
* systemProperties
* systemEnvironment
> 注意Servlet相关的属性源在这个阶段只是被添加为存根，并没有将具体的属性添加进来