# 初始化beanFactory属性 DefaultListableBeanFactory

执行GenericApplicationContext无参构造函数
```java
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```
初始化beanFactory，创建Bean工厂，我们来看一下创建DefaultListableBeanFactory时作了哪些事情

* 配置在依赖检查和自动装配时应该忽略的依赖接口
* 初始化创建bean实例的策略 CglibSubClassingInstantiationStrategy
```java
	public AbstractAutowireCapableBeanFactory() {
		super();
		//配置在依赖检查和自动装配时应该忽略的依赖接口
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
		//初始化创建bean实例的策略
		if (NativeDetector.inNativeImage()) {
			this.instantiationStrategy = new SimpleInstantiationStrategy();
		}
		else {
			this.instantiationStrategy = new CglibSubclassingInstantiationStrategy();
		}
	}
```