---
title: springboot-springsession-redis分析
date: 2020-05-17 22:57:44
tags: 
- spring-boot
- spring-session
- redis
---
# 先总结
1、spring session redis的核心类为RedisOperationsSessionRepository与SessionRepositoryFilter。
2、一个请求打进来会经过SessionRepositoryFilter，这个Filter会将Request包装为SessionRepositoryRequestWrapper，然后重写了getSession方法，要知道获得Session的唯一方法就是这个方法，这个方法会从本地缓存里面取，然后从Cookie里找sessionId然后从redis里面取（同时更新最近访问时间），都没有就新创建一个新的Session。
3、Session的保存在一次请求结束的时候使用CommitSession保存到Redis里面，并且由于response的缓冲区有可能通过flush发方法随时被清空，因此response继承OnCommittedResponseWrapper重写了onResponseCommitted的方法保证session能被写入redis。
4、被写入的session包含三个key值，key=expired:sessionId，value=无记录 这个键在Redis中的过期时间就是当前session的过期时间 key=sessionId，value=包含session的全部信息，过期时间间隔、最近访问时间、一些attributes等等。还一个key = expiration，value=当前时间节点刚刚过期1分钟的sessionId集合。
5、关于登出操作，session.invalidate()会删除key=expired:sessionId，以及expiration集合中刚刚过期的sessionId，关于sessionId的删除是将session有效期设为0，惰性删除。

> 使用分布式Session的作用是什么？
> 因为服务多机部署，当前端访问后端的时候，请求到了不同的主机上面，能根据cookie里的sessionId获得用户的登录状态

# spring boot+spring session+redis实现session共享

## 为什么要实现session共享

因为目前目前要实现一个校验内核，将登录、鉴权以及其他一些通用操作抽离出来复用。这样一个内核使用起来肯定会搭成分布式结构，部署多个机房多个主机，通过nginx来进行负载均衡。如果不进行session共享，那么session是由tomcat容器管理的，当两次请求分别打进了不同的主机，第二次请求cookie中携带了sessionId，要么sessionId无效重新登录，要么跳过登录使用session时报错，总之体验不好。

- 传统Session:request进入web容器，根据request获取Session，如果web容器里面存在Session则返回，如果不存在Session则新创建一个Session放到header或者header的cookie里面，tomcat的SessionId为jsessionid。
- Spring Session:request进入web容器，根据request获取Session，首先会从请求头中获得Session，如果不为空则直接返回，如果请求头里的Session为空，则从cookie里面获取sessionId不为空，则拿着这个sessionId从redis获得session，这个session不为空的话保存这个session到request的header里，并返回这个session。如果从cookie里取得sessionId为空或者redis取出的session为空，就得重新创建一个session，保存一些session参数，并保存到request的header中，返回这个session。

## Spring Session + Redis快速搭建操作
   
### springboot方式
1. 依赖包如下:
```
        <!-- Spring Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-redis</artifactId>
        </dependency>
        <!-- Spring session -->
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
            <version>2.1.4.RELEASE</version>
        </dependency>
```

2. 使用@EnableRedisHttpSession注解开启使用Spring Session
3. redis配置不作介绍
   

## Spring Session原理分析

### @EnableRedisHttpSession注解

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({RedisHttpSessionConfiguration.class})
@Configuration
public @interface EnableRedisHttpSession {
    int maxInactiveIntervalInSeconds() default 1800;

    String redisNamespace() default "spring:session";

    RedisFlushMode redisFlushMode() default RedisFlushMode.ON_SAVE;

