# 准备应用上下文ApplicationContext

```java
    public ConfigurableApplicationContext run(String... args) {
		...
		//创建应用上下文 AnnotationConfigServletWebServerApplicationContext
		ConfigurableApplicationContext context = createApplicationContext();
		context.setApplicationStartup(this.applicationStartup);
		//准备应用上下文
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
		...
	}
```

```java
    private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		//为应用上下文设置Environment
		//将给定的环境委托给底层AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner成员
		context.setEnvironment(environment);
		//在ApplicationContext中应用任何相关的后处理
		postProcessApplicationContext(context);
		//在刷新上下文之前应用任何ApplicationContextInitializers
		applyInitializers(context);
		//事件发布 ApplicationContextInitializedEvent
		listeners.contextPrepared(context);
		//引导上下文关闭，事件发布BootstrapContextClosedEvent
		bootstrapContext.close(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		//添加特定的单例bean
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
			((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
			if (beanFactory instanceof DefaultListableBeanFactory) {
				((DefaultListableBeanFactory) beanFactory)
						.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
			}
		}
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
		//加载源文件
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		//将bean加载到ApplicationContext
		load(context, sources.toArray(new Object[0]));
		//将ApplicationListener添加至ApplicationContext
		//事件发布 ApplicationPreparedContext
		listeners.contextLoaded(context);
	}
```