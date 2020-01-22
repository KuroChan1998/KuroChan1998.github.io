---
layout:     post
title:      简要分析springboot运行流程
subtitle:   倚靠源码简要分析springboot的启动方式和启动流程
date:       2020-1-22
author:     Kuro
header-img: img/tag-bg.jpg
catalog: true
tags:
    - springboot

---

# springboot运行流程分析

## springboot两种启动方式

### 静态方法启动。

```java
@SpringBootApplication(scanBasePackages = "com.jzy")
public class App {
	public static void main(String[] args) {		
		ConfigurableApplicationContext applicationContext = SpringApplication.run(App.class, args);
		applicationContext.close();
    }
}
```

其实这种方式也是用第二种方式实现的，因为这个静态方法内部调用了2中的实例方法。

```java
	/**
	 * Static helper that can be used to run a {@link SpringApplication} from the
	 * specified sources using default settings and user supplied arguments.
	 * @param primarySources the primary sources to load
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return the running {@link ApplicationContext}
	 */
	public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}

```

### 实例方法启动。

```java
@SpringBootApplication(scanBasePackages = "com.jzy")
public class App {
	public static void main(String[] args) {
		SpringApplication springApplication=new SpringApplication();
		HashSet<String> set=new HashSet<>();
		set.add("com.jzy.App");
		springApplication.setSources(set);
		ConfigurableApplicationContext applicationContext=springApplication.run(args);
		applicationContext.close();
	}
}
```

## 流程分析

### new SpringApplication()

以静态方法启动为例，上面`new SpringApplication(...)`的过程如下：

```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		//判断是否是web环境
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		//加载回调ApplicationContextInitializer
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		//加载ApplicationListener
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		//推断运行main方法所在的位置
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

### run()

new完成后调用run方法。

该方法中实现了如下几个关键步骤：

1. 创建了应用的监听器*SpringApplicationRunListeners*并开始监听

   ```java
   	private SpringApplicationRunListeners getRunListeners(String[] args) {
   		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
   		return new SpringApplicationRunListeners(logger,
   				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
   	}
   ```

   ```java
   class SpringApplicationRunListeners {
       ...
       private final List<SpringApplicationRunListener> listeners;
       ...
   }
   ```

2. 加载SpringBoot配置环境(*ConfigurableEnvironment*)，如果是通过web容器发布，会加载*StandardEnvironment*。

   进入`prepareEnvironment`方法：

   ![Snipaste_2020-01-22_15-51-17](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2020-01-22_15-51-17.jpg?raw=true)

   `getOrCreateEnvironment()`创建了environment。

   ```java
   	private ConfigurableEnvironment getOrCreateEnvironment() {
   		if (this.environment != null) {
   			return this.environment;
   		}
   		switch (this.webApplicationType) {
   		case SERVLET:
   			return new StandardServletEnvironment();
   		case REACTIVE:
   			return new StandardReactiveWebEnvironment();
   		default:
   			return new StandardEnvironment();
   		}
   	}
   ```

   这里由于是用的springboot源码做简单调试，所以不是web环境，返回了*StandardEnvironment*。

3. `createApplicationContext()`创建spring上下文context，这个context也是静态run方法的返回值。

   ```java
   	protected ConfigurableApplicationContext createApplicationContext() {
   		Class<?> contextClass = this.applicationContextClass;
   		if (contextClass == null) {
   			try {
   				switch (this.webApplicationType) {
   				case SERVLET:
   					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
   					break;
   				case REACTIVE:
   					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
   					break;
   				default:
   					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
   				}
   			}
   			catch (ClassNotFoundException ex) {
   				throw new IllegalStateException(
   						"Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
   						ex);
   			}
   		}
   		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
   	}
   ```

4. 接下来`prepareContext()`将*listeners、environment、applicationArguments、banner*等重要组件与上下文对象关联

   ```java
   	private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
   			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
   		context.setEnvironment(environment);
   		postProcessApplicationContext(context);
   		applyInitializers(context);
   		listeners.contextPrepared(context);
   		if (this.logStartupInfo) {
   			logStartupInfo(context.getParent() == null);
   			logStartupProfileInfo(context);
   		}
   		// Add boot specific singleton beans
   		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
   		if (printedBanner != null) {
   			beanFactory.registerSingleton("springBootBanner", printedBanner);
   		}
   		if (beanFactory instanceof DefaultListableBeanFactory) {
   			((DefaultListableBeanFactory) beanFactory)
   					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   		}
   		// Load the sources
   		Set<Object> sources = getAllSources();
   		Assert.notEmpty(sources, "Sources must not be empty");
   		//把所有source加载到spring上下文容器
   		load(context, sources.toArray(new Object[0]));
   		listeners.contextLoaded(context);
   	}
   ```

5. refresh后回调所有SpringApplicationRunListeners，springboot就已启动完毕





