---
title: 注解importResource与注解import相关知识点
date: 2020-05-19 13:56:43
tags: 
- '@import'
- '@importResource'
- spring
---

# @importResource注解与@import注解

## @import注解
@import注解用于导入某些特殊的Bean，包括添加了@Configuration注解的类、importSelector接口的实现类、importBeanDefinitionRegiser接口的实现类

### 导入@Configuration类
在springboot中一般使用@CompantScan注解配置扫描的路径，就能自动导入，如果不在扫描的路径下面的配置类就得使用@import注解导入，第三方jar包都需要借助@import注解导入

### 导入importSelector接口的实现类

```
public class MyImportSelector implements ImportSelector {
    
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        全类名数组 = doSomething获得要加载的全类名数组;
        return 全类名数组;
    }
}

@import(MyImportSelector.class)
@SpringBootApplication
public class Application(){
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        System.out.println("start success......");
    }
}
```

此种方式便能导入selectImports方法返回的类名数组。
案例：参考springboot自动装配

### 导入importBeanDefinitionRegiser接口的实现类

```
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(TestBean.class);
        registry.registerBeanDefinition("TestBean", rootBeanDefinition);
    }
}

@import(MyImportBeanDefinitionRegistrar.class)
@SpringBootApplication
public class Application(){
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        System.out.println("start success......");
    }
}
```
为了在springboot启动阶段动态地向容器中注册Bean Definition。
案例：MapperScan导入容器的原理

## @importResource注解
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
public @interface ImportResource {
    @AliasFor("locations")
    String[] value() default {};

    @AliasFor("value")
    String[] locations() default {};

    Class<? extends BeanDefinitionReader> reader() default BeanDefinitionReader.class;
}
```
通过locations属性加载对应的xml配置文件，同时需要配合@Configuration注解一起使用，定义为配置类。