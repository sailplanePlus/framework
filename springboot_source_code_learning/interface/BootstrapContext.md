# 【SpringBoot 源码学习】BootstrapContext 详解

### 官方解读：
* 在启动期间和环境后处理过程中，有一个简单的引导上下文可用，直到ApplicationContext准备就绪的时候。
* 提供对可能昂贵创建或需要在ApplicationContext可用之前共享的单例的延迟访问。

### 源码：
```java
public interface BootstrapContext {

	/**
	 * Return an instance from the context if the type has been registered. The instance
	 * will be created it if it hasn't been accessed previously.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @return the instance managed by the context
	 * @throws IllegalStateException if the type has not been registered
	 */
	<T> T get(Class<T> type) throws IllegalStateException;

	/**
	 * Return an instance from the context if the type has been registered. The instance
	 * will be created it if it hasn't been accessed previously.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @param other the instance to use if the type has not been registered
	 * @return the instance
	 */
	<T> T getOrElse(Class<T> type, T other);

	/**
	 * Return an instance from the context if the type has been registered. The instance
	 * will be created it if it hasn't been accessed previously.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @param other a supplier for the instance to use if the type has not been registered
	 * @return the instance
	 */
	<T> T getOrElseSupply(Class<T> type, Supplier<T> other);

	/**
	 * Return an instance from the context if the type has been registered. The instance
	 * will be created it if it hasn't been accessed previously.
	 * @param <T> the instance type
	 * @param <X> the exception to throw if the type is not registered
	 * @param type the instance type
	 * @param exceptionSupplier the supplier which will return the exception to be thrown
	 * @return the instance managed by the context
	 * @throws X if the type has not been registered
	 * @throws IllegalStateException if the type has not been registered
	 */
	<T, X extends Throwable> T getOrElseThrow(Class<T> type, Supplier<? extends X> exceptionSupplier) throws X;

	/**
	 * Return if a registration exists for the given type.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @return {@code true} if the type has already been registered
	 */
	<T> boolean isRegistered(Class<T> type);

}
```