    String cleanupCron() default "0 * * * * *";
}
```
由注解可见，导入了RedisHttpSessionConfiguration类

### RedisHttpSessionConfiguration类分析

```
@Configuration
@EnableScheduling
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration implements BeanClassLoaderAware, EmbeddedValueResolverAware, ImportAware, SchedulingConfigurer {
```
该类是一个配置类，父类为SpringHttpSessionConfiguration 稍后分析，实现了EmbeddedValueResolverAware, ImportAware接口

该配置类获得了从父类SpringHttpSessionConfiguration中创建的SessionRepositoryFilter的Bean类
```
    @Bean
    public <S extends Session> SessionRepositoryFilter<? extends Session> springSessionRepositoryFilter(SessionRepository<S> sessionRepository) {
        SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter(sessionRepository);
        sessionRepositoryFilter.setServletContext(this.servletContext);
        sessionRepositoryFilter.setHttpSessionIdResolver(this.httpSessionIdResolver);
        return sessionRepositoryFilter;
    }
```
创建filter需要SessionRepository的Bean，这个类的创建在RedisHttpSessionConfiguration中，由于Spring Session有三种支持分别是redis、mongoDB、jdbc，此处创建的就是redis的sessionRepository类RedisOperationsSessionRepository如下：

```
      @Bean
    public RedisOperationsSessionRepository sessionRepository() {
        RedisTemplate<Object, Object> redisTemplate = this.createRedisTemplate();
        RedisOperationsSessionRepository sessionRepository = new RedisOperationsSessionRepository(redisTemplate);
        sessionRepository.setApplicationEventPublisher(this.applicationEventPublisher);
        if (this.defaultRedisSerializer != null) {
            sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
        }

        sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
        if (StringUtils.hasText(this.redisNamespace)) {
            sessionRepository.setRedisKeyNamespace(this.redisNamespace);
        }

        sessionRepository.setRedisFlushMode(this.redisFlushMode);
        int database = this.resolveDatabase();
        sessionRepository.setDatabase(database);
        return sessionRepository;
    }
```
由此可见，对于session的管理主要是由SessionRepository和SessionRepositoryFilter完成的。

### SessionRepositoryFilter
@Order(-2147483598)表明这个filter的优先级非常高

```
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);
        SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryFilter.SessionRepositoryRequestWrapper(request, response, this.servletContext);
        SessionRepositoryFilter.SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryFilter.SessionRepositoryResponseWrapper(wrappedRequest, response);

        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            wrappedRequest.commitSession();
        }

    }
```
对request和response进行包装，然后向下一个filter传递，在此次调用链结束之前会调用commitSession

### SessionRepositoryRequestWrapper extends HttpServletRequestWrapper
#### commitSession
commitSession是SessionRepositoryRequestWrapper具有的方法。
```
        private void commitSession() {
            /**
            * 从当前request的header中获得Session的包装类
            */
            SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper wrappedSession = this.getCurrentSession();
            /**
            * 如果header中保存的session为空
            */
            if (wrappedSession == null) {
                /**
                * 判断标志位有没有失效，requestedSessionInvalidated标志位只有在session调invalidate方法后才会设为true表示失效
                */
                if (this.isInvalidateClientSession()) {
                    SessionRepositoryFilter.this.httpSessionIdResolver.expireSession(this, this.response);
                }
            } else {
                /**
                * 如果请求header中的Session不为空，则从HttpSessionWrapper中取得ExpiringSession，调用sessionRepository的save方法保存到redis
                */
                S session = wrappedSession.getSession();

                this.clearRequestedSessionCache();

                SessionRepositoryFilter.this.sessionRepository.save(session);

                String sessionId = session.getId();

                /**
                * 判断请求的sessionId是否有效（这里面的标志位requestedSessionIdValid在getSession中被修改为true表示这个sessionId有效，如果requestedSessionIdValid为null会从cookie中取出sessionId从redis中取session不为空就返true）  或者  判断这个当前请求header里的session的id是否与cookie里保存的sessionid相同（getRequestedSessionId()是从cookie中获得当前请求的session），如果不同就执行里面的方法。
                */
                if (!this.isRequestedSessionIdValid() || !sessionId.equals(this.getRequestedSessionId())) {
                    //这里就是把session的id添加到response的cookie里
                    SessionRepositoryFilter.this.httpSessionIdResolver.setSessionId(this, this.response, sessionId);
                }
            }

        }

        /**
        * 从request的header取得当前session对象HttpSessionWrapper
        */
        private SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper getCurrentSession() {
            return (SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper)this.getAttribute(SessionRepositoryFilter.CURRENT_SESSION_ATTR);
        }

        /**
        * 此方法判断session失效，1是当前session为空，2是requestedSessionInvalidated为true，这个状态是在HttpSessionWrapper里的invalidate()方法里置true的表示session已失效
        */
        private boolean isInvalidateClientSession() {
            return this.getCurrentSession() == null && this.requestedSessionInvalidated;
        }

        /**
        * 该方法是往response中清空cookie
        */
        public void expireSession(HttpServletRequest request, HttpServletResponse response) {
            this.cookieSerializer.writeCookieValue(new CookieValue(request, response, ""));
        }

        /**
        * 清空请求session，包括已缓存session标志位、requestSession与requestSessionId置空，这几个值都在getRequestedSession()方法中赋值
        */
        private void clearRequestedSessionCache() {
            this.requestedSessionCached = false;
            this.requestedSession = null;
            this.requestedSessionId = null;
        }
        /**
        * 更新session的一下参数，如果是新的会话则入redis保存，同时将session置旧
        */
        public void save(RedisOperationsSessionRepository.RedisSession session) {
            session.saveDelta();
            if (session.isNew()) {
                String sessionCreatedKey = this.getSessionCreatedChannel(session.getId());
                this.sessionRedisOperations.convertAndSend(sessionCreatedKey, session.delta);
                session.setNew(false);
            }

        }

        /**
        * 这个方法也比较清晰，如果当前请求header里的session的id与WRITTEN_SESSION_ID_ATTR的值不同的话就保存这个sessionId，并写到cookie中
        */
        public void setSessionId(HttpServletRequest request, HttpServletResponse response, String sessionId) {
            if (!sessionId.equals(request.getAttribute(WRITTEN_SESSION_ID_ATTR))) {
                request.setAttribute(WRITTEN_SESSION_ID_ATTR, sessionId);
                this.cookieSerializer.writeCookieValue(new CookieValue(request, response, sessionId));
            }
        }
