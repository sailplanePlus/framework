# 创建应用上下文 AnnotationConfigServletWebServerApplicationContext

一、 源码入口：
```java
context = createApplicationContext();
```
二、创建过程

1. 通过ApplicationContextFactory接口，创建ConfigurableApplicationContext
```java
	protected ConfigurableApplicationContext createApplicationContext() {
		return this.applicationContextFactory.create(this.webApplicationType);
	}
```
2. 根据webApplicationType为SpringApplication创建应用上下文
```java
ApplicationContextFactory DEFAULT = (webApplicationType) -> {
		try {
			for (ApplicationContextFactory candidate : SpringFactoriesLoader
					.loadFactories(ApplicationContextFactory.class, ApplicationContextFactory.class.getClassLoader())) {
				ConfigurableApplicationContext context = candidate.create(webApplicationType);
				if (context != null) {
					return context;
				}
			}
			return new AnnotationConfigApplicationContext();
		}
		catch (Exception ex) {
			throw new IllegalStateException("Unable create a default ApplicationContext instance, "
					+ "you may need a custom ApplicationContextFactory", ex);
		}
	};
```
首先加载spring.factories文件中的ApplicationContextFactory.class完全限定名，并实例化。
```
# Application Context Factories
org.springframework.boot.ApplicationContextFactory=\
org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext.Factory,\
org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext.Factory
```
```java
SpringFactoriesLoader.loadFactories(ApplicationContextFactory.class,
    ApplicationContextFactory.class.getClassLoader())
```
遍历ApplicationContextFactory，执行create方法，创建出ConfigurableApplicationContext

org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext.Factory#create
```java
	static class Factory implements ApplicationContextFactory {

		@Override
		public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
			return (webApplicationType != WebApplicationType.SERVLET) ? null
					: new AnnotationConfigServletWebServerApplicationContext();
		}

	}
```

org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext.Factory#create
```java
	static class Factory implements ApplicationContextFactory {

		@Override
		public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
			return (webApplicationType != WebApplicationType.REACTIVE) ? null
					: new AnnotationConfigReactiveWebServerApplicationContext();
		}

	}
```

根据webApplicationType，转中创建出AnnotationConfigServletWebServerApplicationContext

三、通过无参构造函数，创建AnnotationConfigServletWebServerApplicationContext

创建过程中做了很多事情，我们分开来讲。

1. 执行AnnotationConfigServletWebServerApplicationContext构造函数
```java
	public AnnotationConfigServletWebServerApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```
