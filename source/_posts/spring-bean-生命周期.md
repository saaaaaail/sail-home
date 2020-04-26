---
title: spring bean 生命周期
date: 2020-04-26 13:25:19
tags:
- java
- spring bean
- spring
---
# spring bean 生命周期
ConfigurableApplicationContext中的refresh()方法，其中几个重要步骤：

1. prepareRefresh();//刷新前的预处理
   - initPropertySource();//初始化一些属性设置
   - getEnvironment().validateRequiredProperties();//验证属性合法性
   - ```earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();//保存容器中的一些事件```
2. obtainBeanFactory();
   - refreshBeanFactory();//创建beanFactory，根据配置加载bean到beanFactory中为BeanDefinitionMap，其中保存的是bean的定义信息，完成bean的注册。
3. prepareBeanFactory(beanFactory);
   - 设置BeanFactory的类加载器
   - 添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
   - 设置忽略的自动装配接口EnvironmentAware、EmbeddedValueResolverAware、xxx
   - 注册可以解析的自动装配:BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
   - 添加BeanPostProcessor【ApplicationListenerDetector】
   - 添加编译时的AspectJ
   - 给BeanFactory中测试一些能用的组件:environment【ConfigurableEnvironment】、
	systemProperties【Map<String, Object>】、
	systemEnvironment【Map<String, Object>】
4. postProcessBeanFactory(beanFactory);
   - 如果子类实现了BeanFactoryPostProcessor接口的postProcessBeanFactory(beanFactory)后置处理器方法，则此时调用这些方法
5. invokeBeanFactoryPostProcessors(beanFactory);
   - 执行BeanFactoryPostProcessor后置处理器的postProcessBeanFactory(beanFactory)方法
   - 包含两个接口:BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
   - 先执行BeanDefinitionRegistryPostProcessor
     - 获取所有实现了BeanDefinitionRegistryPostProcessor接口的后置处理器
     - 首先执行实现了PriorityOrdered优先级接口的后置处理器 ```postProcessor.postProcessBeanDefinitionRegistry(registry)```
     - 然后执行实现了Ordered顺序接口的后置处理器 ```postProcessor.postProcessBeanDefinitionRegistry(registry)```
     - 最后执行没有优先级接口与顺序接口的后置处理器```postProcessor.postProcessBeanDefinitionRegistry(registry)```
   - 然后执行BeanFactoryPostProcessor
     - 获取所有实现了BeanFactoryPostProcessor接口的后置处理器
     - 首先执行实现了PriorityOrdered优先级接口的后置处理器 ```postProcessor.postProcessBeanFactory()```
     - 然后执行实现了Ordered顺序接口的后置处理器 ```postProcessor.postProcessBeanFactory()```
     - 最后执行没有优先级接口与顺序接口的后置处理器 ```postProcessor.postProcessBeanFactory()```
6. registerBeanPostProcessors(beanFactory);//注册Bean的后置处理器
   - 不同接口类型的BeanPostProcessor，在bean创建前后的执行时机也不同，例：```-。-```
   - 获取所有的BeanPostProcessor后置处理器；后置处理器都可以使用PriorityOrdered、Ordered接口来设置执行的优先级
   - 先注册了实现PriorityOrdered接口的BeanPostProcessor到BeanFactory中:```beanFactory.addBeanPostProcessor(postProcessor);```
   - 然后注册Ordered接口的
   - 最后注册没有实现优先级接口与顺序接口的
   - 然后注册MergedBeanDefinitionPostProcessor
   - 然后注册一个ApplicationListenerDetector，用于判断bean创建完成后检查是否是ApplicationListener，如果是添加监听器```applicationContext.addApplicationListener((ApplicationListener<?>) bean);```
7. initMessageSource();
   - 获取beanFactory
   - 看容器中是否有id为messageSource，类型为MessageSource的组件
   - 如果有，赋值给messageSource，如果没有创建一个DelegatingMessageSource；
   - 把创建好大的messageSource注册到容器中，获取国际化配置文件的值的时候，可以自动注入MessageSource ```beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);MessageSource.getMessage(String code, Object[] args, String defaultMessage, Locale locale);```
8. initApplicationEventMulticaster();//初始化事件注册表，用于根据事件类型获得其对应的监听器集合
   - 获取beanFactory
   - 从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster
   - 如果上一步为空，则创建一个SimpleApplicationEventMulticaster添加到beanFactory中
