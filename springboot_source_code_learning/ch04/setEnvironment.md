# 将准备好的ApplicationServletEnvironment 设置到应用上下文中

> 为此应用程序上下文设置环境。
将给定的环境委托给底层的AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner成员。
```java
	@Override
	public void setEnvironment(ConfigurableEnvironment environment) {
		super.setEnvironment(environment);
		this.reader.setEnvironment(environment);
		this.scanner.setEnvironment(environment);
	}
```