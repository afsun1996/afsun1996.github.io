---
title: "「Spring」AOP的实现"
subtitle: "Spring中AOP的实现原理"
author: "afsun"
header-style: text
header-img:"img/post-bg-alitrip.jpg"
tags:
  - Spring
---
# AOP

JAVA实现 AOP

Spring配置类



```java
package com.test.config;

import com.alibaba.druid.pool.DruidDataSource;
import com.test.aop.AopTest;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.*;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.stereotype.Component;
import org.springframework.transaction.TransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.sql.DataSource;

@Configuration
@ComponentScan(basePackages = {"com.test"})
@EnableTransactionManagement
@EnableAspectJAutoProxy(proxyTargetClass=false)
@MapperScan(basePackages = "com.test.mapper",annotationClass = Mapper.class)
public class SpringStart {

    @Bean("dataSource")
    public DataSource getDataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://47.96.234.172:3306/test");
        dataSource.setUsername("root");
        dataSource.setPassword("sunaifei123");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

    @Bean("sqlSessionFactory")
    public SqlSessionFactoryBean getSqlSessionFactoryBean(@Qualifier("dataSource") DataSource dataSource){
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        return sqlSessionFactoryBean;
    }

    @Bean("transactionManager")
    public DataSourceTransactionManager getTransactionManager(@Qualifier("dataSource")@Autowired DataSource dataSource){
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
        transactionManager.setDataSource(dataSource);
        return transactionManager;
    }

    // 模板
    @Bean(name = "sqlSessionTemplate")
    public SqlSessionTemplate testSqlSessionTemplate(
            @Qualifier("sqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

AOP切面配置类

```java
package com.test.aop;   
 /**
 * Created by 孙爱飞 on 2020/3/21.
 */

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

/**
 *@description: //TODO
 *@author: your name
 *@create: 2020-03-21 13:49
 *@version: 1.0
 */
@Aspect
@Component
public class AopTest {

    @Pointcut("execution(* com.test.service.ServiceTest.*(..))")
    public void pointcut11(){}

    @Pointcut("execution(* com.test.service.ServiceTest.update())")
    public void pointcut12(){}

    @Before("pointcut12()")
    public void strengthen(JoinPoint joinPoint){
        System.out.printf("AOP!!!");
    }

}
```

测试类

```java
package com.test;

import com.test.aop.AopTest;
import com.test.config.SpringStart;
import com.test.service.ServiceSupport;
import com.test.service.ServiceTest;
import org.apache.ibatis.session.SqlSessionFactory;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {SpringStart.class})
public class TestSpring {

    @Autowired
    ServiceSupport service;

    @Autowired
    SqlSessionFactory sqlSessionFactory;

    @Autowired
    AopTest aopTest;

    @Test
    public void test1(){

       System.out.println(service);
       System.out.println(service.update());
    }

}
```

## 源码详解

分析注解： @`EnableAspectJAutoProxy`(proxyTargetClass=false)

注解中Import了 `AspectJAutoProxyRegistrar`类，且实现了`ImportBeanDefinitionRegistrar`接口，因此Spring在扫描BD的时候会执行registerBeanDefinitions方法

→ AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

将`AnnotationAwareAspectJAutoProxyCreator`类以`org.springframework.aop.config.internalAutoProxyCreator`为Key注册到BeanFctory的BeanDefinitionMaps中。

因为`AnnotationAwareAspectJAutoProxyCreator`实现了InstantionAwareBeanPostProcessor→BeanPostProcess方法，所以在所有的BD初始化时需要调用`AnnotationAwareAspectJAutoProxyCreator`的xxxbefore和after方法。

分析`AnnotationAwareAspectJAutoProxyCreator`类：

```java
@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				// 看进去，在这里进行代理
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		// 拿到此类上面的AOP切点和拦截器，也就是将AOP做了一次封装。如果不需要代理则为null
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

![AOP%2001b4b87484be4d71908e53f656a13485/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145027-918485.png)

![AOP%2001b4b87484be4d71908e53f656a13485/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145034-194414.png)

总体流程：

`@EnableAspectJAutoProxy`注解，引入一个ImportRegisterBean的实例（`AspectJAutoProxyRegistrar`），该实例向`BeanDefinitionMap`中放入了一个`BeanPostProcessor`的实现类（`AnnotationAwareAspectJAutoProxyCreator`），在Bean实例化的时候也就是getBean时，在初始化阶段（initialzingBean）时会将`BeanPostProcessor`的Set列表进行遍历,逐个的执行`BeanPostProcessor`.before和after方法，在执行`AnnotationAwareAspectJAutoProxyCreator`的时候，会判断此类是否需要被代理，然后判断是使用JDK动态代理或者是CgLib代理。