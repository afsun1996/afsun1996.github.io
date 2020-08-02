# SpringBoot自动化装配源码阅读

```
---
title: "「Spring」SpringBoot自动化装配源码阅读"
subtitle: "SpringBoot自动化装配源码阅读"
layout: post
author: "afsun"
header-style: text
tags:
  - Spring
---
```

基于约定大于配置的原则来实现自动化装配。

##  @SpringBootApplication注解

在SpringBoot中都会对主类加上一个@SpringBootApplication的注解，加完这个注解就可以直接启动这个类下面的main方法执行：`SpringApplication.run(Client1Application.class, args); ` 就直接将Springboot给运行起来了。 

进入@SpringBootApplication代码内部看一下

![image-20200801162306129](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801162306129.png)

可以看到这个注解上面总共包含了3个注解：`@SpringBotConfiguration`，`@EnableAutoConfiguration`，`@ComponentScan`

![image-20200801162419950](C:\Users\afsun\AppData\Roaming\Typora\typora-user-images\image-20200801162419950.png)1 `@SpringBotConfiguration`，这个注解其实就是一个`@Configuration的`**包装**注解，将被@SpringBootConfiguration修饰的类当作是一个Configuration，在Spring初始化的时候放入到Spring的容器中去。

@ComponentScan显而易见就是把当前类目录和子目录下的所有的Bean都放入Spring容器中去，起到扫描功能。

接下来的注解就是SpringBoot自动化注解的重要注解：@EnableAutoConfiguration

![image-20200801171556353](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801171556353.png)

@EnableAutoConfiguration引入了两个Bean：`AutoConfigurationPackages.Registrar`和`AutoConfigurationImportSelector`

## @AutoConfigurationPackage注解

阅读`AutoConfigurationPackages.Registrar`并调试代码：

![image-20200801204653750](C:\Users\afsun\AppData\Roaming\Typora\typora-user-images\image-20200801204653750.png)

sun.praticle.client1是SpringBootApplication的主类的包路径。PackageImports这个类的数据结构主要是将被@AutoConfigurationPackage修饰的类的root包路径包装起来放入到Spring中管理

## AutoConfigurationImportSelector(※)

`AutoConfigurationImportSelector``实现了`DeferredImportSelector`，`DeferredImportSelector`实现了ImportSelector接口，所以直接阅读selectImports方法。

![image-20200801214844145](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801214844145.png)

阅读loadSpringFactories方法：

![image-20200801215255763](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801215255763.png)zhao 

可以得知，该配置类主要是在META-INF/spring.factories中加载（key = org.springframework.boot.autoconfigure.EnableAutoConfiguration）配置类

![image-20200801220129725](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801220129725.png)

再回到上层代码：

![image-20200801231942762](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200801231942762.png)

该行代码的意思是从`META-INF/spring.factories`中将所有的Key-Value给读取出来，放入一个Map中去。然后根据Key = `EnableAutoConfiguration.class`来获取到相应的Value就是自动化装配的一些启动类。

![image-20200802113317515](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200802113317515.png)



## RedisTemplate自动装配的跟踪

![image-20200802113644883](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200802113644883.png)进入到`RedisAutoConfiguration`代码中阅读：

![image-20200802113715135](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200802113715135.png)

一共有四个注解：

1.  `@Configuration`这个最简单，就是将此Class对象交给Spring来管理。
2. `@ConditionalOnClass`如果Value中的class存在于Spring的IOC时生效
3. `@EnableConfigurationProperties`此注解是将Spring.properties/Spring.ymal中的Kv赋值到此注解对应的类中（这个类其实就是一个POJO），此注解最能体现Spring的**约定大于配置**的理念。
4. `@Import`，这个注解是一个很普通的Spirng注解了，可以导入普通的Bean类，也可以导入Importer类和Selector类

进入@Import中的两个类：

### `JedisConnectionConfiguration` && `LettuceConnectionConfiguration`

![image-20200802114654315](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200802114654315.png)

可以看出来，`JedisConnectionConfiguration`因为缺少`GenericObjectPool`和`Jedis`，所以这个配置类不会生效

### `RedisTemplate`自动装配的顺序

1. 初始化LettuceConnectionConfiguration，根据注解`@ConditionalOnMissingBean(RedisConnectionFactory.class)`，@ConditionalOnClass，这个条件注解会去classpath下查找，jar包里面是否有这个条件依赖类，所以必须有了相应的jar包，才有这些依赖类，才会生成IOC环境需要的一些默认配置Bean
2. ![image-20200802120536904](https://tuchuansun.oss-cn-hangzhou.aliyuncs.com/image-20200802120536904.png)

将创建的RedisConnectionFactory放入到RedusTemplate中去，并将RedisTemplate放入到Spring中管理。

## 自动化装配总结

​	从@EnableAutoConfiguration注解进入到AutoConfigurationImportSelector中去，获取所有JAR包下的META-INF/Spring.factories文件中的键值对。将Key = org.springframework.boot.autoconfigure.EnableAutoConfiguration的Value提取出来，通过@Conditonal注解来过滤，获取到可以注入到Spring中的Bean对象，然后注入。

​	其实SpringBoot主要依赖的是@Import的注解，@Conditionalxx注解和@EnableConfigurationProperties注解来实现自动装配和约定大于编写的功能。