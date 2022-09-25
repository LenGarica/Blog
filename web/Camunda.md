## 1. 流程部署
### 1.1 流文件部署文件
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
### 1.2 流文件部署文件
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
### 1.3 压缩包部署文件

```java
    InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream(压缩包路径);
    ZipInputStream zipInputStream = new ZipInputStream(inputStream);
    RepositoryService repositoryService1 = processEngine.getRepositoryService();
    Deployment deployment = repositoryService1.createDeployment()//
            .addZipInputStream(zipInputStream).deploy();
    System.out.println("流程部署id：" + deployment.getId());
    System.out.println("流程部署名称：" + deployment.getName());
```

## 2. 流程启动

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

## 3. 查询代办任务
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

## 4. 查询已办任务
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
## 5. 完成任务(或审批某个东西)

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

## 6. 查询审批意见

```java
    ProcessEngine processEngine=ProcessEngines.getDefaultProcessEngine();
    TaskService taskService=processEngine.getTaskService();
    HistoryService hisService = processEngine.getHistoryService();
    List<Comment> list=taskService.getTaskComments(taskId);
```


## 7. 查询这个流程的审批意见

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

## 8. 查询流程实例

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

## 9. 任务驳回到起始点

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

## 10. 查询流程定义

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

## 11. 真实案例