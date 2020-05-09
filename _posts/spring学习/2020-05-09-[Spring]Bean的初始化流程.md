---
title: "「Spring」Bean的初始化流程"
subtitle: "Spring的初始化流程,源码分析"
layout: post
author: "afsun"
header-style: text
hidden: true
categories:
  - Spring源码
tags:
  - Spring
---
# Bean的初始化和获取

在AbstractApplicationContext.refresh方法中的`finishBeanFactoryInitialization`()方法,去初始化非`lazy`且`Singleton`的Bean

分析`preInstantiateSingletons`方法

```java
public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// 只有非抽象的,单例的,非懒加载的BD会进入
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				// 如果BD为FactoryBean则采用FactoryBean的方式初始化
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				// 正常方式初始化
				else {
					getBean(beanName);
				}
			}
		}
	}
```

进入getBean()→doGetBean方法:

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 从(123级)缓存中拿到需要的对象
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		// 如果缓存中没有对象
		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			// 判断是否该bean为原型且已经在创建的
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查父类的容器 不重要
			// ....

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 保证dependon的对象 先被初始化
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		return (T) bean;
	}
```

先分析单例的情况:

`AbstractAutowireCapableBeanFactory`.`createBean`→`doCreateBean`

在`doCreateBean`中会先用`BeanWrapper`将Bean给实例化出来

随后将此实例化对象放入到`SingletonFactory`三级缓存中

```java
// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```

之后再调用此对象的**属性装载**和**初始化**操作

### 引入缓存的意义?

容器中有三级缓存 分别为:

- singletonObjects  对象已经初始化完成
- earlySingletonObjects 未初始化,提前
- singletonFactories 未初始化对象

在getBean时,就会通过判断缓存中是否有对象来直接返回,`singletonObjects` 中如果已经有了说明对象已经初始化完成. 如果没有则去`earlySingletonObjects` 中获取,如果没有再去`singletonFactories` 中获取,如果有则remove对象,并add到`earlySingletonObjects` 中.

### 为什么有三级缓存呢?

SingletonFactories在add时,会判断该BD是否为`SmartInstantiationAwareBeanPostProcessor`的实现类,如果是则调用`getEarlyBeanReference`方法,算是Spring留给对外扩展的点

prototype类型的Bean是不可以有循环依赖的,因为在Bean创建前期就在容器中的ThreadLoca中将此BeanName放入其中,一旦再次获取同一个BeanName就会抛出异常.

此外在构造方法中的循环依赖也是会报错的.因为只有构造方法完成后才会将对象放入到3级缓存中去

### 生命周期

![Bean%2022fd2df30a1a447aa407ce95b5dc3f43/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/141721-429567.png)