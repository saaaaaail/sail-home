---
title: Activiti6部分源码解析
date: 2020-08-24 19:47:17
tags:
- Activiti6
---

# 开启流程实例
```java
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(ActivitiConstant.VIDEO_TRANSLATE_PROCESS_ID,
                        translateTask.getTranslatePartnerId().toString(),
                        vars);
```
以startProcessInstanceByKey方法为例，点进去
```java
    public ProcessInstance startProcessInstanceByKey(String processDefinitionKey, String businessKey, Map<String, Object> variables) {
        return (ProcessInstance)this.commandExecutor.execute(new StartProcessInstanceCmd(processDefinitionKey, (String)null, businessKey, variables));
    }
```
可以看到实例化了一个StartProcessInstanceCmd对象，看看如何实例化的。  
在StartProcessInstanceCmd的execute方法中看到从deploymentcache中取得流程定义
```java
public ProcessInstance execute(CommandContext commandContext) {
        DeploymentManager deploymentCache = commandContext.getProcessEngineConfiguration().getDeploymentManager();
        ProcessDefinition processDefinition = null;
        if (this.processDefinitionId != null) {
            processDefinition = deploymentCache.findDeployedProcessDefinitionById(this.processDefinitionId);
            if (processDefinition == null) {
                throw new ActivitiObjectNotFoundException("No process definition found for id = '" + this.processDefinitionId + "'", ProcessDefinition.class);
            }
        } else if (this.processDefinitionKey == null || this.tenantId != null && !"".equals(this.tenantId)) {
            if (this.processDefinitionKey == null || this.tenantId == null || "".equals(this.tenantId)) {
                throw new ActivitiIllegalArgumentException("processDefinitionKey and processDefinitionId are null");
            }

            processDefinition = deploymentCache.findDeployedLatestProcessDefinitionByKeyAndTenantId(this.processDefinitionKey, this.tenantId);
            if (processDefinition == null) {
                throw new ActivitiObjectNotFoundException("No process definition found for key '" + this.processDefinitionKey + "' for tenant identifier " + this.tenantId, ProcessDefinition.class);
            }
        } else {
            processDefinition = deploymentCache.findDeployedLatestProcessDefinitionByKey(this.processDefinitionKey);
            if (processDefinition == null) {
                throw new ActivitiObjectNotFoundException("No process definition found for key '" + this.processDefinitionKey + "'", ProcessDefinition.class);
            }
        }

        this.processInstanceHelper = commandContext.getProcessEngineConfiguration().getProcessInstanceHelper();
        ProcessInstance processInstance = this.createAndStartProcessInstance(processDefinition, this.businessKey, this.processInstanceName, this.variables, this.transientVariables);
        return processInstance;
    }
```
然后会去调用findDeployedLatestProcessDefinitionByKey方法
```java
    public ProcessDefinition findDeployedLatestProcessDefinitionByKey(String processDefinitionKey) {
        ProcessDefinition processDefinition = this.processDefinitionEntityManager.findLatestProcessDefinitionByKey(processDefinitionKey);
        if (processDefinition == null) {
            throw new ActivitiObjectNotFoundException("no processes deployed with key '" + processDefinitionKey + "'", ProcessDefinition.class);
        } else {
            ProcessDefinition processDefinition = this.resolveProcessDefinition(processDefinition).getProcessDefinition();
            return processDefinition;
        }
    }
```
上面的findLatestProcessDefinitionByKey方法最终会去执行sql，查询表ACT_RE_PROCDEF
```XML
<select id="selectLatestProcessDefinitionByKey" parameterType="string" resultMap="processDefinitionResultMap">
    select *
    from ${prefix}ACT_RE_PROCDEF 
    where KEY_ = #{key} and
          (TENANT_ID_ = ''  or TENANT_ID_ is null) and
          VERSION_ = (select max(VERSION_) from ${prefix}ACT_RE_PROCDEF where KEY_ = #{processDefinitionKey} and (TENANT_ID_ = '' or TENANT_ID_ is null))
  </select>
```
下面的resolveProcessDefinition则会去继续解析流程定义ACT_GE_BYTEARRAY、ACT_RE_DEPLOYMENT。
```java
   public ProcessDefinitionCacheEntry resolveProcessDefinition(ProcessDefinition processDefinition) {
        String processDefinitionId = processDefinition.getId();
        String deploymentId = processDefinition.getDeploymentId();
        ProcessDefinitionCacheEntry cachedProcessDefinition = (ProcessDefinitionCacheEntry)this.processDefinitionCache.get(processDefinitionId);
        if (cachedProcessDefinition == null) {
            CommandContext commandContext = Context.getCommandContext();
            if (commandContext.getProcessEngineConfiguration().isActiviti5CompatibilityEnabled() && Activiti5Util.isActiviti5ProcessDefinition(Context.getCommandContext(), processDefinition)) {
                return Activiti5Util.getActiviti5CompatibilityHandler().resolveProcessDefinition(processDefinition);
            }

            DeploymentEntity deployment = (DeploymentEntity)this.deploymentEntityManager.findById(deploymentId);
            deployment.setNew(false);
            this.deploy(deployment, (Map)null);
            cachedProcessDefinition = (ProcessDefinitionCacheEntry)this.processDefinitionCache.get(processDefinitionId);
            if (cachedProcessDefinition == null) {
                throw new ActivitiException("deployment '" + deploymentId + "' didn't put process definition '" + processDefinitionId + "' in the cache");
            }
        }

        return cachedProcessDefinition;
    }
```
如果cachedProcessDefinition为null，则会执行this.deploy(deployment, (Map)null)方法，此方法里会完成对xml的解析保存工作。


