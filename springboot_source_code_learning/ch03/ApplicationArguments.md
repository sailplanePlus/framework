# 创建ApplicationArguments，并解析命令行参数

### 一、创建DefaultApplicationArguments
```java
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
```

```java
    public DefaultApplicationArguments(String... args) {
		Assert.notNull(args, "Args must not be null");
		this.source = new Source(args);
		this.args = args;
	}
```
### 二、创建DefaultApplicationArguments内部类Source
```java
    private static class Source extends SimpleCommandLinePropertySource {

		Source(String[] args) {
			super(args);
		}

		@Override
		public List<String> getNonOptionArgs() {
			return super.getNonOptionArgs();
		}

		@Override
		public List<String> getOptionValues(String name) {
			return super.getOptionValues(name);
		}

	}
```
### 三、执行父类SimpleCommandLinePropertySource构造方法
创建SimpleCommandLineArgsParser解析命令行参数，解析一个String[]类型的命令行参数以填充一个CommandLineArgs对象。
```java
    public SimpleCommandLinePropertySource(String... args) {
		super(new SimpleCommandLineArgsParser().parse(args));
	}
```
解析方法parse():
```java
    public CommandLineArgs parse(String... args) {
		CommandLineArgs commandLineArgs = new CommandLineArgs();
		for (String arg : args) {
			if (arg.startsWith("--")) {
				String optionText = arg.substring(2);
				String optionName;
				String optionValue = null;
				int indexOfEqualsSign = optionText.indexOf('=');
				if (indexOfEqualsSign > -1) {
					optionName = optionText.substring(0, indexOfEqualsSign);
					optionValue = optionText.substring(indexOfEqualsSign + 1);
				}
				else {
					optionName = optionText;
				}
				if (optionName.isEmpty()) {
					throw new IllegalArgumentException("Invalid argument syntax: " + arg);
				}
				commandLineArgs.addOptionArg(optionName, optionValue);
			}
			else {
				commandLineArgs.addNonOptionArg(arg);
			}
		}
		return commandLineArgs;
	}
```

1. 处理选项参数，必须遵循精确的语法：
> --optName[=optValue]
> 也就是说，选项必须以"--"作为前缀，可以指定值，也可以不指定。如果指定来值，名称和值必须通过等号("=")分割，且不能有空格。值可以是空字符串。
> 示例：
> --foo
> --foo=
> --foo=""
> --foo=bar
> --foo="bar then baz"
> --foo=bar,baz,biz
2. 处理非选项参数
> 在命令行中指定的任何和所有没有"--"选项前缀的参数都将被视为“非选项参数”，并通过CommandLineArgs.getNonOptionArgs()方法提供

### 四、将解析好的CommandLineArgs对象设置到source属性，name为"commandLineArgs"
```java
public static final String COMMAND_LINE_PROPERTY_SOURCE_NAME = "commandLineArgs";

public CommandLinePropertySource(T source) {
	super(COMMAND_LINE_PROPERTY_SOURCE_NAME, source);
}
	
public EnumerablePropertySource(String name, T source) {
	super(name, source);
}
public PropertySource(String name, T source) {
    Assert.hasText(name, "Property source name must contain at least one character");
    Assert.notNull(source, "Property source must not be null");
    this.name = name;
    this.source = source;
}
```