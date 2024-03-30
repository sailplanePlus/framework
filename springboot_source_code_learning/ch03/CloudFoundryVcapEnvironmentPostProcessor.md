# CloundFoundryVcapEnvironmentPostProcessor 添加"vcap"属性源

> 这是一个EnvironmentPostProcessor，它知道如何在现有环境中找到VCAP（也称为Cloud Foundry）元数据。它解析出VCAP_APPLICATION和VCAP_SERVICES元数据，并将其转换成易于Environment用户消费的形式。如果应用程序在Cloud Foundry中运行，则这两个元数据项都是以OS环境变量编码的JSON对象。VCAP_APPLICATION是一个基本信息的浅哈希（名称、实例ID、实例索引等），而VCAP_SERVICES是一个列表哈希，其中键是服务标签，值是服务实例元数据的哈希列表。例如：

```css
VCAP_APPLICATION: {"instance_id":"2ce0ac627a6c8e47e936d829a3a47b5b","instance_index":0,
    "version":"0138c4a6-2a73-416b-aca0-572c09f7ca53","name":"foo",
    "uris":["foo.cfapps.io"], ...}
VCAP_SERVICES: {"rds-mysql-1.0":[{"name":"mysql","label":"rds-mysql-1.0","plan":"10mb",
    "credentials":{"name":"d04fb13d27d964c62b267bbba1cffb9da","hostname":"mysql-service-public.clqg2e2w3ecf.us-east-1.rds.amazonaws.com",
    "host":"mysql-service-public.clqg2e2w3ecf.us-east-1.rds.amazonaws.com","port":3306,"user":"urpRuqTf8Cpe6",
    "username":"urpRuqTf8Cpe6","password":"pxLsGVpsC9A5S"}
  }]}
```

> 这些对象被展开成属性。VCAP_APPLICATION对象直接转换为vcap.application.*，方式相当明显，而VCAP_SERVICES对象则被展开，使其成为一个哈希，其中键等于服务实例名称（例如上面的示例中的"mysql"），值等于该实例的属性，然后以相同的方式展开。例如：

```makefile
vcap.application.instance_id: 2ce0ac627a6c8e47e936d829a3a47b5b
vcap.application.version: 0138c4a6-2a73-416b-aca0-572c09f7ca53
vcap.application.name: foo
vcap.application.uris[0]: foo.cfapps.io

vcap.services.mysql.name: mysql
vcap.services.mysql.label: rds-mysql-1.0
vcap.services.mysql.credentials.name: d04fb13d27d964c62b267bbba1cffb9da
vcap.services.mysql.credentials.port: 3306
vcap.services.mysql.credentials.host: mysql-service-public.clqg2e2w3ecf.us-east-1.rds.amazonaws.com
vcap.services.mysql.credentials.username: urpRuqTf8Cpe6
vcap.services.mysql.credentials.password: pxLsGVpsC9A5S
...
```
请注意，此初始化程序主要用于信息用途（应用程序和实例ID特别有用）。对于服务绑定，您可能会发现Spring Cloud更方便，更能抵御Cloud Foundry潜在变化的影响。

```java
	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		if (CloudPlatform.CLOUD_FOUNDRY.isActive(environment)) {
			Properties properties = new Properties();
			JsonParser jsonParser = JsonParserFactory.getJsonParser();
			addWithPrefix(properties, getPropertiesFromApplication(environment, jsonParser), "vcap.application.");
			addWithPrefix(properties, getPropertiesFromServices(environment, jsonParser), "vcap.services.");
			MutablePropertySources propertySources = environment.getPropertySources();
			if (propertySources.contains(CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME)) {
				propertySources.addAfter(CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME,
						new PropertiesPropertySource("vcap", properties));
			}
			else {
				propertySources.addFirst(new PropertiesPropertySource("vcap", properties));
			}
		}
	}
```