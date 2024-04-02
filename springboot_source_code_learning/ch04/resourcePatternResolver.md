# 初始化resourcePatternResolver属性 ServletContextResourcePatternResolver

1. 执行AbstractApplicationContext无参数构造函数
```java
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
```
初始化属性resourcePatternResolver，创建资源解析器
```java
	@Override
	protected ResourcePatternResolver getResourcePatternResolver() {
		return new ServletContextResourcePatternResolver(this);
	}
```