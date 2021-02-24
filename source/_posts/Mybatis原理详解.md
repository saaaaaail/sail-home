---
title: Mybatis原理详解
date: 2020-08-26 22:02:01
tags:
- Mybaits
---
# MybatisAutoConfiguration
首先这个自动配置类包含注解 @AutoConfigureAfter({DataSourceAutoConfiguration.class}) 表示当前类要在这个类之后加载，假设数据源已经加载好了。

然后通过SqlSessionFactoryBean生成SqlSessionFactory并注入
```java
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setVfs(SpringBootVFS.class);
        if (StringUtils.hasText(this.properties.getConfigLocation())) {
            factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
        }

        org.apache.ibatis.session.Configuration configuration = this.properties.getConfiguration();
        if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
            configuration = new org.apache.ibatis.session.Configuration();
        }

        if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
            Iterator var4 = this.configurationCustomizers.iterator();

            while(var4.hasNext()) {
                ConfigurationCustomizer customizer = (ConfigurationCustomizer)var4.next();
                customizer.customize(configuration);
            }
        }

        factory.setConfiguration(configuration);
        if (this.properties.getConfigurationProperties() != null) {
            factory.setConfigurationProperties(this.properties.getConfigurationProperties());
        }

        if (!ObjectUtils.isEmpty(this.interceptors)) {
            factory.setPlugins(this.interceptors);
        }

        if (this.databaseIdProvider != null) {
            factory.setDatabaseIdProvider(this.databaseIdProvider);
        }

        if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
            factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
        }

        if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
            factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
        }

        if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
            factory.setMapperLocations(this.properties.resolveMapperLocations());
        }

        return factory.getObject();
    }
```
通过SqlSessionFactory生成sqlSessionTemplate
```java
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        return executorType != null ? new SqlSessionTemplate(sqlSessionFactory, executorType) : new SqlSessionTemplate(sqlSessionFactory);
    }
```
上面已经取得了SqlSessionFactory对象，那么只需要openSession就能获得一个SqlSession对象DefaultSqlSession。

# Mapper对象咋来的
首先Mybatis将我们的所有配置解析为Configuration对象。
首先看DefaultSqlSession的getMapper方法，我们都知道，通过sqlSession.getMapper。
```java
@Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```
configuration对象是我们在加载DefaultSqlSessionFactory时传入的。

然后看下这个的getMapper方法，通过mapperRegistry。
```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
关于mapperRegistry里面有什么
```java
public class MapperRegistry {
    private final Configuration config;
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap();

    public MapperRegistry(Configuration config) {
        this.config = config;
    }
    ....
}
```
有Configuration对象与一个Map，包含一个类型键与Mapper代理工厂。
```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
可以看到一个return了，返回了mapperProxyFactory生成的一个实例对象。往下看。
```java
 public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```
通过这个方式可以看出来mapperProxyFactory.newInstance里的创建了一个MapperProxy对象。后面的newInstance先不管，首先肯定返回出去的Mapper对象是一个Mapper的代理对象。

既然是Mapper代理对象，看一下MapperProxy这个类，实现了InvocationHandler接口，并重写了invoke方法。
```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```
上面的方法。  
1. 从代码中我们可以看到前面做了一个判断，这个判断主要是保证我们调用像toString方法或者equals方法时也能正常调用。  
2. 然后我们可以看到它调用cachedMapperMethod返回MapperMethod对象，这个cachedMapperMethod方法主要是能缓存我们使用过的一些mapperMethod对象，方便下次使用。
3. 这个MapperMethod对象主要是获取方法对应的sql命令和执行相应SQL操作等的处理。

接着往execute方法。
```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
这里的Command是哪来的？




# Mybatis预编译
使用jdbc的preparedStatement来预编译语句。
使用#{}与${}的时候，Mybatis动态sql解析的时候，解析成BoundSql对象的时候#{}会被解析成?占位符，得到一串没有参数的sql语句。

即使没有开启预编译，preparedStatement
preparedStatement提供了一个setString的方法，能对字符串里面的部分符号做转义处理。

如果开启了预编译，那么一条语句会与Mysql进行两次传输，第一次传输?占位符的sql语句进行预编译，第二次传递完整转义的sql语句执行。

一条sql语句执行包括
- 解析阶段
    - 主要做各种校验
- 编译阶段
- 优化阶段
    - 编译优化结束以后，语句执行逻辑就确定了
- 缓存阶段
- 执行阶段

所以在set参数的时候，preparedStatement就已经完成了编译的过程，后续只是将数据进行替换，参数中的执行逻辑是不会被解析的。

所以预编译的优点就是性能优势，sql语句的分析、编译、优化，在第一次传输时已经完成，后续如果是相同的sql语句直接执行就可以。

# 什么情况下不能使用预编译


# 数据库连接池参数
1. maxActive 连接池支持的最大连接数，这里取值为20，表示同时最多有20个数据库连接。一般把maxActive设置成可能的并发量就行了设 0 为没有限制。

2. maxIdle 连接池中最多可空闲maxIdle个连接 ，这里取值为20，表示即使没有数据库连接时依然可以保持20空闲的连接，而不被清除，随时处于待命状态。设 0 为没有限制。

3. minIdle 连接池中最小空闲连接数，当连接数少于此值时，连接池会创建连接来补充到该值的数量

4. initialSize 初始化连接数目

5. maxWait 连接池中连接用完时,新的请求等待时间,毫秒，这里取值-1，表示无限等待，直到超时为止，也可取值9000，表示9秒后超时。超过时间会出错误信息