```

#### getSession
还有一个方法同样重要就是SessionRepositoryRequestWrapper重写的HttpServletRequestWrapper类的getSession方法，通过这个方法

```
        public SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper getSession(boolean create) {
            /**
            * 从request的header中获得Session
            */
            SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper currentSession = this.getCurrentSession();
            if (currentSession != null) {
                return currentSession;
            } else {
                /**
                * 如果requestedSessionCached没有被缓存过为false，则会从cookie中找sessionId并从redis找到session并返回。
                */
                S requestedSession = this.getRequestedSession();
                if (requestedSession != null) {
                    if (this.getAttribute(SessionRepositoryFilter.INVALID_SESSION_ID_ATTR) == null) {
                        //设置session最近访问的时间
                        requestedSession.setLastAccessedTime(Instant.now());
                        //设置session有效
                        this.requestedSessionIdValid = true;
                        //包装ExpiringSession为HttpSessionWrapper
                        currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(requestedSession, this.getServletContext());
                        currentSession.setNew(false);
                        //保存session到header里面，并返回
                        this.setCurrentSession(currentSession);
                        return currentSession;
                    }
                } else {
                    if (SessionRepositoryFilter.SESSION_LOGGER.isDebugEnabled()) {
                        SessionRepositoryFilter.SESSION_LOGGER.debug("No session found by id: Caching result for getSession(false) for this HttpServletRequest.");
                    }
                    //设置header头标志位session无效为true
                    this.setAttribute(SessionRepositoryFilter.INVALID_SESSION_ID_ATTR, "true");
                }

                /**
                * 上面的操作均未取得session，认为session不存在，create=true就新创建session
                */
                if (!create) {
                    return null;
                } else {
                    if (SessionRepositoryFilter.SESSION_LOGGER.isDebugEnabled()) {
                        SessionRepositoryFilter.SESSION_LOGGER.debug("A new session was created. To help you troubleshoot where the session was created we provided a StackTrace (this is not an error). You can prevent this from appearing by disabling DEBUG logging for " + SessionRepositoryFilter.SESSION_LOGGER_NAME, new RuntimeException("For debugging purposes only (not an error)"));
                    }
                    //通过sessionRepository创建ExpiringSession
                    S session = SessionRepositoryFilter.this.sessionRepository.createSession();
                    //设置session的最近访问时间
                    session.setLastAccessedTime(Instant.now());
                    //包装ExpiringSession为HttpSessionWrapper
                    currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(session, this.getServletContext());
                    //保存新创建的session到header中，并返回这个session
                    this.setCurrentSession(currentSession);
                    return currentSession;
                }
            }
        }

        private S getRequestedSession() {
            /**
            * 这个标志位表示session是否被缓存，为false表示没有缓存的session，进入语句块
            */
            if (!this.requestedSessionCached) {
                /**
                * 这个方法从cookie列表中找到当前session名称的sessionId列表
                */
                List<String> sessionIds = SessionRepositoryFilter.this.httpSessionIdResolver.resolveSessionIds(this);
                Iterator var2 = sessionIds.iterator();

                /**
                * 循环遍历sessionId列表
                */
                while(var2.hasNext()) {
                    /**
                    * 如果暂存的requestedSessionId为空就先将requestedSessionId置为sessionId
                    */
                    String sessionId = (String)var2.next();
                    if (this.requestedSessionId == null) {
                        this.requestedSessionId = sessionId;
                    }
                    /**
                    * 从redis中查找这个sessionId的session，就暂存并跳出循环，返回session，如果没找到session，保存的就是列表中第一个sessionId，并返回空session
                    */
                    S session = SessionRepositoryFilter.this.sessionRepository.findById(sessionId);
                    if (session != null) {
                        this.requestedSession = session;
                        this.requestedSessionId = sessionId;
                        break;
                    }
                }
                
                this.requestedSessionCached = true;
            }

            return this.requestedSession;
        }

        /**
        * 这个方法从cookie列表中找到当前session名称的sessionId列表
        */
        public List<String> resolveSessionIds(HttpServletRequest request) {
            return this.cookieSerializer.readCookieValues(request);
        }
