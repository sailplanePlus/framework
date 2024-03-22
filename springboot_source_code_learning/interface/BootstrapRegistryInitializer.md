# 【SpringBoot 源码学习】BootstrapRegistryInitializer 详解

## 初识BootstrapRegistryInitializer
**BootstrapRegistryInitializer** 接口的源码：
```java
@FunctionalInterface
public interface BootstrapRegistryInitializer {

	/**
	 * Initialize the given {@link BootstrapRegistry} with any required registrations.
	 * @param registry the registry to initialize
	 */
	void initialize(BootstrapRegistry registry);
}
```
可以看到**BootstrapRegistryInitializer**接口被**@FunctionalInterface**注解修饰。
> @FunctionalInterface 是 Java 8 中引入的一个注解，用于标识一个函数式接口。函数式接口是只有一个抽象方法的接口，常用于实现 Lambda 表达式和方法引用。
使用 @FunctionalInterface 注解可以向编译器指示该接口是一个函数式接口，从而在编译时进行类型检查，确保该接口 只包含一个抽象方法。此外，该注解还可以为函数式接口生成特殊的方法，如默认方法（default method）和 静态方法（static method）

**BootstrapRegistryInitializer**接口只定义了**initialize**方法，该方法只有一个参数是**BootstrapRegistry**

### BootstrapRegistryInitializer 是Spring Boot 2.4引入的一个新接口，用于允许在Spring应用程序上下文创建之前对BootStrapRegitry进行自定义注册。它的作用主要是在Spring应用程序启动过程中允许开发人员注册自定义的组件。

这个接口的工作原理可以通过以下步骤概括：

1. **实现接口**：你需要编写一个类来实现`BootstrapRegistryInitializer`接口，并重写其中的方法，以便在Spring应用程序上下文初始化之前执行一些自定义的逻辑。
2. **注册自定义组件**：在实现的`initialize`方法中，你可以使用`BootstrapRegistry`提供的方法来注册自定义的组件，例如注册自定义的bean定义、条件注解等。
3. **注册到Spring应用程序**：最后，你要确保这个实现了`BootstrapRegistryInitializer`接口的类注册到Spring应用程序中。两种方式：spring.factories;
SpringApplication.addBootstrapRegistryInitializer(BootstrapRegistryInitializer)

总的来说，`BootstrapRegistryInitializer`的工作原理是在Spring应用程序启动时，在Spring应用程序上下文创建之前，允许开发人员注册自定义的组件或执行其他一些预处理逻辑。

具体执行`initialize`方法代码：
```java
DefaultBootstrapContext bootstrapContext = createBootstrapContext();
```

```java
private DefaultBootstrapContext createBootstrapContext() {
		DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
		this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
		return bootstrapContext;
	}
```