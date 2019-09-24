---
layout:     post
title:      springmvc工作流程
subtitle:   从源码来简要分析springmvc工作流程
date:       2019-09-17
author:     Kuro
header-img: img/tag-bg.jpg
catalog: true
tags:
    - java
    - springmvc
    - springmvc工作流程
    - spring
    - 源码解析

---

# SpringMVC的工作流程?

![img](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/733213-20180401021502228-788157259.jpg?raw=true)

## 源码解析

*DispatcherServlet*类核心方法`doDispatch`

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...
    // Determine handler for the current request.
    mappedHandler = getHandler(processedRequest);
	...
    // Determine handler adapter for the current request.
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	...
    // Actually invoke the handler.
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    ...
}
```

`getHandler(processedRequest)`;获取当前request的处理器映射器

![Snipaste_2019-09-19_12-54-19.jpg](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_12-54-19.jpg?raw=true)

在*handlerMappings*这个列表的所有*handlers*中有多个*HandlerMapping*，起中两个关键的是用来存放以beanName形式和`@Controller`形式注册的Controller，**参见下文Controller的2大类型**！！！！

在`gethandler`过程中，requestMapping中的请求路径参数被存到了*lookupPath*中

![Snipaste_2019-09-19_15-01-57](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_15-01-57.jpg?raw=true)

`gethandler`方法返回的*HandlerExecutionChain*中有我们加了注解的的ViewController!

![Snipaste_2019-09-19_13-03-20](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_13-03-20.jpg?raw=true)

`getHandlerAdapter()`获取当前request下得到的*handlers*返回处理器适配器

![Snipaste_2019-09-19_13-12-00](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_13-12-00.jpg?raw=true)

- *handler*和*adaptor*都是在DispatcherServlet.properties中获取，springboot就是基于这里拓展*handlerMappings*

  ![Snipaste_2019-09-19_13-15-18](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_13-15-18.jpg?raw=true)

- 注意！：在***support()***方法中会判断*handler*的特性用来返回合适的<u>adaptor</u>，这里support方法有多种判断方式，如对`@Controller`注解的Controller中的方法对用*RequestHandlerMapping*对应的support实现来检查，对beanName形式的也有其他类实现的support方法检查

  ![Snipaste_2019-09-19_15-22-53](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_15-22-53.jpg?raw=true)

  因此我们的Controller（handler）的注册也有多种方法！**不只是@Controller**！

  在springmvc中Controller有：

  - 2大类型

    1. 在spring中注册beanName

       ```java
       @Component("/BeanNameController")
       ```

       注意：在`getHandler`方法中的*handlerMappings*中有两个*HandlerMapping*，这里*BeanNameUrlHandlerMapping*中存放了我们通过spring组件形式注册的Controller!!这里存储的是**map结构**！！！

       ![Snipaste_2019-09-19_14-50-10](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_14-50-10.jpg?raw=true)

    2. `@Controller`注解

       *RequestHandlerMapping*中存放了`@Controller`注解注册的Controller

       ![Snipaste_2019-09-19_14-47-11](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_14-47-11.jpg?raw=true)

  - 3种实现

    1. @Controller

    2. implements Controller

       ```java
       @Component("/BeanNameController")
       public class BeanNameController implements Controller {
           @Override
           public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
               System.out.println("implements Controller");
               return null;
           }
       }
       ```

    3. implements HttpRequestHandler

       ```java
       @Component("/HandleController")
       public class HandleController implements HttpRequestHandler {
           @Override
           public void handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
               System.out.println("extends HttpRequestHandler ");
           }
       }
       ```

调用`handle()`执行Controller方法

- 这里以beanName类型Controller为例，通过强转后直接调用我们的Controller中重写的***handleRequest***方法！！！！这样意味着每个类只能有一个方法调用！

  ![Snipaste_2019-09-19_15-40-07](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_15-40-07.jpg?raw=true)

- @Controller

  通过反射处理Controller中的参数和方法

  在*RequestMappingHandlerAdapter*中`invokeHandleMethod`方法中的`invokeAndHandle`方法

  ```java
  	private ModelAndView invokeHandleMethod(HttpServletRequest request,
  			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
  		....
  		requestMappingMethod.invokeAndHandle(webRequest, mavContainer);
  	}
  ```

  其中调用了`getArgumentResolver`。*argumentResolverCache*为变量缓存池

  ```java
  private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
  	//缓存池中区
      HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
      if (result == null) {
          ....
              //这里判断入参是什么类型的是String?还是自定义的对象User?还是request、response（这两者在serlevt父类中有因此不用额外解析）
              if (methodArgumentResolver.supportsParameter(parameter)) {
                  result = methodArgumentResolver;
                  this.argumentResolverCache.put(parameter, result);
                  break;
              }
          ...
      }
      return result;
  }
  ```

  （注意java1.8之前由于无法获取函数变量名称，所以springmvc是通过字节码获得变量名的）

  *InvocableHandlerMethod*中的`getMethodArgumentValues`方法中获取入参user对象的值，通过调用`resolveArgument`存在args[]数组中

  ![Snipaste_2019-09-19_18-45-40](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_18-45-40.jpg?raw=true)

  这里调用invoke反射执行获得了返回的json数据“resultSuccess"！！！

  ![Snipaste_2019-09-19_18-47-46](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-19_18-47-46.jpg?raw=true)



## 简略版流程：

1. 用户发送请求至前端控制器**DispatcherServlet**
2. DispatcherServlet收到请求调用**HandlerMapping**处理器映射器。
3. 处理器映射器根据请求url找到具体的处理器handler(即涵盖了我们的Controller，见上图），生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet。
4. DispatcherServlet通过**HandlerAdapter**处理器适配器调用处理器
5. **HandlerAdapter**执行处理器(Controller)。
6. **Controller**执行完成返回ModelAndView ----------------------------之后源码待分析，我太菜了.....
7. HandlerAdapter将controller执行结果ModelAndView返回给DispatcherServlet
8. DispatcherServlet将ModelAndView传给**ViewReslover**视图解析器
9. ViewReslover解析后返回具体View
10. DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）。
11. DispatcherServlet响应用户

## 附：springmvc初始化的两种方式

- java

  ```java
  public class MyWebApplicationInitializer implements WebApplicationInitializer {
  
      @Override
      public void onStartup(ServletContext servletCxt) {
  
          // Load Spring web application configuration
          AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
          ac.register(AppConfig.class);
          ac.refresh();
  
          // Create and register the DispatcherServlet
          DispatcherServlet servlet = new DispatcherServlet(ac);
          ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
          registration.setLoadOnStartup(1);
          registration.addMapping("/app/*");
      }
  }
  ```

  *SpringServletContainerInitializer*类`onStartup`方法即完成初始化

  ```java
  //扫描所有实现WebApplicationInitializer接口的类，将其封装到onStartup方法中的 Set<Class<?>> webAppInitializerClasses中
  @HandlesTypes(WebApplicationInitializer.class)
  public class SpringServletContainerInitializer implements ServletContainerInitializer {
      	@Override
  	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
  			throws ServletException {
  		...
  		//这里遍历initializers，包括可以是我们自己定义的实现WebApplicationInitializer的类，如上MyWebApplicationInitializer
  		for (WebApplicationInitializer initializer : initializers) {
  			initializer.onStartup(servletContext);
  		}
  	}
  }
      
  ```

- xml

  ```xml
  <web-app>
  
      <listener>
          <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
      </listener>
  
      <context-param>
          <param-name>contextConfigLocation</param-name>
          <param-value>/WEB-INF/app-context.xml</param-value>
      </context-param>
  
      <servlet>
          <servlet-name>app</servlet-name>
          <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
          <init-param>
              <param-name>contextConfigLocation</param-name>
              <param-value></param-value>
          </init-param>
          <load-on-startup>1</load-on-startup>
      </servlet>
  
      <servlet-mapping>
          <servlet-name>app</servlet-name>
          <url-pattern>/app/*</url-pattern>
      </servlet-mapping>
  
  </web-app>
  ```

## 参考

[https://www.bilibili.com/video/av32519360?from=search&seid=6662549753567244872](https://www.bilibili.com/video/av32519360?from=search&seid=6662549753567244872)

