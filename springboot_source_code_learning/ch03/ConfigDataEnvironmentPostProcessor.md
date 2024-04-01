# ConfigDataEnvironmentPostProcessor 加载配置文件数据(ConfigData)到spring的Environment中

> ConfigDataEnvironmentPostProcessor是SpringBoot2.4引入的一个环境后处理器，用于处理配置数据。它的主要作用是在SpringBoot应用程序启动时，对配置数据进行加载、解析和处理，以便应用程序能够正确地获取和使用配置数据。

一. 执行postProcessorEnvironment()方法
```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		postProcessEnvironment(environment, application.getResourceLoader(), application.getAdditionalProfiles());
	}
	
	void postProcessEnvironment(ConfigurableEnvironment environment, ResourceLoader resourceLoader,
			Collection<String> additionalProfiles) {
		try {
			this.logger.trace("Post-processing environment to add config data");
			resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader();
			getConfigDataEnvironment(environment, resourceLoader, additionalProfiles).processAndApply();
		}
		catch (UseLegacyConfigProcessingException ex) {
			this.logger.debug(LogMessage.format("Switching to legacy config file processing [%s]",
					ex.getConfigurationProperty()));
			configureAdditionalProfiles(environment, additionalProfiles);
			postProcessUsingLegacyApplicationListener(environment, resourceLoader);
		}
	}
```

二. 创建ConfigDataEnvironment
> 这是一个ConfigurableEnvironment的包装器，可用于导入和应用ConfigData。
> 通过包装来自Spring环境的属性源并添加初始位置，配置初始的ConfigDataEnvironmentCOntributors集合。
> 初始位置可以通过LOCATION_PROPERTY(spring.config.location)，"spring.config.additonal-location"和"spring.config.import"属性进行影响。如果没有显式设置属性，则将使用DEFAULT_SEARCH_LOCATIONS。

我们来看一下创建ConfigDataEnvironment 做了哪些事情：

1. 执行静态代码块，初始化配置文件默认搜索位置数组DEFAULT_SEARCH_LOCATIONS
```java
	static {
		List<ConfigDataLocation> locations = new ArrayList<>();
		locations.add(ConfigDataLocation.of("optional:classpath:/;optional:classpath:/config/"));
		locations.add(ConfigDataLocation.of("optional:file:./;optional:file:./config/;optional:file:./config/*/"));
		DEFAULT_SEARCH_LOCATIONS = locations.toArray(new ConfigDataLocation[0]);
	}
```

由此为我们得出springboot的默认加载路径有5个,会从这5个路径下加载applicaiton.properties或application.yml，分别是
* classpath:/
* classpath:/config/
* file:./      (项目根路径)
* file:./config/       (项目根路径/config/)
* file:./config/*/     (项目根路径/config/ */) 

