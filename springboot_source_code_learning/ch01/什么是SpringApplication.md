# 什么是SpringApplication

该类可用于从Java主方法引导和启动Spring应用程序。默认情况下，该类将执行以下步骤来引导你的应用程序：
* 创建一个适当的ApplicationContext实例
* 注册一个CommandLinePropertySource，将命令行参数公开为Spring属性
* 刷新应用程序上下文，加载所有单例bean
* 触发任何CommandlineRunner bean

在大多数情况下，静态run(Class, String[]) 方法可以直接从main方法中调用来引导饮用程序：
```java
  @Configuration
  @EnableAutoConfiguration
  public class MyApplication  {
 
    // ... Bean definitions
 
    public static void main(String[] args) {
      SpringApplication.run(MyApplication.class, args);
    }
  }
```

对于更高级的配置，可以在运行之前创建和定制一个SpringApplication实例：
```java
  public static void main(String[] args) {
    SpringApplication application = new SpringApplication(MyApplication.class);
    // ... customize application settings here
    application.run(args)
  }
```

SpringApplication可以从各种不同的来源读取bean。通常建议使用一个@Configuration类来引导你的应用程序，但是，你可可以设置源：
> 由AnnotatedBeanDefinitionReader加载的完全限定类名
> 由XmlBeanDefinitionReader加载XML资源，或者GroovyBeanDefinitionReader加载groovy脚本
> 要被ClassPathBeanDefinitionScanner扫描的包名称

配置属性也绑定到SpringApplication。这使得动态设置SpringApplication属性成为可能，比如额外的资源("spring.main.sources" - a CSV list)、表示web环境的标志("spring.main.web-application-type=none")或者关闭banner的标志 ("spring.main.banner-mode=off")