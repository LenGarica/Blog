# 基于Ruoyi-vue实现某些需求

## 一、Redis实现文章自动保存

我们将文章设置为map结构。

### 1. 定义RedisUtil方法

```java
public class RedisUtil{
    @Autowired
    private RedisTemplate redisTemplate;
    /**
     * Hash 存储 map 实现多个键值保存并设置时间
     * @param key 键
     * @param map 对应多个键值
     * @param time 时间(秒)
     * @return true成功 false失败
     */
    public boolean hmset(String key, Map<String,Object> map, long time){
        try {
            redisTemplate.opsForHash().putAll(key, map);
            if(time>0){
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 获取hashKey对应的所有键值
     * @param key 键
     * @return 对应的多个键值
     */
    public Map<Object,Object> hmget(String key){
        return redisTemplate.opsForHash().entries(key);
    }

    /**
     * 删除hash表中的值
     * @param key 键 不能为null
     * @param item 项 可以使多个 不能为null
     */
    public void hdel(String key, Object... item){
        redisTemplate.opsForHash().delete(key,item);
    }
}
```

### 2. 定义RedisService业务操作

```java
    /**
     * 保存文章
     *
     * @param key
     * @param article 文章
     * @param expireTime 过期时间
     * @return
     */
    boolean saveArticle(String key, ArticlePublishParam article, long expireTime);

    /**
     * 获取文章
     *
     * @param key
     * @return
     */
    ArticlePublishParam getArticle(String key);

    /**
     * 删除文章
     *
     * @param key
     */
    void deleteArticle(String key);
```

### 3. RedisServiceImpl 

```java
@Override
public boolean saveArticle(String key, ArticlePublishParam articlePublishParam, long expireTime) {

    // 1. 首先将文章转为 map
    BeanMap beanMap = BeanMap.create(articlePublishParam);

    // 2. 保存到 redis
    return redisUtil.hmset(key, beanMap, expireTime);
}

@Override
public ArticlePublishParam getArticle(String key) {
    Map<Object, Object> map = redisUtil.hmget(key);

    if (CollectionUtils.isEmpty(map)){
        return null;
    }else {
        return JSON.parseObject(JSON.toJSONString(map), ArticlePublishParam.class);
    }
}

@Override
public void deleteArticle(String key) {
    // 1. 首先获取 Article 类的所有字段名称
    List<String> fieldNameList = getFieldNameList(ArticlePublishParam.class);

    // 2. 删除对应的对象 hash
    redisUtil.hdel(key, fieldNameList.toArray());
}

/**
 * 获取一个类的所有字段名称
 * @param clazz
 * @return
 */
private List<String> getFieldNameList(Class clazz) {
    List<String> fieldNameList = new ArrayList<>();

    // 1. 获取本类字段
    Field[] filed = clazz.getDeclaredFields();
    for(Field fd : filed) {
        String filedName = fd.getName();
        // 将序列化的属性排除
        if (!"serialVersionUID".equals(filedName)) {
            fieldNameList.add(filedName);
        }
    }

    // 2. 获取父类字段
    Class<?> superClazz = clazz.getSuperclass();
    if (superClazz != null) {
        Field[] superFields = superClazz.getDeclaredFields();
        for (Field superField : superFields) {
            String filedName = superField.getName();
            // 将序列化的属性排除
            if (!"serialVersionUID".equals(filedName)) {
                fieldNameList.add(filedName);
            }
        }
    }

    return fieldNameList;
}
```

### 4. controller层

前端每3分钟，或者自定义时间，或者设置光标焦点

