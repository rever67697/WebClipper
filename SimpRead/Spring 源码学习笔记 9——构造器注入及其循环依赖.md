> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/cuzzz/p/16538845.html)

[系列文章目录和关于我](https://www.cnblogs.com/cuzzz/p/16609728.html)

一丶前言[#](#一丶前言)
--------------

前面我们分析了 spring 基于字段的和基于 set 方法注入的原理，但是没有分析第二常用的注入方式（构造器注入）（第一常用字段注入），并且在循环依赖问题上构造器注入常被说 spring 无法解决构造器注入的循环依赖，下面我们来分析构造器注入和其循环依赖的源码

二丶构造器依赖注入[#](#二丶构造器依赖注入)
------------------------

在 spring 初始化每一个非抽象，单例，非懒加载的 bean 的时候，会调用 createBeanInstance 方法去创建 bean 的实例，在使用默认的策略——无参构造 or CGLIB 生成子类对象的方式之前，先会使用所有的 SmartInstantiationAwareBeanPostProcessor 来判断是否有用户指定的后置处理器

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801070957670-1695257199.png)

### 1. 循环调用所有的 SmartInstantiationAwareBeanPostProcessor 实现类来推断该使用哪个构造器[#](#1循环调用所有的smartinstantiationawarebeanpostprocessor实现类来推断该使用哪个构造器)

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071008017-337094425.png)

这里重点是使用到了 AutowiredAnnotationBeanPostProcessor 来推断

### 2.AutowiredAnnotationBeanPostProcessor 推断构造器的逻辑[#](#2autowiredannotationbeanpostprocessor-推断构造器的逻辑)

首先 AutowiredAnnotationBeanPostProcessor 有一个缓存 key 是 bean 类型，value 是之前推断的所有构造器数组。自然是一波双 if+synchronized 方式来读缓存中内容，缓存没有命中再进行推断

```
遍历每一个构造器（这里省去了spring对Kotlin语言支持的部分）
1.获取当前构造器上面的@Autowired 和  @Value 注解
2.如果注解信息为空，且当类是一个被CGLIB代理后的类，
会获取父类构造器上面的注解（要求父类构造器和当前遍历到的构造器参数类型顺序相同）
3.如果注解信息不为空，会检查 required 属性的信息
（spring还支持ejb等的注解，所有这里检查require的信息）像@Autowired 和  @Value这种不包含required属性的注解会默认是required为true，
并且记录当前构造器为候选者，也就是说有注解的才算在候选者
4.如果存在多个required=true的构造器抛出异常，
否则使用遍历记录当前required=true的构造器
5.如果候选者不为空，required=true的构造器为空
（一般这种情况是标注了EJB的注解指定required为false）spring会把无参构造加入到候选者中
6.如果原来类只定义了一个有参构造，那么使用这个有参构造

如果不考虑EJB中的依赖注入注解
那么就是如果构造器有@Autowired or @Value那么默认使用它
如果定义了一个有参构造那么使用这个有参构造

其他情况一律返回null(后续spring可能使用无参构造or CGLIB生成子类的方式)
```

### 3. 使用构造器进行依赖注入的逻辑[#](#3使用构造器进行依赖注入的逻辑)

推断出使用哪个构造器之后，spring 调用 autowireConstructor 方法进行构造器注入，具体逻辑委托给 ConstructorResolver 的 autowireConstructor 方法，最终解析依赖注入参数的在 createArgumentArray 方法中进行，一般是调用 resolveAutowiredArgument 方法

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071022753-1256086516.png)

最终还是殊途同归的调用到了 beanFactory 的 resolveDependency 方法

### 4. 实例化对象[#](#4实例化对象)

这一步还是调用的 instantiate 方法

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071031740-474092791.png)

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071039966-1598317908.png)

三丶为什么构造器的循环依赖 spring 无法解决[#](#三丶为什么构造器的循环依赖spring无法解决)
------------------------------------------------------

### 1. 构造方法注入无法解决循环依赖的情况及其原因[#](#1构造方法注入无法解决循环依赖的情况及其原因)

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071048065-809295066.png)

如果是这种循环依赖的情况 spring 是无法解决循环依赖的，下面是这种循环依赖出现时候的代码流程

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071056616-1898110180.png)

问题出现在 beforeSingletonCreation 方法中

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071104020-257687329.png)

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071111078-308965123.png)

在 spring 创建 bean 的之前使用了 set 维护正在创建 bean 的名称，B 需要 A，再次创建 A 的时候就发现 set 中原本存在 A，这时候就是无法解决循环依赖的情况，会抛出异常

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071117866-825747844.png)

同理这种情况也是不可以解决的

### 2. 构造方法注入可以解决循环依赖的情况及其原因[#](#2构造方法注入可以解决循环依赖的情况及其原因)

![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071126113-452193988.png)

这种情况下 spring 又是可以解决循环依赖的  
![](https://img2022.cnblogs.com/blog/2605549/202208/2605549-20220801071134330-1267454967.png)  
原因是 A 是可以成功使用无参构造实例化的，所以 B 需要 A 的时候可以在三级缓存拿到 A 的 ObjectFactory，调用后得到 A，而不是再次创建 A

```
我们可以得到结论

如果加载顺序是A然后B，那么A只能是字段注入or方法注入，不能是构造器注入
B无所谓那种注入方式
```