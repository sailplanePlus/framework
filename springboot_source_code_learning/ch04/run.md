# 准备应用上下文ApplicationContext

```java
    public ConfigurableApplicationContext run(String... args) {
		...
		ConfigurableApplicationContext context = createApplicationContext();
		context.setApplicationStartup(this.applicationStartup);
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
		...
	}
```