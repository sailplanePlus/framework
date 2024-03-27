# 运行Spring应用程序，创建并刷新一个新的ApplicationContext

```java
public ConfigurableApplicationContext run(String... args) {
		long startTime = System.nanoTime();
		/**
		  * 1.创建引导上下文，
		  * 回调org.springframework.boot.BootstrapRegistryInitializer#initialize方法
		  * 初始化DefaultBootstrapContext
		  */
		DefaultBootstrapContext bootstrapContext = createBootstrapContext();
		ConfigurableApplicationContext context = null;
		//2.设置系统属性java.awt.headless=true
		configureHeadlessProperty();
		/**
		  * 3.创建SpringApplicationRunListeners，从META-INF/spring.factories中加载
		  * EventPublishingRunListener
		*/
		SpringApplicationRunListeners listeners = getRunListeners(args);
		//4.事件发布：ApplicationStartingEvent
		listeners.starting(bootstrapContext, this.mainApplicationClass);
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			context.setApplicationStartup(this.applicationStartup);
			prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
			}
			listeners.started(context, timeTakenToStartup);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, listeners);
			throw new IllegalStateException(ex);
		}
		try {
			Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
			listeners.ready(context, timeTakenToReady);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```