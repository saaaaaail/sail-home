---
title: Spring_Boot_AOP原理详解
date: 2020-08-26 14:49:13
tags:
- spring
- aop
---

# 大致流程
主要分为三个步骤：
1. 创建AnnotationAwareAspectJAutoProxyCreator对象
2. 扫描容器中的切面，创建PointcutAdvisor对象
3. 生成代理类

先总结:
1. 主要使用AnnotationAwareAspectJAutoProxyCreator类来初始化Aop
2. 在Bean实例化之前执行beanPostProcess.postProcessBeforeInstantiation方法来生成所有的Advisor
3. 在Bean实例初始化之后执行beanPostProcess.postProcessAfterInitialization方法来判断一个bean实例能否获得切在它上面的Advisor，如果能获得，说明需要生成代理类，否则不需要生成代理类

# 创建AnnotationAwareAspectJAutoProxyCreator对象
先从自动装配开始
观察AutoConfiguration类
```java
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
```
```java
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class,
		AnnotatedElement.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = false)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
	public static class JdkDynamicAutoProxyConfiguration {

	}

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = true)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
	public static class CglibAutoProxyConfiguration {

	}
}
```
继续观察@EnableAspectJAutoProxy注解，其中proxyTargetClass属性默认为true
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AspectJAutoProxyRegistrar.class})
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;

    boolean exposeProxy() default false;
}
```
导入了一个Register，继续观察这个AspectJAutoProxyRegistrar
```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		//注册 AnnotationAwareAspectJAutoProxyCreator
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		//将 aop 代理方式相关的变量设置到 AopConfigUtils，创建代理类时会读取变量
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
}
	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
			@Nullable Object source) {

		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}
```
可以看到注册了一个AnnotationAwareAspectJAutoProxyCreator类的BeanDefinition对象

AnnotationAwareAspectJAutoProxyCreator这个类实现了BeanFactoryAware接口，能拿到Beanfactory，并最终执行initBeanFactory方法
```java
@Override
	protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.initBeanFactory(beanFactory);
		if (this.aspectJAdvisorFactory == null) {
			//advisor 工厂类
			this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
		}
		//用于创建 advisor
		this.aspectJAdvisorsBuilder =
				new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
	}
```

# 扫描容器中的切面，创建PointcutAdvisor对象
注意上面创建的AnnotationAwareAspectJAutoProxyCreator对象，是SmartInstantiationAwareBeanPostProcessor的子类
在bean初始化之前会完成Advisor对象的初始化。通过执行beanPostProcessor.postProcessBeforeInstantiation()

```java
@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			//advisedBeans用于存储不可代理的bean，如果包含直接返回
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			//判断当前bean是否可以被代理，然后存入advisedBeans
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		//到这里说明该bean可以被代理，所以去获取自定义目标类，如果没有定义，则跳过。
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			//如果最终可以获得代理类，则返回代理类，直接执行实例化后置通知方法
			return proxy;
		}

		return null;
	}
```

# 生成代理类
常规的生成代理类的时机在bean的实例初始化完成以后postProcessAfterInitialization中

```java
@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			//处理循环依赖的判断
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```
```java
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		//获取到合适的advisor，如果为空。如果不为空，则生成代理类。
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
上述方法通过调用getAdvicesAndAdvisorsForBean()方法来获取advisor，该方法最终会调用findEligibleAdvisors()，Eligible意为有资格的，合适的。
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		//获取到合适的advisor，如果为空。如果不为空，则生成代理类。
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
上述方法通过调用getAdvicesAndAdvisorsForBean()方法来获取advisor，该方法最终会调用findEligibleAdvisors()，Eligible意为有资格的，合适的。具体来看下：
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//这里会对获取的advisor进行筛选
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		//添加一个默认的advisor，执行时用到。
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```
最终的筛选规则在AopUtils中：
```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		//......
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			} 
			//调用 canApply 方法，遍历所有的方法进行匹配
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		//......
	}
```
调用canApply方法，遍历被代理类的所有的方法，跟进切面表达式进行匹配，如果有一个方法匹配到，也就意味着该类会被代理。

重点看一下生成代理类的方法
```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			//如果代理目标是接口或者Proxy类型，则走jdk类型
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
- optimize：官方文档翻译为设置代理是否应执行积极的优化，默认为false。
- proxyTargetClass：这个在上面已经提到了，AopAutoConfiguration中指定，默认为true，也就是选择使用 cglib 代理。可以看到该变量和optimize意义一样，之所以这么做，个人理解是为了可以在不同的场景中使用。
- hasNoUserSuppliedProxyInterfaces：是否设置了实现接口。


# jdk实现的代理类
新建一个类IUser实现InvocationHandler类，重写invoke接口
```java
public interface IUserManager {
    void addUser(String id, String password);
}

public class UserManagerImpl implements IUserManager {
 
    @Override
    public void addUser(String id, String password) {
        System.out.println("======调用了UserManagerImpl.addUser()方法======");
    }
}

public class JDKProxy implements InvocationHandler {
    /** 需要代理的目标对象 */
    private Object targetObject;
 
    /**
     * 将目标对象传入进行代理
     */
    public Object newProxy(Object targetObject) {
        this.targetObject = targetObject;
        //返回代理对象
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(), this);
    }
 
    /**
     * invoke方法
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 一般我们进行逻辑处理的函数比如这个地方是模拟检查权限
        checkPopedom();
        // 设置方法的返回值
        Object ret = null;
        // 调用invoke方法，ret存储该方法的返回值
        ret  = method.invoke(targetObject, args);
        return ret;
    }
 
    /**
     * 模拟检查权限的例子
     */
    private void checkPopedom() {
        System.out.println("======检查权限checkPopedom()======");
    }
}
```
在方法执行前后添加操作。
jdk动态代理只能针对实现了接口的类来生成代理，

# cglib实现的代理

```java
package com.jpeony.spring.proxy.compare;
 
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
 
import java.lang.reflect.Method;
 
/**
 * CGLibProxy动态代理类
 */
public class CGLibProxy implements MethodInterceptor {
    /** CGLib需要代理的目标对象 */
    private Object targetObject;
 
    public Object createProxyObject(Object obj) {
        this.targetObject = obj;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(obj.getClass());
        enhancer.setCallback(this);
        Object proxyObj = enhancer.create();
        // 返回代理对象
        return proxyObj;
    }
 
    @Override
    public Object intercept(Object proxy, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        Object obj = null;
        // 过滤方法
        if ("addUser".equals(method.getName())) {
            // 检查权限
            checkPopedom();
        }
        obj = method.invoke(targetObject, args);
        return obj;
    }
 
    private void checkPopedom() {
        System.out.println("======检查权限checkPopedom()======");
    }
}
```

# JDK代理类与Cglib代理类的区别
jdk的是java类库提供的，cglib是第三方库。
jdk需要代理类实现接口，cglib不需要。
jdk是通过反射来实现一个代理接口的匿名类，cglib通过修改字节码来生成子类来处理。


# AspectJ aop与 Spring aop的区别是
AspectJ aop的织入是编译时织入，编译完成生成的class文件就将切面织入到了目标对象或者方法上面。
Spring aop的织入是运行时织入，在运行时生成目标类的代理对象并织入。