```java
/**
 * 自动保存，编辑文章时每隔 3 分钟自动将数据保存到 Redis 中（以防数据丢失）
 *
 * @param param
 * @param principal
 * @return
 */
@PostMapping("/autoSave")
public ReturnResult autoSave(@RequestBody ArticlePublishParam param, Principal principal) {
    if (Objects.isNull(param)) {
        return ReturnResult.error("参数错误");
    }
    if (Objects.isNull(principal)) {
        return ReturnResult.error("当前用户未登录");
    }

    // 1. 获取当前用户 ID
    User currentUser = userService.findUserByUsername(principal.getName());

    // 2. 生成存储的 key
    // key的生成格式——文章自动保存时存储在 Redis 中的 key ,后面 {0} 是用户 ID
    /**
     * 文章自动保存时存储在 Redis 中的 key ,后面 {0} 是用户 ID
     */
    // String AUTO_SAVE_ARTICLE = "auto_save_article::{0}";
    String key = MessageFormat.format(AUTO_SAVE_ARTICLE, currentUser.getId());

    // 3. 保存到 Redis 中, 过期时间为 1 天。此处是文章的参数类 ArticlePublishParam
    boolean flag = redisService.saveArticle(key, param, 24L * 60 * 60 * 1000);
    if (flag) {
        log.info("保存 key=" + key + " 的编辑内容文章到 Redis 中成功！");
        return ReturnResult.success();
    } else {
        return ReturnResult.error("自动保存文章失败");
    }
}

/**
 * 从 Redis 中获取当前登录用户的草稿文章
 *
 * @param principal
 * @return
 */
@GetMapping("/getAutoSaveArticle")
public ReturnResult getAutoSaveArticle(Principal principal) {
    if (Objects.isNull(principal)) {
        return ReturnResult.error("当前用户未登录");
    }

    // 1. 获取当前用户 ID
    User currentUser = userService.findUserByUsername(principal.getName());

    // 2. 生成存储的 key
    String key = MessageFormat.format(AUTO_SAVE_ARTICLE, currentUser.getId());

    // 3. 获取文章信息
    ArticlePublishParam article = redisService.getArticle(key);

    if (article != null && StringUtils.isNotBlank(article.getTagsStr())){
        String[] split = article.getTagsStr().split(",");
        article.setTagStringList(Arrays.asList(split));
    }

    log.info("获取草稿文章 key=" + key + " 的内容为：" + article);
    return ReturnResult.success(article);
} 


// 文章新增或修改成功，则将当前用户在 Redis 中的草稿进行删除
// 生成存储的 key
// MessageFormat是java.text.MessageFormat类
String key = MessageFormat.format(AUTO_SAVE_ARTICLE, currentUser.getId());
redisService.deleteArticle(key);
log.info("删除草稿文章 key=" + key + " 成功！");
```

## 二、RabbitMQ实现延迟消息

### 1. 延时队列的使用场景

那么什么时候需要用延时队列呢？考虑一下以下场景：

1. 订单在十分钟之内未支付则自动取消。
2. 新创建的店铺，如果在十天内都没有上传过商品，则自动发送消息提醒。
3. 账单在一周内未支付，则自动结算。
4. 用户注册成功后，如果三天内没有登陆则进行短信提醒。
5. 用户发起退款，如果三天内没有得到处理则通知相关运营人员。
6. 预定会议后，需要在预定的时间点前十分钟通知各个与会人员参加会议。


这些场景都有一个特点，需要在某个事件发生之后或者之前的指定时间点完成某一项任务，如：发生订单生成事件，在十分钟之后检查该订单支付状态，然后将未支付的订单进行关闭；发生店铺创建事件，十天后检查该店铺上新商品数，然后通知上新数为 0 的商户；发生账单生成事件，检查账单支付状态，然后自动结算未支付的账单；发生新用户注册事件，三天后检查新注册用户的活动数据，然后通知没有任何活动记录的用户；发生退款事件，在三天之后检查该订单是否已被处理，如仍未被处理，则发送消息给相关运营人员；发生预定会议事件，判断离会议开始是否只有十分钟了，如果是，则通知各个与会人员。

### 2. 延时任务实现方式

1. 定期轮询（数据库等）
2. DelayQueue
3. Timer
6. RabbitMQ
7. Quartz
8. Redis Zset

### 3. 死信队列实现消息延迟
死信，顾名思义就是无法被消费的消息。producer 将消息投递到 broker 或者直接到queue 里了，consumer 从 queue 取出消息
进行消费，但某些时候由于特定的原因导致 queue 中的某些消息无法被消费，这样的消息如果没有后续的处理，就变成了死信，有死信自然就有了死信队列。

应用场景:为了保证订单业务的消息数据不丢失，比如说: 用户在商城下单成功并点击去支付后在指定时间未支付时自动失效，但是用户能够看到这条订单的支付失败记录。

#### （1）死信的来源

- 消息 TTL 过期
- 队列达到最大长度(队列满了，无法再添加数据到 mq 中)
- 消息被拒绝(basic.reject 或 basic.nack)并且 requeue=false

#### （2）具体实现

再使用消息队列的时候，应该先画一个消息传递图，这样能够是我们的思路很清晰。

![](D:\blog\picture\dead_queue.png)



## 三、整合Camunda实现工作流

### 1. 流程部署

#### 1.1 流文件部署文件

