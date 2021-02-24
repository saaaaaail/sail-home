---
title: SpringBoot自动装配详解
date: 2020-08-12 18:40:51
tags:
---


# @EnableAutoConfiguration

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
//1.5.x版本@Import({EnableAutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

# @Import({AutoConfigurationImportSelector.class})
首先，import注解导入了AutoConfigurationImportSelector这个Selector类。
在这个类中的selectImport方法
```java
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
            AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
            AutoConfigurationImportSelector.AutoConfigurationEntry autoConfigurationEntry = this.getAutoConfigurationEntry(autoConfigurationMetadata, annotationMetadata);
            return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
        }
    }
```
点击getAutoConfigurationEntry方法进去，一层层点进去到loadSpringFactories方法。

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryClassName = ((String)entry.getKey()).trim();
                        String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        int var10 = var9.length;

                        for(int var11 = 0; var11 < var10; ++var11) {
                            String factoryName = var9[var11];
                            result.add(factoryClassName, factoryName.trim());
                        }
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var13) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
            }
        }
    }
```
这个方法会将spring.factories文件里的所有类的全限定名读进来。这里先记住导入的AutoConfigurationImportSelector指向selectImport方法会将文件里类的全限定名导入。

然后来看看@AutoConfigurationPackage
# @AutoConfigurationPackage
AutoConfigurationPackage注解的作用是将 添加该注解的类所在的package 作为 自动配置package 进行管理。

看看内部
```java
    
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import({Registrar.class})//导入了一个Registrar包
    public @interface AutoConfigurationPackage {
    }
    //看看AutoConfigurationPackages这个类里的静态内部类Registrar
    static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
        Registrar() {
        }

        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AutoConfigurationPackages.register(registry, (new AutoConfigurationPackages.PackageImport(metadata)).getPackageName());
        }

        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new AutoConfigurationPackages.PackageImport(metadata));
        }
    }
```
可以看到实现了ImportBeanDefinitionRegistrar接口，第一个方法就是用来做Bean注册的，或到了当前注解所修饰的主类的类信息并注册成BeanDefinition


# 观察selectorImport方法在哪里调用

```java
//SpringBoot生命周期的这个方法里
 invokeBeanFactoryPostProcessors(beanFactory);
 //会执行下面的方法
 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
 //方法里面会接着执行
 BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry
 //关注BeanDefinitionRegistryPostProcessor接口的实现类
 ConfigurationClassPostProcessor.processConfigBeanDefinitions(BeanDefinitionRegistry registry)里
```
```java
    @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
          "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
          "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);
 
    processConfigBeanDefinitions(registry);
  }
```
继续点进去
```java
        do {
                parser.parse(candidates);
                parser.validate();
                Set<ConfigurationClass> configClasses = new LinkedHashSet(parser.getConfigurationClasses());
                configClasses.removeAll(alreadyParsed);
                if (this.reader == null) {
                    this.reader = new ConfigurationClassBeanDefinitionReader(registry, this.sourceExtractor, this.resourceLoader, this.environment, this.importBeanNameGenerator, parser.getImportRegistry());
                }

                this.reader.loadBeanDefinitions(configClasses);
                alreadyParsed.addAll(configClasses);
                candidates.clear();
                if (registry.getBeanDefinitionCount() > candidateNames.length) {
                    String[] newCandidateNames = registry.getBeanDefinitionNames();
                    Set<String> oldCandidateNames = new HashSet(Arrays.asList(candidateNames));
                    Set<String> alreadyParsedClasses = new HashSet();
                    Iterator var12 = alreadyParsed.iterator();
```

可以看到parse方法