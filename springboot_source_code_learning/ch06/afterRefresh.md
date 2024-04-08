# 调用afterRefresh()方法

```java
afterRefresh(context, applicationArguments);

protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
}
```

> 在上下文刷新后调用。这一步什么都没做，留给子类做扩展用。