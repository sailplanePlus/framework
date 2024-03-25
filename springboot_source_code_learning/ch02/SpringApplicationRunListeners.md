# 获取SpringApplicationRunListeners

源码：
```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger,
				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args),
				this.applicationStartup);
	}
```

### 一、从"META-INF/spring.factories"中加载SpringApplicationRunListener的完全限定类名，然后使用反射创建实例，并排序。
```java
getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args)
```

```java
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

### 二、这里着重讲一下EventPublishingRunListener的实例过程

在构造方法，创建了一个简单的应用事件多播器SimpleApplicationEventMulticaster
```java
public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
```

将SpringApplication中所有的ApplicationListener添加到org.springframework.context.event.AbstractApplicationEventMulticaster.DefaultListenerRetriever#applicationListeners集合中
```java
@Override
	public void addApplicationListener(ApplicationListener<?> listener) {
		synchronized (this.defaultRetriever) {
			// 如果已经注册了代理的目标，可以显式地将其移除，以避免同一个监听器的双重调用。
			Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
			if (singletonTarget instanceof ApplicationListener) {
				this.defaultRetriever.applicationListeners.remove(singletonTarget);
			}
			this.defaultRetriever.applicationListeners.add(listener);
			this.retrieverCache.clear();
		}
	}
```