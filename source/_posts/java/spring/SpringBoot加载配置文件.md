---
title: SpringBoot加载配置文件
categories: 
    - Java
date: 2018-10-22 22:40:23
tags:
    - Spring
    - SpringBoot
cover: https://static.jiangliuhong.top/blogimg/spring/springboot.jpg
---

# SpringBoot加载配置文件

> 读过SpringBoot源码的同学应该都知道它会在启动过程中根据spring.factories加载监听器，而其中有一个名叫`ConfigFileApplicationListener`的监听器，它的作用为加载配置信息，即application.xml、application.yml。

## 常量值说明

在`ConfigFileApplicationListener`定义了一批常量，他们主要为加载配置文件服务，现在就总体地看看这些常量吧。

| 名称                      | 类型   | 默认值                                                | 说明                     |
| ------------------------- | ------ | ----------------------------------------------------- | ------------------------ |
| DEFAULT_SEARCH_LOCATIONS  | String | classpath:/,classpath:/config/,file:./,file:./config/ | 配置文件加载顺序         |
| DEFAULT_NAMES             | String | application                                           | 默认配置文件名称         |
| ACTIVE_PROFILES_PROPERTY  | String | spring.profiles.active                                | “活动配置文件”属性名称   |
| INCLUDE_PROFILES_PROPERTY | String | spring.profiles.include                               | “包含配置文件”属性名称。 |
| CONFIG_NAME_PROPERTY      | String | spring.config.name                                    | “配置名称”属性名称。     |
| CONFIG_LOCATION_PROPERTY  | String | spring.config.location                                | “配置位置”属性名称。     |
| DEFAULT_ORDER             | int    | Integer.MIN_VALUE+10                                  | 处理器的默认顺序。       |

## 配置文件加载

### 监听器入口

阅读SpringBoot启动源码时，我们都知道监听器真实生效的方法是`onApplicationEvent`，`ConfigFileApplicationListener`中的`onApplicationEvent`的源码如下：

```java
public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ApplicationEnvironmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent(
					(ApplicationEnvironmentPreparedEvent) event);
		}
		if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent(event);
		}
	}
```

通过`onApplicationEnvironmentPreparedEvent `=>`postProcessEnvironment`方法

```java
public void postProcessEnvironment(ConfigurableEnvironment environment,
			SpringApplication application) {
    	//将配置文件属性源添加到指定的环境
		addPropertySources(environment, application.getResourceLoader());
    	//设置spring.beaninfo.ignore变量
		configureIgnoreBeanInfo(environment);
    	//把环境绑定到SpringApplication。
		bindToSpringApplication(environment, application);
	}
```

addPropertySources源码如下：

```java
protected void addPropertySources(ConfigurableEnvironment environment,
			ResourceLoader resourceLoader) {
		RandomValuePropertySource.addToEnvironment(environment);
		new Loader(environment, resourceLoader).load();
	}
```

通过上述代码，可以看出配置文件的接在就在`load`方法中。

### 文件加载

load方法源码：

```java
public void load() {
			this.propertiesLoader = new PropertySourcesLoader();
			this.activatedProfiles = false;
			this.profiles = Collections.asLifoQueue(new LinkedList<Profile>());
			this.processedProfiles = new LinkedList<Profile>();

			// Pre-existing active profiles set via Environment.setActiveProfiles()
			// are additional profiles and config files are allowed to add more if
			// they want to, so don't call addActiveProfiles() here.
			Set<Profile> initialActiveProfiles = initializeActiveProfiles();
			this.profiles.addAll(getUnprocessedActiveProfiles(initialActiveProfiles));
			if (this.profiles.isEmpty()) {
				for (String defaultProfileName : this.environment.getDefaultProfiles()) {
					Profile defaultProfile = new Profile(defaultProfileName, true);
					if (!this.profiles.contains(defaultProfile)) {
						this.profiles.add(defaultProfile);
					}
				}
			}

			// The default profile for these purposes is represented as null. We add it
			// last so that it is first out of the queue (active profiles will then
			// override any settings in the defaults when the list is reversed later).
    		//翻译：下面一行代码的目的是将默认配置文件被表示为null。将null添加到最后，以便它首先从队列中取出（主动概要文件将在稍后颠倒列表时覆盖缺省情况下的任何设置）。
			this.profiles.add(null);

			while (!this.profiles.isEmpty()) {
				Profile profile = this.profiles.poll();
                 //查找配置文件
				for (String location : getSearchLocations()) {
					if (!location.endsWith("/")) {
						// location is a filename already, so don't search for more
						// filenames
						load(location, null, profile);
					}
					else {
						for (String name : getSearchNames()) {
							load(location, name, profile);
						}
					}
				}
				this.processedProfiles.add(profile);
			}

			addConfigurationProperties(this.propertiesLoader.getPropertySources());
		}
```

