---
title: '"自定义配置文件properties与Spring boot源码启动配置文件分析（三）"' 
categories: java 
tags: springboot 
date: 2018-10-21 22:32:29 
---
在一二中我们分别分析了自定义的配置文件和springboot配置文件如何启动的。接下来我们就对文件启动的顺序进行一个探究。

## 三、配置文件优先级读取分析
### 1.同为默认配置文件，读取顺序
在二中我们还记得，其中的变量为：
>  private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/";  

但是，代码中做了一个reverse的操作，即顺序应该是
>./config/application.properties,
./application.properties,
classpath:./config/application.properties,
classpath:./application.properties

<!--more-->
用图更能说明问题，详见图1。
![图1. 默认配置文件启动顺序图](https://upload-images.jianshu.io/upload_images/14043408-c53db6e9482c12ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.同为自定义配置文件
> 启动顺序是：启动哪个模块(一般为web模块)就以哪个模块的优先。

但是其余模块顺序未知，很有可能跟module的设置顺序有关(*博主未曾实验过，只是一种猜测*)。但是，既然是自定义配置文件了，还是**建议不要有同名的变量配置**，以免给自己找麻烦。

### 3.自定义配置文件和默认配置文件冲突
> 这个时候的优先级是**默认配置文件**的优先级要跟高一些。

换句话说，默认配置文件会比自定义配置文件更加后启动。

### 4.出现多个默认的配置文件
根据二中的分析，我们知道系统只会找到一份默认的配置文件，就会停止搜索。那么会先搜索哪个模块下面的配置文件。
> 还是跟第2点一样，会先搜索启动项目的模块，再去找其他模块。

对于其他模块的搜索顺序，还是未确定数。有兴趣的读者可以自行分析一下源码。
