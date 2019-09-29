---
layout:     post
title:      @Autowired写在变量上和构造器上的区别
subtitle:   看书的时候正好看到@Autowired写构造器上的用法，而平时一直使用加在变量上，所以查阅了以下资料
date:       2019-09-29
author:     Kuro
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - java
    - spring
    - @Autowired
	- spring ioc
	- di

---

# @Autowired写在变量上和构造器上的区别

```java
public class Test{
    @Autowired
    private A a;
 
    private final String prefix = a.getExcelPrefix();
 
	........
}
```

这种方式会报错。

这样写就不报错了

```java
public class Test{
    private final String prefix;
 
    @Autowired
    public Test(A a) {
        this.prefix= a.getExcelPrefix();
    }
 
........
}
```

其实这两种方式都可以使用，但报错的原因是**加载顺**序的问题，@autowired写在变量上的注入要等到类完全加载完，才会将相应的bean注入，而变量是在加载类的时候按照相应顺序加载的，所以**变量的加载要早于@autowired变量的加载**，那么给变量**prefix 赋值的时候所使用的a，其实还没有被注入**，所以报空指针，而使用构造器就在加载类的时候将a加载了，这样在内部使用a给prefix 赋值就完全没有问题。

如果不适用构造器，那么也可以不给prefix 赋值，而是在接下来的代码使用的地方，通过a.getExcelPrefix()进行赋值，这时的对a的使用是在类完全加载之后，即a被注入了，所以也是可以的。

总之，@Autowired一定**要等本类构造完成**后，才能从外部引用设置进来。所以**@Autowired的注入时间一定会晚于构造函数的执行时间**。

## 参考

[https://blog.csdn.net/qq_17312239/article/details/80819507](https://blog.csdn.net/qq_17312239/article/details/80819507)

