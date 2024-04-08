# 【SpringBoot 源码学习】ApplicationRunner 详解

> ApplicationRunner接口是Spring Boot中的一个回调接口，用于在Spring Boot应用程序启动后执行特定的逻辑。其作用是让开发人员可以在Spring Boot 应用程序完全启动后执行一些初始化任务、数据加载或其他逻辑操作。

```java
@FunctionalInterface
public interface ApplicationRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming application arguments
	 * @throws Exception on error
	 */
	void run(ApplicationArguments args) throws Exception;

}
```
具体来说，ApplicationRunner 接口定义了一个 run() 方法，该方法在 Spring Boot 应用程序启动完成后自动调用。开发人员可以在 run() 方法中编写需要在应用程序启动后立即执行的逻辑代码，例如初始化数据库、加载初始数据、发送通知等。

调用位置：
```java
callRunners(context, applicationArguments);

private void callRunners(ApplicationContext context, ApplicationArguments args) {
	List<Object> runners = new ArrayList<>();
	runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
	runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
	AnnotationAwareOrderComparator.sort(runners);
	for (Object runner : new LinkedHashSet<>(runners)) {
		if (runner instanceof ApplicationRunner) {
			callRunner((ApplicationRunner) runner, args);
		}
		if (runner instanceof CommandLineRunner) {
			callRunner((CommandLineRunner) runner, args);
		}
	}
}
```