#### 查找配置文件路径

值得注意的是`getSearchLocations`方法，其源码如下：

```
private Set<String> getSearchLocations() {
			Set<String> locations = new LinkedHashSet<String>();
			// User-configured settings take precedence, so we do them first
			// 判断当前环境中是否有spring.config.location属性，如果有的话，则加载spring.config.location指定的配置文件
			if (this.environment.containsProperty(CONFIG_LOCATION_PROPERTY)) {
				for (String path : asResolvedSet(
						this.environment.getProperty(CONFIG_LOCATION_PROPERTY), null)) {
					if (!path.contains("$")) {
						path = StringUtils.cleanPath(path);
						if (!ResourceUtils.isUrl(path)) {
							path = ResourceUtils.FILE_URL_PREFIX + path;
						}
					}
					locations.add(path);
				}
			}
			//添加默认的配置文件，按照类中定义的顺序加载文件：
			//其顺序为：classpath:/,classpath:/config/,file:./,file:./config/
			locations.addAll(
					asResolvedSet(ConfigFileApplicationListener.this.searchLocations,
							DEFAULT_SEARCH_LOCATIONS));
			return locations;
		}
```

DEFAULT_SEARCH_LOCATIONS为文件默认加载顺序，其值为classpath:/,classpath:/config/,file:./,file:./config/，然后springboot会按照这几个位置的由后到前的顺序去加载。

通过查看`asResolvedSet`的源码：

```java
private Set<String> asResolvedSet(String value, String fallback) {
			List<String> list = Arrays.asList(StringUtils.trimArrayElements(
					StringUtils.commaDelimitedListToStringArray(value != null
							? this.environment.resolvePlaceholders(value) : fallback)));
			Collections.reverse(list);
			return new LinkedHashSet<String>(list);
		}
```

不难看出，这里对list进行了`Collections.reverse`(反转处理)。

即配置文件真正的加载顺序为：

- file:./config/
- file:./
- classpath:/config/
- classpath:/
- spring.config.location

#### 正式加载配置文件

`load(String location, String name, Profile profile)`方法源码：

```java
private void load(String location, String name, Profile profile) {
			String group = "profile=" + (profile == null ? "" : profile);
			if (!StringUtils.hasText(name)) {
				// Try to load directly from the location
				loadIntoGroup(group, location, profile);
			}
			else {
				// Search for a file with the given name
				for (String ext : this.propertiesLoader.getAllFileExtensions()) {
					if (profile != null) {
						// Try the profile-specific file
						loadIntoGroup(group, location + name + "-" + profile + "." + ext,
								null);
						for (Profile processedProfile : this.processedProfiles) {
							if (processedProfile != null) {
								loadIntoGroup(group, location + name + "-"
										+ processedProfile + "." + ext, profile);
							}
						}
						// Sometimes people put "spring.profiles: dev" in
						// application-dev.yml (gh-340). Arguably we should try and error
						// out on that, but we can be kind and load it anyway.
						loadIntoGroup(group, location + name + "-" + profile + "." + ext,
								profile);
					}
					// Also try the profile-specific section (if any) of the normal file
					loadIntoGroup(group, location + name + "." + ext, profile);
				}
			}
		}

```

在Load方法中，首先会通过`getAllFileExtensions`方法去组装所有可加载文件的扩展名，然后在通过`loadIntoGroup`方法加载配置文件，而我们跟读到`loadIntoGroup`中会发现其只执行了 `doLoadIntoGroup`方法， `doLoadIntoGroup`源码如下：

```java
private PropertySource<?> doLoadIntoGroup(String identifier, String location,
				Profile profile) throws IOException {
			Resource resource = this.resourceLoader.getResource(location);
			PropertySource<?> propertySource = null;
			StringBuilder msg = new StringBuilder();
			if (resource != null && resource.exists()) {
				String name = "applicationConfig: [" + location + "]";
				String group = "applicationConfig: [" + identifier + "]";
				propertySource = this.propertiesLoader.load(resource, group, name,
						(profile == null ? null : profile.getName()));
				if (propertySource != null) {
					msg.append("Loaded ");
					handleProfileProperties(propertySource);
				}
				else {
					msg.append("Skipped (empty) ");
				}
			}
			else {
				msg.append("Skipped ");
			}
			msg.append("config file ");
			msg.append(getResourceDescription(location, resource));
			if (profile != null) {
				msg.append(" for profile ").append(profile);
			}
			if (resource == null || !resource.exists()) {
				msg.append(" resource not found");
				this.logger.trace(msg);
			}
			else {
				this.logger.debug(msg);
			}
			return propertySource;
		}
```

