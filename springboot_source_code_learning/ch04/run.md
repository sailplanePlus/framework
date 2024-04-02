# 准备应用上下文ApplicationContext

```java
    public ConfigurableApplicationContext run(String... args) {
		...
		//创建应用上下文 AnnotationConfigServletWebServerApplicationContext
		ConfigurableApplicationContext context = createApplicationContext();
		context.setApplicationStartup(this.applicationStartup);
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
		...
	}
```