5个默认的加载路径的优先级为：
* 项目根路径下的config/*/ > 项目根路径下的config > 项目根路径 > classpath:/config > classpath:/
* 同级下，application.properties文件优先级大于application.yml

2. 执行构造函数方法
```java
	ConfigDataEnvironment(DeferredLogFactory logFactory, ConfigurableBootstrapContext bootstrapContext,
			ConfigurableEnvironment environment, ResourceLoader resourceLoader, Collection<String> additionalProfiles,
			ConfigDataEnvironmentUpdateListener environmentUpdateListener) {
		Binder binder = Binder.get(environment);
		UseLegacyConfigProcessingException.throwIfRequested(binder);
		this.logFactory = logFactory;
		this.logger = logFactory.getLog(getClass());
		this.notFoundAction = binder.bind(ON_NOT_FOUND_PROPERTY, ConfigDataNotFoundAction.class)
				.orElse(ConfigDataNotFoundAction.FAIL);
		this.bootstrapContext = bootstrapContext;
		this.environment = environment;
		this.resolvers = createConfigDataLocationResolvers(logFactory, bootstrapContext, binder, resourceLoader);
		this.additionalProfiles = additionalProfiles;
		this.environmentUpdateListener = (environmentUpdateListener != null) ? environmentUpdateListener
				: ConfigDataEnvironmentUpdateListener.NONE;
		this.loaders = new ConfigDataLoaders(logFactory, bootstrapContext, resourceLoader.getClassLoader());
		this.contributors = createContributors(binder);
	}
```
3. 加载spring.factories中注册的ConfigDataLocationResolver的完全限定名，并实例化它们
```
# ConfigData Location Resolvers
org.springframework.boot.context.config.ConfigDataLocationResolver=\
org.springframework.boot.context.config.ConfigTreeConfigDataLocationResolver,\
org.springframework.boot.context.config.StandardConfigDataLocationResolver
```
```java
	protected ConfigDataLocationResolvers createConfigDataLocationResolvers(DeferredLogFactory logFactory,
			ConfigurableBootstrapContext bootstrapContext, Binder binder, ResourceLoader resourceLoader) {
		return new ConfigDataLocationResolvers(logFactory, bootstrapContext, binder, resourceLoader);
	}
	
	ConfigDataLocationResolvers(DeferredLogFactory logFactory, ConfigurableBootstrapContext bootstrapContext,
			Binder binder, ResourceLoader resourceLoader) {
		this(logFactory, bootstrapContext, binder, resourceLoader, SpringFactoriesLoader
				.loadFactoryNames(ConfigDataLocationResolver.class, resourceLoader.getClassLoader()));
	}
```
4. 我们来看StandardConfigDataLocationResolver的实例化过程
```java
	public StandardConfigDataLocationResolver(Log logger, Binder binder, ResourceLoader resourceLoader) {
		this.logger = logger;
		this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
				getClass().getClassLoader());
		this.configNames = getConfigNames(binder);
		this.resourceLoader = new LocationResourceLoader(resourceLoader);
	}
```
我们发现在构造方法内，又去加载了spring.factories中的PropertySourceLoader.class属性源加载器，并实例化了它们。
```
# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
```
5. 初始化ConfigDataLoaders，加载spring.factories中的ConfigDataLoarer完全限定名，并实例化它们。
```java
	ConfigDataLoaders(DeferredLogFactory logFactory, ConfigurableBootstrapContext bootstrapContext,
			ClassLoader classLoader) {
		this(logFactory, bootstrapContext, classLoader,
				SpringFactoriesLoader.loadFactoryNames(ConfigDataLoader.class, classLoader));
	}
```
```
# ConfigData Loaders
org.springframework.boot.context.config.ConfigDataLoader=\
org.springframework.boot.context.config.ConfigTreeConfigDataLoader,\
org.springframework.boot.context.config.StandardConfigDataLoader
```
6. 处理所有的contributions 并将任何新倒入的属性源应用到环境中
```java
	void processAndApply() {
		ConfigDataImporter importer = new ConfigDataImporter(this.logFactory, this.notFoundAction, this.resolvers,
				this.loaders);
		registerBootstrapBinder(this.contributors, null, DENY_INACTIVE_BINDING);
		ConfigDataEnvironmentContributors contributors = processInitial(this.contributors, importer);
		ConfigDataActivationContext activationContext = createActivationContext(
				contributors.getBinder(null, BinderOption.FAIL_ON_BIND_TO_INACTIVE_SOURCE));
		contributors = processWithoutProfiles(contributors, importer, activationContext);
		activationContext = withProfiles(contributors, activationContext);
		contributors = processWithProfiles(contributors, importer, activationContext);
		applyToEnvironment(contributors, activationContext, importer.getLoadedLocations(),
				importer.getOptionalLocations());
	}
```

```java
	private ConfigDataEnvironmentContributors processInitial(ConfigDataEnvironmentContributors contributors,
			ConfigDataImporter importer) {
		this.logger.trace("Processing initial config data environment contributors without activation context");
		contributors = contributors.withProcessedImports(importer, null);
		registerBootstrapBinder(contributors, null, DENY_INACTIVE_BINDING);
		return contributors;
	}
```
```java
	ConfigDataEnvironmentContributors withProcessedImports(ConfigDataImporter importer,
			ConfigDataActivationContext activationContext) {
		ImportPhase importPhase = ImportPhase.get(activationContext);
		this.logger.trace(LogMessage.format("Processing imports for phase %s. %s", importPhase,
				(activationContext != null) ? activationContext : "no activation context"));
		ConfigDataEnvironmentContributors result = this;
		int processed = 0;
		while (true) {
			ConfigDataEnvironmentContributor contributor = getNextToProcess(result, activationContext, importPhase);
			if (contributor == null) {
				this.logger.trace(LogMessage.format("Processed imports for of %d contributors", processed));
				return result;
			}
			if (contributor.getKind() == Kind.UNBOUND_IMPORT) {
				ConfigDataEnvironmentContributor bound = contributor.withBoundProperties(result, activationContext);
				result = new ConfigDataEnvironmentContributors(this.logger, this.bootstrapContext,
						result.getRoot().withReplacement(contributor, bound));
				continue;
			}
			ConfigDataLocationResolverContext locationResolverContext = new ContributorConfigDataLocationResolverContext(
					result, contributor, activationContext);
			ConfigDataLoaderContext loaderContext = new ContributorDataLoaderContext(this);
			List<ConfigDataLocation> imports = contributor.getImports();
			this.logger.trace(LogMessage.format("Processing imports %s", imports));
			Map<ConfigDataResolutionResult, ConfigData> imported = importer.resolveAndLoad(activationContext,
					locationResolverContext, loaderContext, imports);
			this.logger.trace(LogMessage.of(() -> getImportedMessage(imported.keySet())));
			ConfigDataEnvironmentContributor contributorAndChildren = contributor.withChildren(importPhase,
					asContributors(imported));
			result = new ConfigDataEnvironmentContributors(this.logger, this.bootstrapContext,
					result.getRoot().withReplacement(contributor, contributorAndChildren));
			processed++;
		}
	}
```
**解析并加载给定的位置列表，过滤掉已经加载过的位置**
```java
	Map<ConfigDataResolutionResult, ConfigData> resolveAndLoad(ConfigDataActivationContext activationContext,
			ConfigDataLocationResolverContext locationResolverContext, ConfigDataLoaderContext loaderContext,
			List<ConfigDataLocation> locations) {
		try {
			Profiles profiles = (activationContext != null) ? activationContext.getProfiles() : null;
			//YamlPropertySourceLoader、PropertiesPropertySourceLoader getFileExtensions方法
			//推断出可以加载DEFAULT_SEARCH_LOCATIONS路径下的哪些配置文件，最终筛选出实际存在的配置文件
			List<ConfigDataResolutionResult> resolved = resolve(locationResolverContext, profiles, locations);
			//使用合适的ConfigDataLoader解析配置文件，得到ConfigData
			return load(loaderContext, resolved);
		}
		catch (IOException ex) {
			throw new IllegalStateException("IO error on loading imports from " + locations, ex);
		}
	}
```
 使用一个合适的ConfigDataLoader(StandardConfigDataLoader)加载ConfigData
 ```java
 	<R extends ConfigDataResource> ConfigData load(ConfigDataLoaderContext context, R resource) throws IOException {
      //获取合适的ConfigDataLoader
		ConfigDataLoader<R> loader = getLoader(context, resource);
		this.logger.trace(LogMessage.of(() -> "Loading " + resource + " using loader " + loader.getClass().getName()));
		return loader.load(context, resource);
	}
```
加载给定资源，根据配置文件类型，使用对应的属性资源加载器进行加载，获取属性资源PropertySource
```java
@Override
	public ConfigData load(ConfigDataLoaderContext context, StandardConfigDataResource resource)
			throws IOException, ConfigDataNotFoundException {
		if (resource.isEmptyDirectory()) {
			return ConfigData.EMPTY;
		}
		ConfigDataResourceNotFoundException.throwIfDoesNotExist(resource, resource.getResource());
		StandardConfigDataReference reference = resource.getReference();
		Resource originTrackedResource = OriginTrackedResource.of(resource.getResource(),
				Origin.from(reference.getConfigDataLocation()));
		String name = String.format("Config resource '%s' via location '%s'", resource,
				reference.getConfigDataLocation());
		List<PropertySource<?>> propertySources = reference.getPropertySourceLoader().load(name, originTrackedResource);
		PropertySourceOptions options = (resource.getProfile() != null) ? PROFILE_SPECIFIC : NON_PROFILE_SPECIFIC;
		return new ConfigData(propertySources, options);
	}
```
org.springframework.boot.env.YamlPropertySourceLoader#load
```java
	@Override
	public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
		if (!ClassUtils.isPresent("org.yaml.snakeyaml.Yaml", getClass().getClassLoader())) {
			throw new IllegalStateException(
					"Attempted to load " + name + " but snakeyaml was not found on the classpath");
		}
		List<Map<String, Object>> loaded = new OriginTrackedYamlLoader(resource).load();
		if (loaded.isEmpty()) {
			return Collections.emptyList();
		}
		List<PropertySource<?>> propertySources = new ArrayList<>(loaded.size());
		for (int i = 0; i < loaded.size(); i++) {
			String documentNumber = (loaded.size() != 1) ? " (document #" + i + ")" : "";
			propertySources.add(new OriginTrackedMapPropertySource(name + documentNumber,
					Collections.unmodifiableMap(loaded.get(i)), true));
		}
		return propertySources;
	}
```
org.springframework.boot.env.PropertiesPropertySourceLoader#load
```java
	@Override
	public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
		List<Map<String, ?>> properties = loadProperties(resource);
		if (properties.isEmpty()) {
			return Collections.emptyList();
		}
		List<PropertySource<?>> propertySources = new ArrayList<>(properties.size());
		for (int i = 0; i < properties.size(); i++) {
			String documentNumber = (properties.size() != 1) ? " (document #" + i + ")" : "";
			propertySources.add(new OriginTrackedMapPropertySource(name + documentNumber,
					Collections.unmodifiableMap(properties.get(i)), true));
		}
		return propertySources;
	}
```
最终解析配置文件,将数据源OriginTrackedMapPropertySource添加到环境PropertySource中
```java
	private void applyToEnvironment(ConfigDataEnvironmentContributors contributors,
			ConfigDataActivationContext activationContext, Set<ConfigDataLocation> loadedLocations,
			Set<ConfigDataLocation> optionalLocations) {
		checkForInvalidProperties(contributors);
		checkMandatoryLocations(contributors, activationContext, loadedLocations, optionalLocations);
		MutablePropertySources propertySources = this.environment.getPropertySources();
		applyContributor(contributors, activationContext, propertySources);
		DefaultPropertiesPropertySource.moveToEnd(propertySources);
		Profiles profiles = activationContext.getProfiles();
		this.logger.trace(LogMessage.format("Setting default profiles: %s", profiles.getDefault()));
		this.environment.setDefaultProfiles(StringUtils.toStringArray(profiles.getDefault()));
		this.logger.trace(LogMessage.format("Setting active profiles: %s", profiles.getActive()));
		this.environment.setActiveProfiles(StringUtils.toStringArray(profiles.getActive()));
		this.environmentUpdateListener.onSetProfiles(profiles);
	}
```