```java
    @0verride
    public RestfulResponse(Object，CeneralMeta> createDeployment(MultipartFile file) throws IOException（
        ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
        RepositoryService repositoryService = processEngine.getRepositoryService();
        InputStream inputStream_bpmn = file.getInputStream();
        System.out.println("———-—————-->" + file.getInputStream());
        Deployment deployment = repositoryService.createDeployment()
            .addInputStream(file.getOriginalFilename(), inputStream_bpmn)
            .name(file.getOriginalFilename())
            .deploy();
        System.out.println("流程部署id:" + deployment.getId());
        System.out.println("流程部署名称:" + deployment.getName());
        return RestfulResponse.condition(true);
```

#### 1.2 流文件部署文件

```java
    // 与流程定义和部署对象相关的Service
    RepositoryService repositoryService = processEngine.getRepositoryService();
    DeploymentBuilder deploymentBuilder = repositoryService.createDeployment();// 创建一个部署对象
    deploymentBuilder.name(添加部署名称);// 添加部署的名称
    deploymentBuilder.addClasspathResource(文件路径到文件名称);// 从classpath的资源加载，一次只能加载一个文件
    Deployment deployment = deploymentBuilder.deploy();// 完成部署
    log.info("流程Id:" + deployment.getId());
    log.info("流程Name:" + deployment.getName());
```

#### 1.3 压缩包部署文件

```java
    InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream(压缩包路径);
    ZipInputStream zipInputStream = new ZipInputStream(inputStream);
    RepositoryService repositoryService1 = processEngine.getRepositoryService();
    Deployment deployment = repositoryService1.createDeployment()//
            .addZipInputStream(zipInputStream).deploy();
    System.out.println("流程部署id：" + deployment.getId());
    System.out.println("流程部署名称：" + deployment.getName());
```

### 2. 流程启动

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    RuntimeService runtimeService=processEngine.getRuntimeService();
    String processDefinitionkey="这里的key指的是流程图定义的key";
    // 可自定义一下变量，使用map 的形式进行存储
    Map<String,Object> map = new HashMap<String, Object>();
    map.put("title",value);
    // 获取的时候，可以使用下面语句获取对应的value
    runtimeService.getVariable(task.getExecutionId(),key);

    IdentityService identityService = processEngine.getIdentityService();
    identityService.setAuthenticatedUserId("发起人用户id");

    ProcessInstance processInstance=runtimeService.startProcessInstanceByKey(ProcessDefinitionKey,BusinessKey,map);
    System.out.println("流程实例ID："+processInstance.getId());//流程实例ID
    System.out.println("流程定义ID："+processInstance.getProcessDefinitionId());//流程定义ID
    System.out.println(processInstance.getBusinessKey());

```

### 3. 查询代办任务

提示：listPage里的两个参数代表从第几行开始，起始行可为0也就是第一行，第二个参数是需要多少个。

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    TaskService taskService = processEngine.getTaskService();
    // 定义分页参数，实际上这里并不需要，知识举个列子，因为listPage已经自动完成了分页
    int i=(pageNum-1)*pageSize ;
    int i1=pageSize;
    // 取出任务列表
    List<Task> list = taskService.createTaskQuery().taskAssignee(assignee).listPage(i,i1);

    List<TaskLike> taskLikeList = new ArrayList<>();
    // 统计总数
    long count = taskService.createTaskQuery().taskAssignee(assignee).count();
    //count是表示一共有多少条。
    //page的分页就很清楚，两参一个其实页一个，页最大容量
    Page<TaskLike> page = new Page<TaskLike>(pageNum, pageSize);
    RuntimeService runtimeService = processEngine.getRuntimeService();

    for (Task task : list) {
        TaskLike taskLike = new TaskLike();
        taskLike.setAssignee(task.getAssignee());
        taskLike.setCreateTime(task.getCreateTime());
        taskLike.setDescription((String)runtimeService.getVariable(task.getExecutionId(),"title"));
        taskLike.setExecutionId(task.getExecutionId());
        taskLike.setProcessDefinitionId(task.getProcessDefinitionId());
        taskLike.setTaskId(task.getId());
        taskLike.setTaskName(task.getName());
        taskLike.setTaskDefinitionKey(task.getTaskDefinitionKey());
        taskLike.setProcessInstanceId(task.getProcessInstanceId());
        taskLikeList.add(taskLike);
    }
    page.setRecords(taskLikeList);
    page.setTotal(count);
```

