---
title: '"自定义配置文件properties与Spring boot源码启动配置文件分析（二）"' 
categories: java 
tags: springboot 
date: 2018-10-21 22:22:29 
---
[上一篇文章](https://www.jianshu.com/p/b325f9ba75bd)说完了自定义配置文件properties的写法，并且介绍了采用classpath*通配模式下读取多个同名文件的内容方法。
下面就要进入到对springboot的默认配置文件application.properties的分析了。

## 二、springboot读取默认配置文件深入探究
### 1.springboot启动用运行
通过run里面跟踪下去，会执行到ConfigurableApplicationContext run(String... args)里面，在里面初始化sprignboot的上下文配置。它的listern（ConfigFileApplicationListener）会对它的执行进行监听。 
ConfigFileApplicationListener里面存在一些static变量，我们先来看一下他们,后面都会用到。
注意下面的 DEFAULT_SEARCH_LOCATIONS ，都是采用classpath:的模式，并没有带*
``` java
public class ConfigFileApplicationListener
        implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {

    private static final String DEFAULT_PROPERTIES = "defaultProperties";

    // Note the order is from least to most specific (last one wins)
    private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/";

    private static final String DEFAULT_NAMES = "application";

...
    /**
     * The "config location" property name.
     */
    public static final String CONFIG_LOCATION_PROPERTY = "spring.config.location";

    /**
     * The "config additional location" property name.
     */
    public static final String CONFIG_ADDITIONAL_LOCATION_PROPERTY = "spring.config.additional-location";
...
}
```
<!--more-->
调用到ConfigFileApplicationListener 里面的
addLoadedPropertySources(ConfigurableEnvironment environment,ResourceLoader resourceLoader)，该方法内部会初始化Loader对象，调用load()函数。
下面进入正文——我们来详细看一下load()函数
```java
public void load() {
            this.profiles = new LinkedList<>();
            this.processedProfiles = new LinkedList<>();
            this.activatedProfiles = false;
            this.loaded = new LinkedHashMap<>();
            initializeProfiles();
            while (!this.profiles.isEmpty()) {
                Profile profile = this.profiles.poll();
                if (profile != null && !profile.isDefaultProfile()) {
                    addProfileToEnvironment(profile.getName());
                }
                load(profile, this::getPositiveProfileFilter,
                        addToLoaded(MutablePropertySources::addLast, false));
                this.processedProfiles.add(profile);
            }
            load(null, this::getNegativeProfileFilter,
                    addToLoaded(MutablePropertySources::addFirst, true));
            addLoadedPropertySources();
        }
```
可以发现里面profiles.poll()一开始为空，所以会调用
 load(profile, this::getPositiveProfileFilter,
                        addToLoaded(MutablePropertySources::addLast, false));
### 2.真正读取的地方
首先来看一下load源码
```java 
private void load(Profile profile, DocumentFilterFactory filterFactory,
                DocumentConsumer consumer) {
            getSearchLocations().forEach((location) -> {
                boolean isFolder = location.endsWith("/");
                Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
                names.forEach(
                        (name) -> load(location, name, profile, filterFactory, consumer));
            });
        }
```



### 3. 先调用getSearchLocations()方法
接着相应的搜索地方调用查找位置，除非配置了spring.config.location
```java
private Set<String> getSearchLocations() {
            if (this.environment.containsProperty(CONFIG_LOCATION_PROPERTY)) {
                return getSearchLocations(CONFIG_LOCATION_PROPERTY);
            }
            Set<String> locations = getSearchLocations(
                    CONFIG_ADDITIONAL_LOCATION_PROPERTY);
            locations.addAll(
                    asResolvedSet(ConfigFileApplicationListener.this.searchLocations,
                            DEFAULT_SEARCH_LOCATIONS));
            return locations;
        }
```

#### 3.1继续进入其中的asResolvedSet()方法
这里面做了一个reverse操作
```java
    private Set<String> asResolvedSet(String value, String fallback) {
            List<String> list = Arrays.asList(StringUtils.trimArrayElements(
                    StringUtils.commaDelimitedListToStringArray((value != null)
                            ? this.environment.resolvePlaceholders(value) : fallback)));
            Collections.reverse(list);
            return new LinkedHashSet<>(list);
        }
```

### 4. 接着我们来看 getSearchNames()方法
操作都差不多，大同小异，先看下有没有设置别名，没有的话就采用默认的application名称
```java
private Set<String> getSearchNames() {
            if (this.environment.containsProperty(CONFIG_NAME_PROPERTY)) {
                String property = this.environment.getProperty(CONFIG_NAME_PROPERTY);
                return asResolvedSet(property, null);
            }
            return asResolvedSet(ConfigFileApplicationListener.this.names, DEFAULT_NAMES);
        }
```

### 5. 最后我们来看一下foreach里面的load方法
如果找得到name(即上面默认情况下的application，或者自定义名称)的情况下，就进行相应的配置文件的读取
```java
private void load(String location, String name, Profile profile,
                DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
            if (!StringUtils.hasText(name)) {
                for (PropertySourceLoader loader : this.propertySourceLoaders) {
                    if (canLoadFileExtension(loader, location)) {
                        load(loader, location, profile,
                                filterFactory.getDocumentFilter(profile), consumer);
                        return;
                    }
                }
            }
            Set<String> processed = new HashSet<>();
            for (PropertySourceLoader loader : this.propertySourceLoaders) {
                for (String fileExtension : loader.getFileExtensions()) {
                    if (processed.add(fileExtension)) {
                        loadForFileExtension(loader, location + name, "." + fileExtension,
                                profile, filterFactory, consumer);
                    }
                }
            }
        }
```

#### 5.1 继续进入第一个if的load()方法
别看这段代码长，实际上就是try里面的第一句有用，后面都是一些情况判断

```java
private void load(PropertySourceLoader loader, String location, Profile profile,
                DocumentFilter filter, DocumentConsumer consumer) {
            try {
                Resource resource = this.resourceLoader.getResource(location);
                if (resource == null || !resource.exists()) {
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Skipped missing config "
                                + getDescription(location, resource, profile));
                    }
                    return;
                }
                if (!StringUtils.hasText(
                        StringUtils.getFilenameExtension(resource.getFilename()))) {
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Skipped empty config extension "
                                + getDescription(location, resource, profile));
                    }
                    return;
                }
                String name = "applicationConfig: [" + location + "]";
                List<Document> documents = loadDocuments(loader, name, resource);
                if (CollectionUtils.isEmpty(documents)) {
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Skipped unloaded config "
                                + getDescription(location, resource, profile));
                    }
                    return;
                }
                List<Document> loaded = new ArrayList<>();
                for (Document document : documents) {
                    if (filter.match(document)) {
                        addActiveProfiles(document.getActiveProfiles());
                        addIncludedProfiles(document.getIncludeProfiles());
                        loaded.add(document);
                    }
                }
                Collections.reverse(loaded);
                if (!loaded.isEmpty()) {
                    loaded.forEach((document) -> consumer.accept(profile, document));
                    if (this.logger.isDebugEnabled()) {
                        this.logger.debug("Loaded config file "
                                + getDescription(location, resource, profile));
                    }
                }
            }
            catch (Exception ex) {
                throw new IllegalStateException("Failed to load property "
                        + "source from location '" + location + "'", ex);
            }
        }
```
#### 5.2 与自定义配置文件相连
自此，就和前面的给串联起来了。getResource因为不是classpath*模式，所以会走到上面代码的最后一个else
```java
 // a single resource with the given name
    return new Resource[] {getResourceLoader().getResource(locationPattern)};
```

而getResourceLoader().getResource(locationPattern)最后返回的是一个Resource对象，所以对于Resource[]中只会有一个对象存在。

#### 5.3我们在进一步查看它的实现类
```java
public interface ResourceLoader {

    /** Pseudo URL prefix for loading from the class path: "classpath:" */
    String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;


    /**
     * Return a Resource handle for the specified resource location.
     * <p>The handle should always be a reusable resource descriptor,
     * allowing for multiple {@link Resource#getInputStream()} calls.
     * <p><ul>
     * <li>Must support fully qualified URLs, e.g. "file:C:/test.dat".
     * <li>Must support classpath pseudo-URLs, e.g. "classpath:test.dat".
     * <li>Should support relative file paths, e.g. "WEB-INF/test.dat".
     * (This will be implementation-specific, typically provided by an
     * ApplicationContext implementation.)
     * </ul>
     * <p>Note that a Resource handle does not imply an existing resource;
     * you need to invoke {@link Resource#exists} to check for existence.
     * @param location the resource location
     * @return a corresponding Resource handle (never {@code null})
     * @see #CLASSPATH_URL_PREFIX
     * @see Resource#exists()
     * @see Resource#getInputStream()
     */
    Resource getResource(String location);
```
进一步看它的实现类，如果是for循环里面找到一个值，就会返回
```java
@Override
    public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");

        for (ProtocolResolver protocolResolver : this.protocolResolvers) {
            Resource resource = protocolResolver.resolve(location, this);
            if (resource != null) {
                return resource;
            }
        }

        if (location.startsWith("/")) {
            return getResourceByPath(location);
        }
        else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
        }
        else {
            try {
                // Try to parse the location as a URL...
                URL url = new URL(location);
                return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            }
            catch (MalformedURLException ex) {
                // No URL -> resolve as resource path.
                return getResourceByPath(location);
            }
        }
    }
```

```java
public ClassPathResource(String path, @Nullable ClassLoader classLoader) {
        Assert.notNull(path, "Path must not be null");
        String pathToUse = StringUtils.cleanPath(path);
        if (pathToUse.startsWith("/")) {
            pathToUse = pathToUse.substring(1);
        }
        this.path = pathToUse;
        this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
    }
```

### 6. 当读取完后会回到 1 下面的执行最后的加载参数
> addLoadedPropertySources();

**总结**:对于classpath*模式会全局查找多处结果合并，而classpath只要找到一个就停止了。所以采用默认模块下面的配置文件，application.properties，在单项目多模块下，记得要在web模块（即启动模块）下放置***一个**就好了。这样一方面便于管理，一方面也不会出现重复覆盖的问题。
