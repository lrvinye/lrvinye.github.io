---
title: Spring Boot 中 AOP的使用
date: 2020-06-14 15:07:59
category: 后端
tags: [SpringBoot,JAVA,AOP,切面编程]
---



**AOP 面向切面编程** 与传统的**OOP**相比，为我们提供了更加强大的功能

在`Spring`大家族中spring的AOP是spring的两大功能模块之一，提供了优秀的解耦解决方案。



*在springboot中，可以很方便的使用spring提供的aop切面*

## 注入依赖

```xml
<dependency>  
   <groupId>org.springframework.boot</groupId>  
   <artifactId>spring-boot-starter-aop</artifactId>  
</dependency>  
```





## 通过注解标识创建切面类

```java
@Aspect //把当前类标识为一个切面供容器读取
@Component //注入spring
public class XXXAspect {...}
```



## 标识切面`PointCut`的切入点

可通过多种函数进行匹配切点

- **execution切点函数**

匹配方法执行的连接点，语法为：

execution(方法修饰符(可选)  返回类型  方法名  参数  异常模式(可选)) 

参数部分允许使用通配符：

\*  匹配任意字符，但只能匹配一个元素

.. 匹配任意字符，可以匹配任意多个元素，表示类时，必须和*联合使用

\+  必须跟在类名后面，如Horseman+，表示类本身和继承或扩展指定类的所有类

`ex:` 

`execution(public * com.example.controller.*.*(..))` 

表示所有`com.example.controller`包下面的**所有**类的所有**公共**方法

`execution(private * com.example.controller.User*.*(..))`

表示所有`com.example.controller`包下面的**以`User`开头的**类的所有**私有**方法

`execution(protected * com.example.controller.*V1.*(..))`

表示所有`com.example.controller`包下面的**以`V1`结尾的**类的所有**包保护**方法



- **@annotation()**

标注了指定注解的目标类**方法**

如：

 `@annotation(org.springframework.transaction.annotation.Transactional)` 表示标注了`@Transactional`的方法



`execution`常用于匹配特定的方法，或者匹配某些类，如所有的controller类，

是一种范围较大的切面方式，多用于日志或者事务处理等。



- **args()**

通过 目标类方法的参数类型 指定切点

如 `args(String)` 表示有且仅有一个`String`型参数的方法



- **@args()**

通过 目标类**参数**的对象类型 是否标注了 指定注解指定切点

如 `@args(org.springframework.stereotype.Service)` 表示有且仅有一个标注了@Service的类参数的方法



- **within()**

通过 类名 指定切点

如 `with(examples.model.Man)` 表示Man类的所有方法



- **@within()**

匹配 标注了指定注解的类 及其**所有**子类

如 `@within(org.springframework.stereotype.Service)` 给Man加上@Service标注，

则Man和Woman的所有方法都匹配



- **target()**

通过 类名 指定，同时包含**所有**子类

如 `target(examples.model.Man) ` 并且 `Woman extends Man`，则两个类的所有方法都匹配



- **@target()**

所有标注了指定注解的**类**

如 `@target(org.springframework.stereotype.Service) `表示所有标注了@Service的类下面的所有方法



- **this()**

与`target()`的区别是`this`是在运行时生成代理类后，才判断代理类与指定的对象类型是否匹配



### **逻辑运算符**

上面提到的函数表达式都可由多个切点函数通过逻辑运算组成

-  **&&**

可以写成and

- **||**

可以写成or

- **!**

可以写成not



## 切面方法使用

```md
@Before
标识一个前置增强方法，相当于BeforeAdvice的功能
    用于在切点前执行一些操作

@AfterReturning
后置增强，相当于AfterReturningAdvice，方法退出时且有返回值时(没有抛出异常)执行
    可以通过@AfterReturning(returning = “XXX”)，获取controller里方法的返回值(XXX)

@AfterThrowing
异常抛出增强，相当于ThrowsAdvice
	可以通过@AfterThrowing(throwing = "XXX")获得抛出的异常(XXX)

@After
final增强，不管是抛出异常或者正常退出都会执行

除了@Around外，每个方法里都可以加或者不加参数JoinPoint，
JoinPoint里包含了类名、被切面的方法名，参数等属性，可供读取使用
================================================================================

@Around
环绕增强，相当于MethodInterceptor
	在这个方法里面,必须带有参数 `ProceedingJoinPoint` 
	使用 ProceedingJoinPoint.proceed 的前后就相当于@Before与@After
	执行 ProceedingJoinPoint.proceed 相当于调用执行了切点方法
	该方法的返回值为 Object 即为切点方法执行的返回值
```



### 切面中方法执行顺序

1. @Around中 ProceedingJoinPoint.proceed 执行前
2. @Before
3. @Around中 ProceedingJoinPoint.proceed 执行后
4. @After
5. @AfterReturning (无异常) 或 @AfterThrowing (有异常被抛出)



**如果有多个切面同时作用：**

进入范围大的 → 进入范围小的 → 离开范围小的 → 离开范围大的

[^范围]:  切面切入点的匹配范围，ex: 切面A匹配了 com.ex.Man 这个类，切面B匹配了 com.ex.Man.dosome 这个类方法，则 范围比较：A>B



