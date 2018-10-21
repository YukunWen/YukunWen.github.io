---
title: '"自定义配置文件properties与Spring boot源码启动配置文件分析（一）"' 
categories: java 
tags: springboot 
date: 2018-10-20 21:57:29 
---
# 引言
springboot项目默认启动的是application.properties文件。在实际的开发中，一个项目中会含有多个模块，每个模块下面可能都含有一个或多个默认的配置信息。另外，有的工程中可能会需要引入额外的自定义配置文件。由此，引出三个问题：

**1. 自定义配置文件如何书写？**
**2. 默认的配置文件启动的顺序与读取？**
**3. 自定义配置文件与默认配置文件发生冲突的时候谁的优先级更高？**

下面我们将仔细的分析三个问题的实现与原理
## 一、自定义配置文件
### 1.1自定义配置文件书写
先上例子，整个项目的架构是parent下面多个模块，每个模块里面可能会有一些自定的参数，又不想集中全部写在application.properties中，这样会默认配置会显得繁琐，难以查找与修改。所以我们在不同的架构下会创建一个自定义的配置文件，名为fantuan.properties。
<!--more-->
![图1. 项目结构图](https://upload-images.jianshu.io/upload_images/14043408-13d2835e1a38846d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于这种额外自定义的配置，肯定要求多个模块下面都要生效。于是，源码要采用classpath*模式，具体代码如下：
```java
@Configuration
public class FantuanPropertiesConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer(
            ApplicationContext applicationContext) throws IOException {
        Resource[] resources = applicationContext.getResources("classpath*:fantuan.properties");
        PropertyPlaceholder placeholder = new PropertyPlaceholder();
        ArrayUtils.reverse(resources);
        placeholder.setResources(resources);
        return placeholder;
    }
}
```

```java
public class PropertyPlaceholder extends PropertySourcesPlaceholderConfigurer {

    @Setter
    private Resource[] resources;

    @Override
    protected void loadProperties(Properties props) throws IOException {
        super.loadProperties(props);
        for (Resource resource : resources) {
            PropertiesLoaderUtils.fillProperties(props, new EncodedResource(resource, "utf-8"));
        }
    }
}
```
让我们进一步来看一下springboot中的getResources()源码：
```java
//在spring-core-5.0.8的jar包下面的org.springframework.core.io.support 的 PathMatchingResourcePatternResolver 类
@Override
    public Resource[] getResources(String locationPattern) throws IOException {
        Assert.notNull(locationPattern, "Location pattern must not be null");
        if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
            // a class path resource (multiple resources for same name possible)
            if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
                // a class path resource pattern
                return findPathMatchingResources(locationPattern);
            }
            else {
                // all class path resources with the given name
                return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
            }
        }
        else {
            // Generally only look for a pattern after a prefix here,
            // and on Tomcat only after the "*/" separator for its "war:" protocol.
            int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
                    locationPattern.indexOf(':') + 1);
            if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
                // a file pattern
                return findPathMatchingResources(locationPattern);
            }
            else {
                // a single resource with the given name
                return new Resource[] {getResourceLoader().getResource(locationPattern)};
            }
        }
    }
```
其中CLASSPATH_ALL_URL_PREFIX为：
```java
public interface ResourcePatternResolver extends ResourceLoader {

    /**
     * Pseudo URL prefix for all matching resources from the class path: "classpath*:"
     * This differs from ResourceLoader's classpath URL prefix in that it
     * retrieves all matching resources for a given name (e.g. "/beans.xml"),
     * for example in the root of all deployed JAR files.
     * @see org.springframework.core.io.ResourceLoader#CLASSPATH_URL_PREFIX
     */
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

···
}
```
由此可见，只要是classpath*这样子的写法，就会是多匹配模式，即便是同名的也会多包含模式。
