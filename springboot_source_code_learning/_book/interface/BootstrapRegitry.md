# 【SpringBoot 源码学习】 BootstrapRegitry 详解

### 官方解读：
> 一个简单的对象注册表，在启动和环境后处理过程中一直可用，直到ApplicationContext准备就绪。
> 可以用来注册可能昂贵创建或需要在ApplicationContext可用之前共享的实例。
> 注册表使用Class作为键，这意味着只能存储给定类型的单个实例。
> addCloseListener(ApplicationListener)方法可用于添加一个监听器，当BootstrapContext已关闭并且ApplicationContext完全准备好时，可以执行操作。例如，一个实例可以选择将自己注册为常规的Spring bean，以便应用程序使用。

### 源码：
```java
public interface BootstrapRegistry {

	/**
	 * Register a specific type with the registry. If the specified type has already been
	 * registered and has not been obtained as a {@link Scope#SINGLETON singleton}, it
	 * will be replaced.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @param instanceSupplier the instance supplier
	 */
	<T> void register(Class<T> type, InstanceSupplier<T> instanceSupplier);

	/**
	 * Register a specific type with the registry if one is not already present.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @param instanceSupplier the instance supplier
	 */
	<T> void registerIfAbsent(Class<T> type, InstanceSupplier<T> instanceSupplier);

	/**
	 * Return if a registration exists for the given type.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @return {@code true} if the type has already been registered
	 */
	<T> boolean isRegistered(Class<T> type);

	/**
	 * Return any existing {@link InstanceSupplier} for the given type.
	 * @param <T> the instance type
	 * @param type the instance type
	 * @return the registered {@link InstanceSupplier} or {@code null}
	 */
	<T> InstanceSupplier<T> getRegisteredInstanceSupplier(Class<T> type);

	/**
	 * Add an {@link ApplicationListener} that will be called with a
	 * {@link BootstrapContextClosedEvent} when the {@link BootstrapContext} is closed and
	 * the {@link ApplicationContext} has been prepared.
	 * @param listener the listener to add
	 */
	void addCloseListener(ApplicationListener<BootstrapContextClosedEvent> listener);

	/**
	 * Supplier used to provide the actual instance when needed.
	 *
	 * @param <T> the instance type
	 * @see Scope
	 */
	@FunctionalInterface
	interface InstanceSupplier<T> {

		/**
		 * Factory method used to create the instance when needed.
		 * @param context the {@link BootstrapContext} which may be used to obtain other
		 * bootstrap instances.
		 * @return the instance
		 */
		T get(BootstrapContext context);

		/**
		 * Return the scope of the supplied instance.
		 * @return the scope
		 * @since 2.4.2
		 */
		default Scope getScope() {
			return Scope.SINGLETON;
		}

		/**
		 * Return a new {@link InstanceSupplier} with an updated {@link Scope}.
		 * @param scope the new scope
		 * @return a new {@link InstanceSupplier} instance with the new scope
		 * @since 2.4.2
		 */
		default InstanceSupplier<T> withScope(Scope scope) {
			Assert.notNull(scope, "Scope must not be null");
			InstanceSupplier<T> parent = this;
			return new InstanceSupplier<T>() {

				@Override
				public T get(BootstrapContext context) {
					return parent.get(context);
				}

				@Override
				public Scope getScope() {
					return scope;
				}

			};
		}

		/**
		 * Factory method that can be used to create an {@link InstanceSupplier} for a
		 * given instance.
		 * @param <T> the instance type
		 * @param instance the instance
		 * @return a new {@link InstanceSupplier}
		 */
		static <T> InstanceSupplier<T> of(T instance) {
			return (registry) -> instance;
		}

		/**
		 * Factory method that can be used to create an {@link InstanceSupplier} from a
		 * {@link Supplier}.
		 * @param <T> the instance type
		 * @param supplier the supplier that will provide the instance
		 * @return a new {@link InstanceSupplier}
		 */
		static <T> InstanceSupplier<T> from(Supplier<T> supplier) {
			return (registry) -> (supplier != null) ? supplier.get() : null;
		}

	}

	/**
	 * The scope of an instance.
	 * @since 2.4.2
	 */
	enum Scope {

		/**
		 * A singleton instance. The {@link InstanceSupplier} will be called only once and
		 * the same instance will be returned each time.
		 */
		SINGLETON,

		/**
		 * A prototype instance. The {@link InstanceSupplier} will be called whenever an
		 * instance is needed.
		 */
		PROTOTYPE

	}

}
```