```

### SessionRepositoryResponseWrapper extends OnCommittedResponseWrapper

```
  private final class SessionRepositoryResponseWrapper extends OnCommittedResponseWrapper {
        private final SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper request;

        SessionRepositoryResponseWrapper(SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper request, HttpServletResponse response) {
            super(response);
            if (request == null) {
                throw new IllegalArgumentException("request cannot be null");
            } else {
                this.request = request;
            }
        }

        protected void onResponseCommitted() {
            this.request.commitSession();
        }
    }
```
首先要知道response对象存在一个outputBuffer，当某些操作导致response对象被提交的话，会强制将缓冲区的东西发送给客户端并清理缓冲区。

导致response提交的操作:
> 1、达到默认最大buffer size  
> 2、调用HttpServletResponse.flushBuffer()   
> 3、调用HttpServletResponse.getOutputStream().flush()或者HttpServletResponse.getWriter().flush()  
> 4、调用HttpServletResponse.sendError()或者HttpServletResponse.sendRedirect()  

继承了OnCommittedResponseWrapper类，这个类会重写这些方法，在清空缓冲区前执行onResponseCommitted()方法，此处就是保证response对象无论何时提交，Spring Session对象都能执行commitSession操作。

### RedisOperationsSessionRepository

sessionRespository根据session存储的不同分为很多种sessionRepository操作仓库，重点关注下RedisOperationsSessionRepository的源码。

首先需要了解redisSession保存到redis中的几个关键的key-value:
- key【spring:session:sessions:sessionId】+ value【保存是这个sessionId对应的session的详细信息，包括Session的过期时间间隔、最近的访问时间、attributes等等】这个kv的过期时间是session最大过期时间+5分钟
- key【spring:session:sessions:expires:sessionId】+ value【不保存有效信息】这个kv在redis中的过期时间就是session的过期时间
- key【spring:session:expirations:expiration】+value【是一个set结构，保存的是expiration时间戳的sessionid的集合】这个key里面的expiration表示的是session过期时间滚动到下一分钟的时间戳，意思就是在到达这个时间点时，集合里面的sessionId刚刚过期小于1分钟

```
    /**
    * 如下是RedisSession的关键属性
    */
    final class RedisSession implements Session {
        //MapSession是RedisSession的本地缓存，从redis中取出session后将session属性保存在MapSession中，通过getAttribute获得属性
        private final MapSession cached;
        //
        private Instant originalLastAccessTime;
        //delta数据用于跟踪数据的变化，持久化到redis中
        private Map<String, Object> delta;
        private boolean isNew;
        private String originalPrincipalName;
        private String originalSessionId;
```

```
    public RedisOperationsSessionRepository.RedisSession createSession() {
        RedisOperationsSessionRepository.RedisSession redisSession = new RedisOperationsSessionRepository.RedisSession();
        if (this.defaultMaxInactiveInterval != null) {
            redisSession.setMaxInactiveInterval(Duration.ofSeconds((long)this.defaultMaxInactiveInterval));
        }

        return redisSession;
    }
```
首先是createSession调用RedisSession构造方法创建一个session。

```
    public void save(RedisOperationsSessionRepository.RedisSession session) {
        //调用redisSession的saveSetla持久化session
        session.saveDelta();
        if (session.isNew()) {
            //如果session为新创建的则发布一个session创建的事件
            String sessionCreatedKey = this.getSessionCreatedChannel(session.getId());
            this.sessionRedisOperations.convertAndSend(sessionCreatedKey, session.delta);
            session.setNew(false);
        }

    }

    /**
    * redisSession的save方法，用于将变化的属性保存到delta中，并持久化到redis
    */
    private void saveDelta() {
        String sessionId = this.getId();
        /**
        * sessionId
        */
        this.saveChangeSessionId(sessionId);
        /**
        * delta为空，表示没有改变的数据
        */
        if (!this.delta.isEmpty()) {
            RedisOperationsSessionRepository.this.getSessionBoundHashOperations(sessionId).putAll(this.delta);
            String principalSessionKey = RedisOperationsSessionRepository.getSessionAttrNameKey(FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME);
            String securityPrincipalSessionKey = RedisOperationsSessionRepository.getSessionAttrNameKey("SPRING_SECURITY_CONTEXT");
            if (this.delta.containsKey(principalSessionKey) || this.delta.containsKey(securityPrincipalSessionKey)) {
                String principal;
                if (this.originalPrincipalName != null) {
                    principal = RedisOperationsSessionRepository.this.getPrincipalKey(this.originalPrincipalName);
                    RedisOperationsSessionRepository.this.sessionRedisOperations.boundSetOps(principal).remove(new Object[]{sessionId});
                }

                principal = RedisOperationsSessionRepository.PRINCIPAL_NAME_RESOLVER.resolvePrincipal(this);
                this.originalPrincipalName = principal;
                if (principal != null) {
                    String principalRedisKey = RedisOperationsSessionRepository.this.getPrincipalKey(principal);
                    RedisOperationsSessionRepository.this.sessionRedisOperations.boundSetOps(principalRedisKey).add(new Object[]{sessionId});
                }
            }

            /**
            * 清空delta，表示delta里的数据已经入库了
            */
            this.delta = new HashMap(this.delta.size());
            /**
            * 更新过期时间，时间是依次下沿至下一时间间隔
            */
            Long originalExpiration = this.originalLastAccessTime != null ? this.originalLastAccessTime.plus(this.getMaxInactiveInterval()).toEpochMilli() : null;
            /**
            * 更新session的三个key
            */
            RedisOperationsSessionRepository.this.expirationPolicy.onExpirationUpdated(originalExpiration, this);
        }
    }
```
save方法完成redisSession的持久化操作。

```
    private RedisOperationsSessionRepository.RedisSession getSession(String id, boolean allowExpired) {
        /**
        * 根据sessionid从redis中取出map结构，保存的是这个sessionId对应的session的参数，是一个map结构
        */
        Map<Object, Object> entries = this.getSessionBoundHashOperations(id).entries();
        if (entries.isEmpty()) {
            return null;
        } else {
            /**
            * 这个方法下面有介绍，就是将entries转为mapSession结构
            */
            MapSession loaded = this.loadSession(id, entries);
            /**
            * 如果不允许session过期allowExpired=false且mapSession对应的redisSession没有过期
            */
            if (!allowExpired && loaded.isExpired()) {
                return null;
            } else {
                //将mapSession转成redisSession
                RedisOperationsSessionRepository.RedisSession result = new RedisOperationsSessionRepository.RedisSession(loaded);
                result.originalLastAccessTime = loaded.getLastAccessedTime();
                return result;
            }
        }
    }

    private MapSession loadSession(String id, Map<Object, Object> entries) {
        MapSession loaded = new MapSession(id);
        Iterator var4 = entries.entrySet().iterator();

        //找到session的参数填入到MapSession中
        while(var4.hasNext()) {
            Entry<Object, Object> entry = (Entry)var4.next();
            String key = (String)entry.getKey();
            if ("creationTime".equals(key)) {
                loaded.setCreationTime(Instant.ofEpochMilli((Long)entry.getValue()));
            } else if ("maxInactiveInterval".equals(key)) {
                loaded.setMaxInactiveInterval(Duration.ofSeconds((long)(Integer)entry.getValue()));
            } else if ("lastAccessedTime".equals(key)) {
                loaded.setLastAccessedTime(Instant.ofEpochMilli((Long)entry.getValue()));
            } else if (key.startsWith("sessionAttr:")) {
                loaded.setAttribute(key.substring("sessionAttr:".length()), entry.getValue());
            }
        }

        return loaded;
    }

    /**
    * 考虑一下其过期方法，即查找的时候会判断mapSession有没有过期，当前时间减去失效时间小于上一次的访问时间，认为session有效
    */
    public boolean isExpired() {
        return this.isExpired(Instant.now());
    }

    boolean isExpired(Instant now) {
        if (this.maxInactiveInterval.isNegative()) {
            return false;
        } else {
            return now.minus(this.maxInactiveInterval).compareTo(this.lastAccessedTime) >= 0;
        }
    }
```
然后getSession方法根据sessionId获得RedisSession。

```
    public void deleteById(String sessionId) {
        /**
        * 根据sessionId获得session，allowExpore参数为true，表示过期了就不创建新的了
        */
        RedisOperationsSessionRepository.RedisSession session = this.getSession(sessionId, true);
        if (session != null) {
            //清除当前session数据的索引
            this.cleanupPrincipalIndex(session);
            //这个onDelete方法删除的是【spring:session:expirations:expiration】这个key的过期时间集合里面删除sessionId
            this.expirationPolicy.onDelete(session);
            //这个删除的是【spring:session:sessions:expires:sessionId】key 这个是表示sessionId的过期时间的key
            String expireKey = this.getExpiredKey(session.getId());
            this.sessionRedisOperations.delete(expireKey);
            //将session有效期设置为0，可见删除方法并没有直接从redis中删除session信息的key
            session.setMaxInactiveInterval(Duration.ZERO);
            this.save(session);
        }
    }
```
根据sessionId删除session，首先删除了过期时间集合中的sessionId，以及与session过期时间相同的redis的key值，然后将session的有效期设置为0，即 使这个session失效而不是从redis中删除，而是更新其在redis中的失效时间并保存，此处采用的是惰性删除，让其自然过期且无法访问，对内存不友好，对cpu友好。

### spring session中的事件传播机制 redis向
Spring Session中事件传播的实现如下:

![spring session事件传播流程图](图1.png)

Spring Session的事件类型： 
- session创建事件
- session过期事件
- session删除事件

首先Spring Session事件的顶层是ApplicationEvent 是基于Spring的事件传播机制的，RedisOperationsSessionRepository负责session的出入库，必然是SessionEvent的事件发布者，持有事件发布者对象ApplicationEventPublisher，在RedisHttpSessionConfiguration中创建Bean的时候就会注入上下文中的applicationEventPublisher Bean对象。对于事件的监听则由开发者自己实现:
```
@Component
public class SessionEventListener implements ApplicationListener<SessionCreatedEvent> {

    @Override
    public void onApplicationEvent(SessionCreatedEvent event) {
        //当session创建的时候doSomething
    }
}
```

由上图可知，Spring Session发布何种事件是由redis的键空间通知机制触发的。

RedisOperationsSessionRepository实现spring-data-redis中的MessageListener接口。

```

public class RedisOperationsSessionRepository implements FindByIndexNameSessionRepository<RedisOperationsSessionRepository.RedisSession>, MessageListener {

    /**
    * 这个方法在RedisOperationsSessionRepository构造方法中就会触发，设置监听redis事件的sessionChannel
    */
    private void configureSessionChannels() {
        this.sessionCreatedChannelPrefix = this.namespace + "event:" + this.database + ":created:";
        this.sessionDeletedChannel = "__keyevent@" + this.database + "__:del";
        this.sessionExpiredChannel = "__keyevent@" + this.database + "__:expired";
    }

    /**
    * 用于监听redis的键空间通知并发布spring的事件
    */
    public void onMessage(Message message, byte[] pattern) {
        byte[] messageChannel = message.getChannel();
        byte[] messageBody = message.getBody();
        String channel = new String(messageChannel);
        if (channel.startsWith(this.sessionCreatedChannelPrefix)) {
            Map<Object, Object> loaded = (Map)this.defaultSerializer.deserialize(message.getBody());
            this.handleCreated(loaded, channel);
        } else {
            String body = new String(messageBody);
            if (body.startsWith(this.getExpiredKeyPrefix())) {
                boolean isDeleted = channel.equals(this.sessionDeletedChannel);
                if (isDeleted || channel.equals(this.sessionExpiredChannel)) {
                    int beginIndex = body.lastIndexOf(":") + 1;
                    int endIndex = body.length();
                    String sessionId = body.substring(beginIndex, endIndex);
                    RedisOperationsSessionRepository.RedisSession session = this.getSession(sessionId, true);
                    if (session == null) {
                        logger.warn("Unable to publish SessionDestroyedEvent for session " + sessionId);
                        return;
                    }

                    if (logger.isDebugEnabled()) {
                        logger.debug("Publishing SessionDestroyedEvent for session " + sessionId);
                    }

                    this.cleanupPrincipalIndex(session);
                    if (isDeleted) {
                        this.handleDeleted(session);
                    } else {
                        this.handleExpired(session);
                    }
                }

            }
        }
    }
}
```