# 查询任务


# 自动创建表
配置文件中配置
```
# 是否自动部署
spring.activiti.database-schema-update=true
```
配置ProcessEngineConfiguration.setDatabaseSchemaUpdate("true")
那么当创建ProcessEngine的时候就会自动创建表

设置寻找bpmn图的位置以及图的名称
```
spring.activiti.process-definition-location-prefix=classpath:/bpmn/
spring.activiti.process-definition-location-suffixes=**.bpmn
```


# 事务
在引擎的AutoConfiguration包里面的processEngineConfigurationBean方法中注入了事务管理器与dataSource。
```java
    public SpringProcessEngineConfiguration processEngineConfigurationBean(Resource[] processDefinitions, DataSource dataSource, PlatformTransactionManager transactionManager, SpringAsyncExecutor springAsyncExecutor) throws IOException {
        SpringProcessEngineConfiguration engine = new SpringProcessEngineConfiguration();
        if (processDefinitions != null && processDefinitions.length > 0) {
            engine.setDeploymentResources(processDefinitions);
        }

        engine.setDataSource(dataSource);
        engine.setTransactionManager(transactionManager);
        if (null != springAsyncExecutor) {
            engine.setAsyncExecutor(springAsyncExecutor);
        }

        return engine;
    }
```
在数据源的AutoConfiguration包
```java
        @Bean
        @ConditionalOnMissingBean
        public PlatformTransactionManager transactionManager(DataSource dataSource) {
            return new DataSourceTransactionManager(dataSource);
        }
```
可以看到生成的事务管理器为DataSourceTransactionManager，与SpringBoot是相同的，而且使用了@ConditionalOnMissingBean注解，保证与SpringBoot是只有一个这个事务管理器。
 
 数据源在源码里没找到，猜想是spring boot jdbc的数据源。


 # 分布式环境下Activiti6主键问题
 因为Activiti6的主键是自己维护的，通过IdGenerator来生成的，通过对getNextId加锁来获得下一个主键id。

 单机可以，分布式就可能会出错，因此：
 1. 使用redis来自增获得主键，不同主机获得的也不一样
 2. 加分布式锁

# 关于视频字幕翻译系统的难点
1. 翻译公司要求能批量添加翻译任务，每一个任务都要使用activiti6开启流程实例，然后获得流程实例id与对应翻译任务关联。但是activiti没有提供一个批量的api，我这边如果一次批量创建了太多流程实例接口就特别的慢，然后将整个开启流程、更新翻译任务表、完成第一个任务节点的操作异步去做了。然后开一个定时任务扫描最近10分钟的失败的，重新开一个流程实例挂到对应翻译任务上。
2. 字幕条数可能在原始字幕校对、翻译、审核、字幕制作阶段变化，之所以要顺序Id，是最后解析成文件的时候，通过orderId顺序来生成的，就是只做关联操作，新增的字幕做插入，更新的字幕做修改，在字幕制作结束以后，根据时间轴