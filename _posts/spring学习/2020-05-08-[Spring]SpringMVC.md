---
title: "「Spring」SpringMVC"
subtitle: "SpringMVC的初始化流程,源码分析"
layout: post
author: "afsun"
header-style: text
hidden: true
tags:
  - Spring
---
# Servlet概述

- `Servlet`就是JAVA定义的一个接口,他并没有什么实质性的功能,只是一个规范,实现了`Servelt`规范的类可以被`Servelt`容器所识别和使用,如Tomcat就是一种Servlet容器,在请求到达Servlet容器后,会自动将请求和相应封装成`ServletRequest`和`ServletResponse`然后传递给Servlet实现类,调用他的Service方法(HttpServlet中则执行doGet,doPost方法),最后再将Response返回给客户端

Tomcat的流程图

![Servlet%2077fd187aa14247b6acf8f2d92b6fdf08/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145447-91968.png)

Tomcat存在四大组件:

Connector连接监听器,监听端口和请求的协议,将请求转发给Service中定义的Engine

```xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
```

Engine容器,每个Service都有一个引擎,引擎可以设置默认的Host

```xml
<Engine name="Catalina" defaultHost="localhost">
```

Host:引擎中可以存在多个host,请求http://xxxx/aa/bb 中的xxx就对应host中的Name

```xml
<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
```

Context 指定的是项目的名称,如下http://localhost:8080/project1 机会进入到该context中.Context中包含了Servlet的保证类:Wrapper. 执行servlet就是依赖这些Wrapper完成的

```xml
<Context path="/project1"  docBase="/project">
```

## Tomcat初始化过程:

找到tomcat目录下的conf/web.xml 将文件解析,接下如下:

`org.apache.catalina.servlets.DefaultServlet`

```xml
<!-- The mapping for the default servlet -->

	<servlet>
        <servlet-name>default</servlet-name>
        <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
        <init-param>
            <param-name>debug</param-name>
            <param-value>0</param-value>
        </init-param>
        <init-param>
            <param-name>listings</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```

`org.apache.jasper.servlet.JspServlet`

```xml
<!-- The mappings for the JSP servlet -->
<servlet>
        <servlet-name>jsp</servlet-name>
        <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
        <init-param>
            <param-name>fork</param-name>
            <param-value>false</param-value>
        </init-param>
        <init-param>
            <param-name>xpoweredBy</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>3</load-on-startup>
    </servlet>
<servlet-mapping>
        <servlet-name>jsp</servlet-name>
        <url-pattern>*.jsp</url-pattern>
        <url-pattern>*.jspx</url-pattern>
</servlet-mapping>
```

load-on-startup 配置项大于 0，那么在 Context 容器启动的时候就会被实例化,也就是两个默认的Servlet会初始化,这两个`Servelt`会作为所有的applictaion的默认servelt.

接下来初始化项目中的/meta-inf/web.xml 文件中的内容

将`Servlet`解析出来并包装为`Wrapper`对象放入`Context`中作为子容器,其他的参数如过滤器,监听器则作为信息保存在`Context`中,所以`Context`才是一个Web项目的单位

一个HTTP请求的过程:curl http://localhost:8080/project1/xxx

1. 监听8080端口的`Connector`收到请求后,将请求转换成Request对象,并传送给此`Service`中的`Engine`中
2. `Engine`收到请求后,解析hostName得到localhost,在host中匹配name="localhost"的host,并将请求转发过去
3. `Host`继续解析url中的project1,在Host容器中匹配对应的Context,并将请求转发过去
4. `Context`执行相应的`Servelt`并返回`Response`对象,通过逐层传递给客户端

## Servlet中的监听器

时间监听器:

- `ServletContextAttributeListene`      ServletContext域被修改时,调用
- `ServeltRequestAttributeListener`    Request的域被修改时,调用
- `HttpSessionAttributeListener`         Session域被修改时调用
- `ServletRequestListener`                    有Request请求时就调用

生命周期监听器:

- `ServletContextListener`  当Servlet启动的时候调用,ContextLoadListener就是通过此,实现的
- `HttpSessionListner`        Session创建的时候调用

### 过滤器和拦截器

![Servlet%2077fd187aa14247b6acf8f2d92b6fdf08/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145451-598918.png)

过滤器(`Filter`)是直接定义在web.xml中,当Servlet容器启动时,读取的 由Servlet容器来控制他的生命周期

而拦截器(`Interceptior`)则是由SpirngMVC定义的,在DispatcherServlet.dispatcher方法中,对某个requestMapping定义的拦截器进行调用

# SpringMVC初始化