### 4. 查询已办任务

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    RuntimeService runtimeService = processEngine.getRuntimeService();
    List<HistoricTaskInstance> list1 = processEngine.getHistoryService() // 历史任务Service
            .createHistoricTaskInstanceQuery() // 创建历史任务实例查询
            .taskAssignee(assignee) // 指定办理人
            .finished() // 查询已经完成的任务
            .listPage(i,i1);
    long count = processEngine.getHistoryService().createHistoricTaskInstanceQuery().taskAssignee(assignee).finished().count();
    //count是表示一共多少条。
    List<TaskLike> taskLikeList = new ArrayList<>();
    Page<TaskLike> page = new Page<TaskLike>(pageNum, pageSize);
    for (HistoricTaskInstance hti : list1) {
        TaskLike taskLike = new TaskLike();//自己创建的任务对象，属性都在下边set里的就不单独摆出来了
        taskLike.setAssignee(hti.getAssignee());
        taskLike.setCreateTime(hti.getStartTime());
        taskLike.setDescription((String)runtimeService.getVariable(hti.getExecutionId(),key));
        //这个传入参数就是启动时设置map的
        taskLike.setExecutionId(hti.getExecutionId());
        taskLike.setProcessDefinitionId(hti.getProcessDefinitionId());
        taskLike.setTaskId(hti.getId());
        taskLike.setTaskName(hti.getName());
        taskLike.setTaskDefinitionKey(hti.getTaskDefinitionKey());
        taskLike.setProcessInstanceId(hti.getProcessInstanceId());
        taskLikeList.add(taskLike);
    }
    page.setRecords(taskLikeList);
    page.setTotal(count);
```

### 5. 完成任务(或审批某个东西)

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    TaskService taskService=processEngine.getTaskService();
    Task task=taskService.createTaskQuery().taskId(taskId).singleResult();
    String processInstancesId=task.getProcessInstanceId();
    IdentityService identityService = processEngine.getIdentityService();
    identityService.setAuthenticatedUserId(userId); //这里设置的是审批人及意见的userId。
    taskService.addComment(taskId,processInstancesId,idea);
    //idea意思是完成时的审批意见，可在Act_Hi_Comment里的massge查询到
    Map<String,Object> map = new HashMap<>();
	map.put("条件1",value)//这个map根据bpmn情况定，传入complete方法
    taskService.complete(taskId);//可多参条件。
```

### 6. 查询审批意见

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    TaskService taskService=processEngine.getTaskService();
    HistoryService hisService = processEngine.getHistoryService();
    List<Comment> list=taskService.getTaskComments(taskId);
```


### 7. 查询这个流程的审批意见

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    HistoryService historyService=processEngine.getHistoryService();
    TaskService taskService=processEngine.getTaskService();
    List<Comment> list = new ArrayList();
    Task task = taskService.createTaskQuery()
            .taskId(taskId)//使用任务ID查询
            .singleResult();

    // 流程实例id
    String processInstanceId = task.getProcessInstanceId();
    // 使用流程实例ID，查询历史任务，获取历史任务对应的每个任务ID
    List<HistoricTaskInstance> htiList = historyService.createHistoricTaskInstanceQuery()//历史任务表查询
        .processInstanceId(processInstanceId)//使用流程实例ID查询
        .list();
    // 遍历集合，获取任务id
    for(HistoricTaskInstance hti:htiList){//任务ID
        String htaskId = hti.getId();//获取批注信息
        List taskList = taskService.getTaskComments(htaskId);//对用历史完成后的任务ID                
        list.addAll(taskList);
    }
```

### 8. 查询流程实例

```java
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    RepositoryService repositoryService = processEngine.getRepositoryService();
    // 查询流程定义
    ProcessDefinitionQuery processDefinitionQuery = repositoryService.createProcessDefinitionQuery();
    // 遍历查询结果
    List<ProcessDefinition> list = processDefinitionQuery.processDefinitionKey(ProcessDefinitionKey)
            .orderByProcessDefinitionVersion().desc().listPage(i,i1);
    List<DefinitionLike> definitionLikeList = new ArrayList<>();
    Page<DefinitionLike> page = new Page<DefinitionLike>(pageNum, pageSize);
    long count =processDefinitionQuery.processDefinitionKey(ProcessDefinitionKey).count();
    for (ProcessDefinition processDefinition : list) {

        DefinitionLike definitionLike  = new DefinitionLike();
        definitionLike.setDeploymentId(processDefinition.getDeploymentId());
        definitionLike.setProcessDefinitionId(processDefinition.getId());
        definitionLike.setProcessDefinitionName(processDefinition.getName());
        definitionLike.setProcessDefinitionKey(processDefinition.getKey());
        definitionLike.setProcessDefinitionVersion(processDefinition.getVersion());
        definitionLikeList.add(definitionLike);
    }

    page.setRecords(definitionLikeList);
    page.setTotal(count);

```

