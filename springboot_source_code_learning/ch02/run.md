# 启动SpringApplication应用程序

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
		
		...
	}
```