9.  onRefresh();//子类重写此方法，容器刷新时自定义逻辑
10. registerListeners();//将容器中的ApplicationListener注册到ApplicationEventMulticaster中
11. finishBeanFactoryInitialization(beanFactory);//初始化剩下的所有单实例bean
    - beanFactory.preIntantiateSingletons();
      - 判断是否是FactoryBean；是否是实现了FactoryBean接口的bean
      - 若不是工厂bean，则利用getBean方法开始创建bean
      - getBean(beanName) -> doGetBean()
      - transformedBeanName(name);//处理bean的别名
      - ```Object sharedInstance = getSingleton(beanName);//从spring的缓存中获得bean```
      - ```BeanFactory parentBeanFactory = this.getParentBeanFactory();//获得bean的父容器，如果该容器不为空，且包含beanName的定义信息，从该容器中返回bean```
      - ```RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);//上面的步骤都没有获得bean的实例则开始创建，首先获得bean的定义信息```
      - 从beanDefinition中获得当前bean DependOn的bean(由@DependsOn或标签depend-on定义，表示当前Bean依赖的bean集合)，循环初始化这些beanName的实例，通过getBean(beanName)方法
        - ```String[] dependsOn = mbd.getDependsOn();//获得当前bean依赖的beanName集合```
        - 循环遍历dependsOn集合（spring中的dependentBeanMap表示【依赖我的】集合，dependenciesForBeanMap表示【我依赖的】集合）
          - 判断```this.isDependent(beanName, dep)```为false继续执行，为true表明存在循环依赖，抛错（判断dependentBeanMap是否包含dep），这里是禁止循环依赖的
          - ```registerDependentBean(dep, beanName);```注册当前beanName到dep的dependentBeanMap集合中去
          - ```getBean(dep);```加载这些依赖的bean
      - 根据beanDefinition mbd中当前bean是Singleton还是Prototype或是其他作用域执行不同的创建实例的方法
      - 对于Singleton而言进行了一层同步回调与单例控制，对于Prototype则直接创建新的实例
      - ```instance = this.createBean(beanName, mbd, args);//开始创建单实例bean```
      - ```beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);//让实现了【InstantiationAwareBeanPostProcessor】接口的后置处理器执行postProcessBeforeInstantiation()方法，如果postProcessBeforeInstantiation()方法的返回值不为null，则调用postProcessAfterInitialization()方法```
      - ```beanInstance = this.doCreateBean(beanName, mbdToUse, args);//创建bean的方法```
        - 【创建Bean实例】```BeanWrapper instanceWrapper = this.createBeanInstance(beanName, mbd, args);//利用工厂方法或者对象的构造器创建出Bean实例```
        - ```this.singletonFactories.put(beanName, singletonFactory);this.earlySingletonObjects.remove(beanName);```【如果允许循环依赖，第一次暴露bean的引用，解决循环依赖问题】，即将刚刚实例化结束的对象暴露出去，暂存到三级缓存singletonFactories中，同时清理二级缓存earlySingletonObjects
        - ```this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);//```
        - 【Bean属性赋值】```this.populateBean(beanName, mbd, instanceWrapper);//Bean属性赋值```
          - 获取InstantiationAwareBeanPostProcessor后置处理器，调用postProcessAfterInstantiation()
          - 获取nstantiationAwareBeanPostProcessor后置处理器，调用postProcessPropertyValues()
          - applyPropertyValues(beanName, mbd, bw, pvs);//使用setter方法为bean属性赋值
        - 【Bean初始化】```this.initializeBean(beanName, exposedObject, mbd);//初始化Bean```
          - 【执行Aware接口方法】```this.invokeAwareMethods(beanName, bean);//执行xxxAware接口的方法，BeanNameAware\BeanClassLoaderAware\BeanFactoryAware Aware接口是spring将对应数据暴露出去的一种方式```
          - 【执行后置处理器初始化之前】```wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);//方法中会执行后置处理器的beanPostProcessor.postProcessBeforeInitialization()方法```
          - 【执行初始化方法】```this.invokeInitMethods(beanName, wrappedBean, mbd);```
            - 判断有没有实现InitializingBean接口，如果实现了则调用afterPropertiesSet方法
            - 如果配置了init-method则执行自定义的初始化方法
          - 【执行后置处理器初始化之后】```wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);//方法中会执行后置处理器的beanPostProcessor.postProcessAfterInitialization()方法```
          - 【获得二级缓存中的对象，若不为空说明存在循环依赖，需要第二次暴露bean的引用，解决循环依赖会导致的一种问题】若当前对象A的引用发生了变化，且存在循环依赖，对象为B，要提前创建B，创建B的时候会通过getSingleton()方法获得A，这时候是从三级缓存中取得刚刚实例化的A引用，并放入到二级缓存中，如果A发生了变化，且B创建成功了说明B中的A与当前A不一样，违反了单例，报错
            - ```Object earlySingletonReference = this.getSingleton(beanName, false);//获得二级缓存中的对象```如果这个对象不为null，说明存在循环引用，因为只有循环引用创建时会将缓存从三级移入到二级缓存
        - ```this.registerDisposableBeanIfNecessary(beanName, bean, mbd);//注册Bean的销毁方法```
      - ```if (newSingleton) {this.addSingleton(beanName, singletonObject);}```如果是Singleton作用域，则会调用上述方法，将初始化完毕的实例更新到一级缓存singletonObjects，并清空二级缓存与三级缓存，如果是其他作用域没有这一步。
12. finishRefresh();//完成BeanFactory初始化创建工作。ioc容器创建完成。
    - initLifecycleProcessor();//初始化和生命周期有关的后置处理器LifecycleProcessor
    - getLifecycleProcessor().onRefresh();//拿到前面定义的生命周期后置处理器，回调onRefresh()
    - publishEvent(new ContextRefreshedEvent(this));//发布容器刷新完成事件
    - LiveBeansView.registerApplicationContext(this);