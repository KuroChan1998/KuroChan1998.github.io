---
layout:     post
title:      Spring AOP源码分析之代理对象的产生
subtitle:   Spring Aop的所谓的代理究竟在哪里得到应用，代理对象是在哪里产生的？
date:       2019-09-24
author:     Kuro
header-img: img/home-bg-geek.jpg
catalog: true
tags:
    - java
    - spring
    - spring aop
    - 代理
    - AspectJ
    - 源码解析

---

# Spring AOP源码分析——代理对象如何产生

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

    StudentService studentService = (StudentService) ac.getBean("studentServiceImpl");
	//这里query()用aop做了增强
    studentService.query();
    ac.close();

}
```

StudentService是一个接口，使用aop增强其方法，则spring会产生一个**实现该接口的代理对象**来实现aop。

实现的时机**在bean初始化阶段(`new AnnotationConfigApplicationContext(AppConfig.class)`)，而非getBean("studentServiceImpl")！！！**

在springbean初始化过程中，**代理对象的产生**是在`doGetBean`方法中。

![Snipaste_2019-09-23_22-01-28](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-01-28.jpg?raw=true)

在`doGetBean`方法中，关键的是getSingleton方法（有两个参数的那个，前面还有一个重载的getSingleton方法）

![Snipaste_2019-09-23_22-12-25](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-12-25.jpg?raw=true)

该方法中，此处getObject()完成了用户的实现StudentService接口的**代理对象**！

![Snipaste_2019-09-23_22-18-29](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-18-29.jpg?raw=true)

根据lambada表达式，真正实现getSingleton方法中的参数return的createBean()实现了getObject()方法。

```java
	sharedInstance = getSingleton(beanName, () -> {
					...
							return createBean(beanName, mbd, args);
					...
					});
```

我们来分析createBean()方法，其中这里doCreateBean方法依旧返回了**代理对象**，继续进入该方法查看。

![Snipaste_2019-09-23_22-33-54](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-33-54.jpg?raw=true)

调用生成的instanceWrapper的getWrappedInstance方法返回了我们的**实例对象**StudentServiceImpl！！！

```java
		instanceWrapper = createBeanInstance(beanName, mbd, args);

		final Object bean = instanceWrapper.getWrappedInstance();
```

![Snipaste_2019-09-23_22-38-21](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-38-21.jpg?raw=true)

然后在initializeBean中生成了实现StudentService的**代理对象**，进入。

![Snipaste_2019-09-23_22-43-34](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-43-34.jpg?raw=true)

在applyBeanPostProcessorsAfterInitialization方法中生成了实现StudentService的**代理对象**

![Snipaste_2019-09-23_22-43-34](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-23_22-43-34.jpg?raw=true)

在applyBeanPostProcessorsAfterInitialization方法中的for循环的所有后置处理器中，AnnotationAwareAspectJAutoProxyCreator实现了对象的代理。

配置类中`@EnableAspectJAutoProxy`注解的作用就是让该处理器出现在这个循环的列表中！

```java
@ComponentScan("com.jzy")
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
```

![Snipaste_2019-09-24_09-09-42](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-24_09-09-42.jpg?raw=true)

步入方法继续调试

![Snipaste_2019-09-24_09-16-03](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-24_09-16-03.jpg?raw=true)

createProxy方法完成代理返回**proxy**！！！

```java
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {
		...
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

由proxyFactory工厂返回代理对象

```java
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
```

- createAopProxy()返回AopProxy接口，该接口的实现：

  - CglibAopProxy：cglib动态代理
  - JdkDynamicAopProxy：jdk动态代理
  - ObjenesisCglibAopProxy

  ```java
  	@Override
  	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
  		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
  			Class<?> targetClass = config.getTargetClass();
  			if (targetClass == null) {
  				throw new AopConfigException("TargetSource cannot determine target class: " +
  						"Either an interface or a target is required for proxy creation.");
  			}
  			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
  				return new JdkDynamicAopProxy(config);
  			}
  			return new ObjenesisCglibAopProxy(config);
  		}
  		else {
  			return new JdkDynamicAopProxy(config);
  		}
  	}
  ```

- getProxy方法返回代理对象

  - createAopProxy()若返回jdk动态代理，调用JdkDynamicAopProxy的getProxy方法

    jdk原生动态代理实现！

    ![Snipaste_2019-09-24_09-33-37](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-24_09-33-37.jpg?raw=true)

  - createAopProxy()若返回cglib动态代理，调用CglibAopProxy的getProxy方法

至此，我们就产生了我们目标对象的aop代理对象。

## 参考

[https://www.bilibili.com/video/av64012960](https://www.bilibili.com/video/av64012960)

[https://shimo.im/docs/Nj0bcFUy3SYyYnbI/read](https://shimo.im/docs/Nj0bcFUy3SYyYnbI/read)