上诉源码中有一段核心的方法

```java
propertySource = this.propertiesLoader.load(resource, group, name,
						(profile == null ? null : profile.getName()));
				if (propertySource != null) {
					msg.append("Loaded ");
					handleProfileProperties(propertySource);
				}
```

如果propertySource存在，则调用handleProfileProperties方法。

handleProfileProperties源码：

```java
private void handleProfileProperties(PropertySource<?> propertySource) {
			SpringProfiles springProfiles = bindSpringProfiles(propertySource);
			maybeActivateProfiles(springProfiles.getActiveProfiles());
			addProfiles(springProfiles.getIncludeProfiles());
		}
```

bindSpringProfiles源码：

```java
private SpringProfiles bindSpringProfiles(PropertySource<?> propertySource) {
			MutablePropertySources propertySources = new MutablePropertySources();
			propertySources.addFirst(propertySource);
			return bindSpringProfiles(propertySources);
		}
```

bindSpringProfiles源码：

```java
private SpringProfiles bindSpringProfiles(PropertySources propertySources) {
			SpringProfiles springProfiles = new SpringProfiles();
			RelaxedDataBinder dataBinder = new RelaxedDataBinder(springProfiles,
					"spring.profiles");
			dataBinder.bind(new PropertySourcesPropertyValues(propertySources, false));
			springProfiles.setActive(resolvePlaceholders(springProfiles.getActive()));
			springProfiles.setInclude(resolvePlaceholders(springProfiles.getInclude()));
			return springProfiles;
		}
```

通过这样一步一步的跟读，我们能发现配置文件的装载主要是通过`new PropertySourcesPropertyValues`来完成。

查看`PropertySourcesPropertyValues`的构造方法可以发现，它在装载值时调用了这样一个方法：

```java
private void processPropertySource(PropertySource<?> source,
			PropertySourcesPropertyResolver resolver) {
		if (source instanceof CompositePropertySource) {
            //加载复杂的属性
			processCompositePropertySource((CompositePropertySource) source, resolver);
		}
		else if (source instanceof EnumerablePropertySource) {
            //加载枚举类型
			processEnumerablePropertySource((EnumerablePropertySource<?>) source,
					resolver, this.includes);
		}
		else {
            //加载非枚举类型，即简单类型
			processNonEnumerablePropertySource(source, resolver);
		}
	}
```

#### 配置文件键值对加载方法

> 此处以`yml`文件加载为例

配置文件键值对加载方法，org.springframework.beans.factory.config.YamlProcessor#process(MatchCallback callback, Yaml yaml, Resource resource) 

在该方法中通过` yaml.loadAll(reader)`去加载文件的属性，然后在下方`process(asMap(object), callback)`通过`callback`进行键值对组装。

关键代码如下：

```java
try {
    for (Object object : yaml.loadAll(reader)) {
        if (object != null && process(asMap(object), callback)) {
            count++;
            if (this.resolutionMethod == ResolutionMethod.FIRST_FOUND) {
                break;
            }
        }
    }
    if (logger.isDebugEnabled()) {
        logger.debug("Loaded " + count + " document" + (count > 1 ? "s" : "") +
                     " from YAML resource: " + resource);
    }
}
finally {
    reader.close();
}
```

该代码中的callback为org.springframework.boot.env.YamlPropertySourceLoader#process()中new的一个，源码如下：

```java
public Map<String, Object> process() {
			final Map<String, Object> result = new LinkedHashMap<String, Object>();
			process(new MatchCallback() {
				@Override
				public void process(Properties properties, Map<String, Object> map) {
					result.putAll(getFlattenedMap(map));
				}
			});
			return result;
		}
```

## 总结

配置文件的加载主要分为两个步骤：

1. 查找配置文件
2. 加载配置文件中的值

其中查找配置文件加载顺序为：

- file:./config/
- file:./
- classpath:/config/
- classpath:/
- spring.config.location

spring.config.location为在启动SpringBoot时，为其指定的配置文件路径，

设置方式为：

```
java -jar demo.jar --Dspring.config.location=application.yml
```







