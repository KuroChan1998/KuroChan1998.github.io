---
layout:     post
title:      Spring Bean的生命周期
subtitle:   从源码来解析spring bean的一生 
date:       2019-09-20
author:     Kuro
header-img: img/tag-bg.jpg
catalog: true
tags:
    - java
    - spring
    - spring bean生命周期
    - 源码解析
---

# 从源码解析Spring Bean的生命周期

![img](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/937513-20160507202024015-234747937.png?raw=true)

> 扫描——解析——getBean——实例化——自动装配——life callback——proxy

1. AnnotationConfigApplicationContext获取bean.class配置，调用父类GenericApplicationContext的构造器
2. GenericBeanDefinition(BeanDefinition)存bean的信息，可以通过postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)方法修改bean的属性

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		GenericBeanDefinition beanDefinition= (GenericBeanDefinition) beanFactory.getBeanDefinition("userDao");
		beanDefinition.setBeanClass(StudentDao.class);
	}
}
```

3. 然后将BeanDefinition中的信息put Map

```java
	/** Map from serialized id to factory instance */
	private static final Map<String, Reference<DefaultListableBeanFactory>> serializableFactories = new ConcurrentHashMap<>(8);
```

![Snipaste_2019-09-17_17-54-52.jpg](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-17_17-54-52.jpg?raw=true)

4. 开始实例化单例对象过程：

```java
				// Instantiate all remaining (non-lazy-init) singletons.
				//实例化单例对象
				finishBeanFactoryInitialization(beanFactory);
```

```java
        // Instantiate all remaining (non-lazy-init) singletons.
        //实例化所有单例对象
        beanFactory.preInstantiateSingletons();
```

getBean(beanName);

调用doGetBean( );

其中singletonObjects这个Map类型就是**所谓的bean容器**！！！！它也可以称作**单例池**

![Snipaste_2019-09-17_22-07-59](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-17_22-07-59.jpg?raw=true)

​	在初始化配置的时候就在池中创造这个bean，再getBean的时候直接调用getSingleton()从池中取（第一次调用）

![Snipaste_2019-09-17_22-35-38](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-17_22-35-38.jpg?raw=true)

doGetBean方法中第二次调用getSingleton()

![Snipaste_2019-09-17_22-35-38](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-17_22-35-38.jpg?raw=true)

5. Initialization callbacks

   initializeBean()方法中完成

   ![Snipaste_2019-09-18_13-44-44](https://github.com/KuroChan1998/KuroChan1998.github.io/blob/master/img/mdimg/Snipaste_2019-09-18_13-44-44.jpg?raw=true)

- @PostConstruct（先）

  ```java
  	@PostConstruct
  	public void  init(){
  		System.out.println("life callback");
  	}
  ```

- init-method="init"

  ```xml
  <bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
  ```

  ```java
  public class ExampleBean {
  
      public void init() {
          // do some initialization work
      }
  }
  ```

- implements InitializingBean（后）

  ```java
  public class AnotherExampleBean implements InitializingBean {
  
      public void afterPropertiesSet() {
          // do some initialization work
      }
  }
  ```

7. BeanPostProcessor后置处理器（在springbean初始化过程中共有9个！如resolveBeforeInstantiation）

   循环 依赖解决也是通过后置处理器完成的

   ```java
   @Component
   public class MyBeanPostProcessor implements BeanPostProcessor {
   	/**
   	 * Initialization callbacks前调用，这是第七个后置处理器
   	 **/
   	@Override
   	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   		if (beanName.equals("userDao")){
   			System.out.println(beanName+" postProcessBeforeInitialization");
   		}
   		return bean;
   	}
   
   	/**
   	 * Initialization callbacks后调用，这是第八次调用处理器
   	 **/
   	@Override
   	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   		if (beanName.equals("userDao")) {
   			System.out.println(beanName + " postProcessAfterInitialization");
   		}
   		return bean;
   	}
   }
   ```

8. 使用吧。。。