由老式的Spring项目看SpringMvc的整合,SpringMVC的配置文件主要是通过/meta-inf/web.xml中的定义:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app id="WebApp_9" version="2.4" xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">

  <!--加载springMVC-->
  <servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>
      org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springMVC.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.action</url-pattern>
  </servlet-mapping>

  <!--Spring配置-->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:springCfg.xml</param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
</web-app>
```

分析`ContextLoaderListener`

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		// 如果在Servlet域中已经存在SpringContext则报错
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}
		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				// servlet context中就有<context-param>中定义的Spring的配置文件的地址
				this.context = createWebApplicationContext(servletContext);
			}
			// 设置父容器
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					// refresh Spring容器
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			// 设置到Servlet域中
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```

- 读取<context_param>中的配置文件地址,初始化一个Spring的容器,并放在ServletContex中

分析:`org.springframework.web.servlet.DispatcherServlet`

继承关系:

![SpringMVC%20551b433c81d8435c9ec8eb39e71622cb/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145518-909863.png)

### HttpServltBean

从类的全限定名可以看出,从HttpServltBean开始才是Spring管理的Servelt

`HttpServletBean`实现了父类预留的init方法

首先获取web.xml<init-param>中定义的参数,并设置到Servlet属性中

```java
@Override
	public final void init() throws ServletException {

		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				// bw就是包装的DispatcherServlet
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				// 获取init-param中定义的参数
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}

		// Let subclasses do whatever initialization they like.
		initServletBean();
	}
```

![SpringMVC%20551b433c81d8435c9ec8eb39e71622cb/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145519-644957.png)

### FrameworkServlet

将webApplicationContext获取到,并从ServletContext中获取rootApplicationContext,并设置父子关系,然后refresh这个webApplicationContext

```java
protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;

		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			onRefresh(wac);
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}

		return wac;
```

## 总结

1. ContextLoaderListener中读取配置文件并创建父容器
2. HttpServletBean中读取资源文件和配置
3. FrameworkServlet中实例化MVC的容器上下文,并设置父容器

# SpringMVC执行

- Servlet对外暴露的是Service方法,`DispatcherServlet`具有4层继承结构,在`HttpServlet`中将doService区分了不同的请求方法分别用不同的方法来完成调用如:doGet,doPost.... 然而分析到`FramworkServlet`中,在所有的doGet类型的方法中又调用了doService方法,就是将原来按照方法调用分开的方法又聚合到一起去了,由此 在`DispatcherServlet`中调用doService方法

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145552-996386.png)

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145553-189083.png)

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%202.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145553-161770.png)

## DispatcherServlet

```java
/**
	 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
	 * for the actual dispatching.
	 */
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		// 保存请求的快照信息
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		// 将一些Spring的参数和对象设置到request域对象中
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}
		// 真正分发请求的地方
		try {
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				// 恢复快照
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
```

将Request域中set进一些Spring可能用到的对象后,就进入到doDispatch方法,此方法才是SpringMVC最重要的方法:

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 通过handlerMappings去遍历找到可以处理该Request的handler并包装成HandlerExcecutionChain
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				// 从handlerAdapters去遍历,可以处理此handler的适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
				//  遍历HandlerExcecutionChain中的所有Interceptor对象,执行拦截器的PreHandler方法
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 真正执行Controller方法的地方
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				//  执行拦截器的postHandler方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 将ModelAndView对象进行解析返回给request
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```

1. 首先请求过来先通过`HandlerMappings`进行遍历获取到`RequestHandler`的包装类`HandlerExecutionChain`
2. 通过`HandlerExecutionChain`获得`requestHandler`,并遍历`HandlerAdaters`找到能处理此Handler的适配器
3. 首先将`HandlerExecutionChain`中的所有Interceptor拿出来遍历,调用preHandler方法
4. 执行handler的真正方法
5. 继续执行`HandlerExecutionChain` 中的postHandler方法
6. 返回MV对象,使用`ViewResolver`解析`ModelAndView`对象并返回View对象给DispatcherServlt
7. 对View进行渲染,然后返回给客户端

### @ResponseBody如何起到作用?

调用堆栈:

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%203.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145559-336838.png)

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%204.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145559-232338.png)

选择了`RequestResponseBodyMethodProcessor`作为解析器

用到了HttpMessageConvert来转换对象成字符串

并将返回对象的值解析到Response的Body中,将Response的Head的`Content-Type`设置为`application/json`

### HttpMessageConvert

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%205.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145601-871137.png)

![SpringMVC%20792ca2013fed455aa8f4f371026f2f4d/Untitled%206.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/09/145734-49076.png)

转换@RequestBody和@ResponseBody中的对象.

通过read方法反序列化请求中的参数为对象

通过write方法序列化为字符串

默认是使用:MappingJackson2HttpMessageConverter 来完成转换的,其内部的转换组件就是jackson

也可以设置为fastjson来完成