### 9. 任务驳回到起始点

```java
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    TaskService taskService = processEngine.getTaskService();
    HistoryService historyService = processEngine.getHistoryService();
    RuntimeService runtimeService = processEngine.getRuntimeService();
    RepositoryService repositoryService = processEngine.getRepositoryService();
    //获取当前任务，未办理任务id
    HistoricTaskInstance currTask = historyService.createHistoricTaskInstanceQuery()
            .taskId(taskId)
            .singleResult();
    //获取流程实例
    ProcessInstance processInstance = runtimeService.createProcessInstanceQuery()
            .processInstanceId(currTask.getProcessInstanceId())
            .singleResult();
    //获取流程定义
    ProcessDefinitionEntity processDefinitionEntity = (ProcessDefinitionEntity) ((RepositoryServiceImpl) repositoryService)
            .getDeployedProcessDefinition(currTask.getProcessDefinitionId());

    ActivityImpl currActivity = (processDefinitionEntity)
            .findActivity(currTask.getTaskDefinitionKey());
    //清除当前活动出口
    List<PvmTransition> originPvmTransitionList = new ArrayList<PvmTransition>();
    List<PvmTransition> pvmTransitionList = currActivity.getOutgoingTransitions();
    for (PvmTransition pvmTransition : pvmTransitionList) {
        originPvmTransitionList.add(pvmTransition);
    }
    pvmTransitionList.clear();
    //查找上一个user task节点
    List<HistoricActivityInstance> historicActivityInstances = historyService
            .createHistoricActivityInstanceQuery().activityType("userTask")
            .processInstanceId(processInstance.getId())
            .finished()
            .orderByHistoricActivityInstanceEndTime().asc().list();
    TransitionImpl transitionImpl = null;
    if (historicActivityInstances.size() > 0) {
        ActivityImpl lastActivity = (processDefinitionEntity)
                .findActivity(historicActivityInstances.get(0).getActivityId());
        //创建当前任务的新出口
        transitionImpl = currActivity.createOutgoingTransition(lastActivity.getId());
        transitionImpl.setDestination(lastActivity);
    }
    // 完成任务
    List<Task> tasks = taskService.createTaskQuery()
            .processInstanceId(processInstance.getId())
            .taskDefinitionKey(currTask.getTaskDefinitionKey()).list();
    for (Task task : tasks) {
        taskService.complete(task.getId());
        historyService.deleteHistoricTaskInstance(task.getId());
    }
    // 恢复方向
    currActivity.getOutgoingTransitions().remove(transitionImpl);
    for (PvmTransition pvmTransition : originPvmTransitionList) {
        pvmTransitionList.add(pvmTransition);
    }
```

### 10. 查询流程定义

```java
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    RepositoryService repositoryService = processEngine.getRepositoryService();

    List<ProcessDefinition> list =  repositoryService.createProcessDefinitionQuery().latestVersion().listPage(i, i1);
    Page<ProcessDefinitionLike> page = new Page<ProcessDefinitionLike>(pageNum, pageSize);
    long count = repositoryService.createProcessDefinitionQuery().latestVersion().count();

    List<ProcessDefinitionLike> processDefinitionList =new ArrayList<>();
    for(ProcessDefinition processDefinition:list){
        ProcessDefinitionLike processDefinitionLike = new ProcessDefinitionLike();
        processDefinitionLike.setDeploymentId(processDefinition.getDeploymentId());
        processDefinitionLike.setProcessDefinitionName(processDefinition.getName());
        processDefinitionLike.setProcessDefinitionVersion(processDefinition.getVersion());       
        processDefinitionLike.setProcessDefinitionDescription(processDefinition.getDescription())；
        processDefinitionLike.setProcessDefinitionKey(processDefinition.getKey());
        processDefinitionLike.setProcessDefinitionId(processDefinition.getId());
        processDefinitionLike.setProcessDefinitionResourceName(processDefinition.getResourceName());
        processDefinitionList.add(processDefinitionLike);
    }
    page.setRecords(processDefinitionList);
    page.setTotal(count);

```
