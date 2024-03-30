# RandomValuePropertySourceEnvironmentPostProcessor向环境中添加属性源"random"：RandomValuePropertySource

1. postProcessEnvironment方法
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		RandomValuePropertySource.addToEnvironment(environment, this.logger);
	}
```

2. 执行RandomValuePropertySource.addToEnvironment(environment, this.logger)方法
```java
    static void addToEnvironment(ConfigurableEnvironment environment, Log logger) {
		MutablePropertySources sources = environment.getPropertySources();
		PropertySource<?> existing = sources.get(RANDOM_PROPERTY_SOURCE_NAME);
		if (existing != null) {
			logger.trace("RandomValuePropertySource already present");
			return;
		}
		RandomValuePropertySource randomSource = new RandomValuePropertySource(RANDOM_PROPERTY_SOURCE_NAME);
		if (sources.get(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME) != null) {
			sources.addAfter(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, randomSource);
		}
		else {
			sources.addLast(randomSource);
		}
		logger.trace("RandomValuePropertySource add to Environment");
	}
```
> 首先判断环境的属性源PropertySources中是否存在名称为"random"的属性源，如果存在不做任何处理直接返回，如果不存在，且存在"systemEnvironment"属性源的情况下，则将"random"属性源添加到其后，否则直接放在属性源集合最后。

3. 我们来看一下添加到属性源"random"对应的RandomPropertySource具备哪些功能：
> RandomPropertySource是一个属性源，对于任何以"random."开头的属性，它返回一个随机值。其中，请求的属性名中"random."前缀之后的部分被称为“未限定属性名”，此属性源返回：
* 当未限定属性名为int时，返回一个随机的Integer值，可以通过指定的范围进行限制。
* 当未限定属性名为long时，返回一个随机的Long值，可以通过指定的范围进行限制。
* 当未限定属性名为uuid时，返回一个随机的UUID值。

在yaml文件中的语法：
```yaml
item1: ${random.int}
item2: ${random.int(100,1000)}
item3: ${random.long}
item4: ${random.long(0L,1000L)}
item5: ${random.uuid}
```

RandomPropertySource源码
```java
public class RandomValuePropertySource extends PropertySource<Random> {

	/**
	 * Name of the random {@link PropertySource}.
	 */
	public static final String RANDOM_PROPERTY_SOURCE_NAME = "random";

	private static final String PREFIX = "random.";

	private static final Log logger = LogFactory.getLog(RandomValuePropertySource.class);

	public RandomValuePropertySource() {
		this(RANDOM_PROPERTY_SOURCE_NAME);
	}

	public RandomValuePropertySource(String name) {
		super(name, new Random());
	}

	@Override
	public Object getProperty(String name) {
		if (!name.startsWith(PREFIX)) {
			return null;
		}
		logger.trace(LogMessage.format("Generating random property for '%s'", name));
		return getRandomValue(name.substring(PREFIX.length()));
	}

	private Object getRandomValue(String type) {
		if (type.equals("int")) {
			return getSource().nextInt();
		}
		if (type.equals("long")) {
			return getSource().nextLong();
		}
		String range = getRange(type, "int");
		if (range != null) {
			return getNextIntInRange(Range.of(range, Integer::parseInt));
		}
		range = getRange(type, "long");
		if (range != null) {
			return getNextLongInRange(Range.of(range, Long::parseLong));
		}
		if (type.equals("uuid")) {
			return UUID.randomUUID().toString();
		}
		return getRandomBytes();
	}

	private String getRange(String type, String prefix) {
		if (type.startsWith(prefix)) {
			int startIndex = prefix.length() + 1;
			if (type.length() > startIndex) {
				return type.substring(startIndex, type.length() - 1);
			}
		}
		return null;
	}

	private int getNextIntInRange(Range<Integer> range) {
		OptionalInt first = getSource().ints(1, range.getMin(), range.getMax()).findFirst();
		assertPresent(first.isPresent(), range);
		return first.getAsInt();
	}

	private long getNextLongInRange(Range<Long> range) {
		OptionalLong first = getSource().longs(1, range.getMin(), range.getMax()).findFirst();
		assertPresent(first.isPresent(), range);
		return first.getAsLong();
	}

	private void assertPresent(boolean present, Range<?> range) {
		Assert.state(present, () -> "Could not get random number for range '" + range + "'");
	}

	private Object getRandomBytes() {
		byte[] bytes = new byte[32];
		getSource().nextBytes(bytes);
		return DigestUtils.md5DigestAsHex(bytes);
	}

	public static void addToEnvironment(ConfigurableEnvironment environment) {
		addToEnvironment(environment, logger);
	}

	static void addToEnvironment(ConfigurableEnvironment environment, Log logger) {
		MutablePropertySources sources = environment.getPropertySources();
		PropertySource<?> existing = sources.get(RANDOM_PROPERTY_SOURCE_NAME);
		if (existing != null) {
			logger.trace("RandomValuePropertySource already present");
			return;
		}
		RandomValuePropertySource randomSource = new RandomValuePropertySource(RANDOM_PROPERTY_SOURCE_NAME);
		if (sources.get(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME) != null) {
			sources.addAfter(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, randomSource);
		}
		else {
			sources.addLast(randomSource);
		}
		logger.trace("RandomValuePropertySource add to Environment");
	}

	static final class Range<T extends Number> {

		private final String value;

		private final T min;

		private final T max;

		private Range(String value, T min, T max) {
			this.value = value;
			this.min = min;
			this.max = max;
		}

		T getMin() {
			return this.min;
		}

		T getMax() {
			return this.max;
		}

		@Override
		public String toString() {
			return this.value;
		}

		static <T extends Number & Comparable<T>> Range<T> of(String value, Function<String, T> parse) {
			T zero = parse.apply("0");
			String[] tokens = StringUtils.commaDelimitedListToStringArray(value);
			T min = parse.apply(tokens[0]);
			if (tokens.length == 1) {
				Assert.isTrue(min.compareTo(zero) > 0, "Bound must be positive.");
				return new Range<>(value, zero, min);
			}
			T max = parse.apply(tokens[1]);
			Assert.isTrue(min.compareTo(max) < 0, "Lower bound must be less than upper bound.");
			return new Range<>(value, min, max);
		}

	}

}
```