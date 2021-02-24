---
title: Spring_Boot_知识点详解
date: 2020-08-23 21:39:52
tags:
- spring
- spring boot
- java
---

# 自动装配原理

# ioc相关

## FactoryBean和BeanFactory
FactoryBean是工厂Bean，是Spring提供的实例化比较复杂的对象的一种方式，通过实现FactoryBean
接口，重写getObject方法来实现一个工厂Bean，隐藏了实现细节，简化xml的配置。  
BeanFactory是ioc容器的顶级接口。主要完成Bean的注册，创建，访问等工作。


# aop原理

1. 在around中可以用,此时可以使用process()来执行被包裹的代码。

```java
public void around(ProceedingJoinPoint joinpoint) {  
    joinpoint.proceed();  
}  
```
1 2 3 6 7 
3
3 2 3 6 5


1. joinPoint可以获得的参数
```java
joinpoint.getArgs();//輸入的參數列表  
  
joinpoint.getTarget().getClass().getName();//類全路徑  
  
joinpoint.getSignature().getDeclaringTypeName();//接口全路徑  
          
joinpoint.getSignature().getName();//調用的方法  
```

# springboot包含哪些设计模式
## 简单工厂模式 BeanFactory或者ApplicationContext中getBean
> BeanFactory或者ApplicationContext这俩类的区别？
> ApplicationContext是BeanFactory的子类，所以ApplicationContext包含BeanFactory的全部功能并且还有额外的优化。
> BeanFactory对Bean是懒加载的，ApplicationContext是预加载的
> ApplicationContext有对国际化信息支持的类
> ApplicationContext自动注册BeanPostProcess、BeanFactoryPostProcess，BeanFactory需要手动注册
> ApplicationContext提供统一的资源加载策略
> ApplicationContext提供事件传播机制

## 工厂方法

## 单例模式
spring 的 ioc容器是单例的，单例的Bean在初始化完成以后会放入到ioc容器中。

## 适配器模式
Spring MVC里的HandlerAdapter就是将不同的Controller实现以及调用其内部的方法封装到Adapter中。
源代码里不需要通过if else来判断执行哪个Controller，只需要给每个Controller写一个Adapter，然后批量执行Adapter集合就可以了。

Handler有许多种形式，最后给每个Handler写一个Adapter，合并成一个Adapter集合遍历执行即可。

## 装饰器模式
RequestWrapper

## 代理模式
AOP通过动态代理实现

## 观察者模式
事件驱动模型
监听者为观察者，事件为被观察者，监听器注册到事件上，事件有发生就通知监听器

## 策略模式
dataSource

## 模板方法

## 责任链模式

# Spring Boot事务

## 传播级别
PROPAGATION_REQUIRED  
如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择

PROPAGATION_SUPPORTS  
支持当前事务，如果当前没有事务，就以非事务方式执行。

PROPAGATION_MANDATORY 使用当前的事务，如果当前没有事务，就抛出异常。

PROPAGATION_REQUIRES_NEW 新建事务，如果当前存在事务，把当前事务挂起。

PROPAGATION_NOT_SUPPORTED 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

PROPAGATION_NEVER 以非事务方式执行，如果当前存在事务，则抛出异常。

PROPAGATION_NESTED  如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。

