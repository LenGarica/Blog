# 代驾小程序全栈项目总结

## 第一章 环境搭建

以下环境的搭建，需要使用docker环境。机器配置要求：

| 硬件配置 | 要求            |
| -------- | --------------- |
| CPU      | 当前主流CPU即可 |
| 内存     | 24GB及以上      |
| 硬盘     | 50GB及以上      |
| JDK      | JDK15           |

### 一、宿主机和虚拟机端口映射

使用虚拟机docker部署各种服务，每种服务都拥有不同的端口，为了方便自己操作，可以将宿主机和虚拟机的端口进行映射。

宿主机添加端口映射的例子如下：


```bash
用管理员权限打开powershell或者cmd，命令如下：
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=12001 connectaddress=192.168.114.4 connectport=12001
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=12002 connectaddress=192.168.114.4 connectport=12002
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=12003 connectaddress=192.168.114.4 connectport=12003
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=12004 connectaddress=192.168.114.4 connectport=12004
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=12005 connectaddress=192.168.114.4 connectport=12005
```

删除映射命令如下：


```bash
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=宿主机端口
```

Docker可以为不同的容器分配固定的内网ip地址，而且可以和其他现存容器ip不冲突。


```bash
docker network create --subnet=172.18.0.0/18 mynet
```

### 二、MySQL集群搭建

创建容器命令（一共创建5个，4个做集群，一个做事务处理）

```bash
docker run -p 12001:3306 --name mysql_1 --net mynet --ip 172.18.0.2 -d -v /root/mysql_1/conf/my.cnf:/etc/mysql/my.cnf -v /root/mysql_1/data:/var/lib/mysql -v /root/mysql_1/logs:/logs -v /root/mysql_1/mysql-files:/var/lib/mysql-files -v /etc/localtime:/etc/localtime -e MYSQL_ROOT_PASSWORD=123456 mysql 
```

#### 1.逻辑库划分

| 分片    | 逻辑库    | 备注          | 具体用途                             |
| ------- | --------- | ------------- | ------------------------------------ |
| MySQL_1 | hxds_cst  | 客户逻辑库    | 客户信息、罚款等                     |
| MySQL_1 | hxds_dr   | 司机逻辑库    | 司机信息、实名认证、罚款、钱包等     |
| MySQL_1 | hxds_mis  | MIS系统逻辑库 | 后台管理用的数据表                   |
| MySQL_1 | hxds_rule | 规则逻辑库    | 代驾费计算规则、分账规则、取消规则等 |
| MySQL_2 | hxds_cst  | 客户逻辑库    | 客户信息、罚款等                     |
| MySQL_2 | hxds_dr   | 司机逻辑库    | 司机信息、实名认证、罚款、钱包等     |
| MySQL_2 | hxds_rule | 规则逻辑库    | 代价计费规则、分账规则、取消规则等   |
| MySQL_3 | hxds_odr  | 订单逻辑库    | 订单、账单、好评、分账等             |
| MySQL_3 | hxds_vhr  | 代金券逻辑库  | 代金券、领取情况、使用情况等         |
| MySQL_4 | hxds_odr  | 订单逻辑库    | 订单、账单、好评、分账等             |
| MySQL_4 | hxds_vhr  | 代金券逻辑库  | 代金券、领取情况、使用情况等         |

#### 2.管理逻辑库

创建好了各个逻辑库后，我们在ShardingSphere上就要创建JDBC连接，去连接这些逻辑库。在ShardingSphere的`config/config-sharding.yaml`文件，配置每个数据库。

```yaml
schemaName: hxds
dataSource:
	rep_s1_mis:
		url: jdbc:mysql://172.18.0.2:3306/hxds_mis?useUnicode=true&characterEncoding=UTF8
		username: root
		password: 123456
		maxPoolSize: 50
		minPollSize: 1
	rep_s1_cst:
		url: jdbc:mysql://172.18.0.2:3306/hxds_cst?useUnicode=true&characterEncoding=UTF8
		username: root
		password: 123456
		maxPoolSize: 50
		minPollSize: 1
......
```

#### 3.数据切分

所有切分规则都要定义在`shardingAlgorithms`之下，比如说自己定义了一个切分规则叫做`cst-inline`。`type`设置成`INLINE`代表使用ShardingSphere内置的切分规则。`algorithm-expression`代表具体的算法，这里我详细说一下。

```yaml
shardingAlgorithms:
	cst-inline:
		type: INLINE
		props:
			algorithm-expression: rep_s${(id%2)+1}_cst
```

比如id主键值对2求模，余数是0，那么计算出来的连接名字是`rep_s1_cst`；如果余数是1，那么计算出来的连接是`rep_s2_cst`。然后ShardingSphere就会把SQL语句路由给这个连接的逻辑库去执行，数据也就切分好了。

这个`cst-inline`的切分规则要应用在哪些数据表上呢，我们还要设置一下。例如下面的截图，就是给`tb_cusomer`数据表应用`cst-inline`规则，按照id字段的值来切分数据。

```yaml
tb_customer:
	actualDataNodes: rep_s${1..2}_cst.tb_customer
	databaseStrategy:
		standard:
			shardingColumn: id
			shardingAlgorithmName: cst-inline
	keyGenerateStrategy:
		column: id
		keyGeneratorName: snowflake
```

平时我们用MySQL都喜欢用主键自增长，但是MySQL集群中，千万不能让MySQL生成主键值，必须有程序或者中间件来生成主键值。你想想看，刚才的切分规则是根据主键值的求模余数来路由SQL语句的，如果ShardingSphere接到的INSERT语句没有主键值，那么也就无法路由SQL语句了。你让MySQL节点生成主键值，太晚了，必须由Java程序或者ShardingSphere先生成主键值，才能路由SQL语句。

ShardingSphere内置了雪花算法生成主键值。雪花算法是18位的数字，所以我们创建数据表主键不能是`INT`类型，必须是`BIGINT`类型。

#### 4.雪花算法

使用一个 64 bit 的 long 型的数字作为全局唯一 ID。在分布式系统中的应用十分广泛，且 ID 引入了时间戳，基本上保持自增的。

分布式ID的特点：全局唯一性，不能出现有重复的ID标识，这是基本要求。递增性，确保生成ID对于用户或业务是递增的。高可用性，确保任何时候都能生成正确的ID。高性能性，在高并发的环境下依然表现良好。

雪花算法的原理就是生成一个的 64 位比特位的 long 类型的唯一 id。

| 1位       | 41位   | 5位    | 5位    | 12位 |
| --------- | ------ | ------ | ------ | ---- |
| 固定值为0 | 时间戳 | 机器id | 服务id | 序号 |

    最高 1 位固定值 0，因为生成的 id 是正整数，如果是 1 就是负数了。
    接下来 41 位存储毫秒级时间戳，2^41/(1000*60*60*24*365)=69，大概可以使用 69 年。
    再接下 10 位存储机器码，包括 5 位 datacenterId 和 5 位 workerId。最多可以部署 2^10=1024 台机器。
    最后 12 位存储序列号。同一毫秒时间戳时，通过这个递增的序列号来区分。即对于同一台机器而言，同一毫秒时间戳下，可以生成 2^12=4096 个不重复 id。

可以将雪花算法作为一个单独的服务进行部署，然后需要全局唯一 id 的系统，请求雪花算法服务获取 id 即可。对于每一个雪花算法服务，需要先指定 10 位的机器码，这个根据自身业务进行设定即可。例如机房号+机器号，机器号+服务号，或者是其他可区别标识的 10 位比特位的整数值都行。

雪花算法有以下几个优点：

    1. 高并发分布式环境下生成不重复 id，每秒可生成百万个不重复 id。
    2. 基于时间戳，以及同一时间戳下序列号自增，基本保证 id 有序递增。
    3. 不依赖第三方库或者中间件。
    4. 算法简单，在内存中进行，效率高。

雪花算法有如下缺点：

    1. 依赖服务器时间，服务器时钟回拨时可能会生成重复 id。算法中可通过记录最后一个生成 id 时的时间戳来解决，每次生成 id 之前比较当前服务器时钟是否被回拨，避免生成重复 id。
`注意:`

其实雪花算法每一部分占用的比特位数量并不是固定死的。例如你的业务可能达不到 69 年之久，那么可用减少时间戳占用的位数，雪花算法服务需要部署的节点超过1024 台，那么可将减少的位数补充给机器码用。

注意，雪花算法中 41 位比特位不是直接用来存储当前服务器毫秒时间戳的，而是需要当前服务器时间戳减去某一个初始时间戳值，一般可以使用服务上线时间作为初始时间戳值。

对于机器码，可根据自身情况做调整，例如机房号，服务器号，业务号，机器 IP 等都是可使用的。对于部署的不同雪花算法服务中，最后计算出来的机器码能区分开来即可。

### 三、ShardingSphere搭建

MySQL单表数据超过两千万，CRUD性能就会急速下降，所以我们需要把同一张表的数据切分到不同的MySQL节点中。这需要引入MySQL中间件，其实就是个SQL路由器而已。

这种集群中间件有很多，比如MyCat, ProxySQL、ShardingSphere等等。因为MyCat弃管了，所以我选择了ShardingSphere，功能不输给MyCat，而且还是Apache负责维护的，国内也有很多项目组在用这个产品，手册资料相对齐全，所以相对来说是个主流的中间件。 

我这里使用的是ShardingSphere 5.0版本，属于最新的版本。5.0版本的配置文件和4.0版本有很大的区别，所以大家百度的时候尽量看清楚ShardingSphere的版本号，目前百度上大多数帖子讲ShardingSphere配置，都是基于4.0版本的。 

创建JDK容器：

```bash
docker run -it -d --name ss -p 3307:3307 --net mynet --ip 172.18.0.7 -v /root/ss:/root/ss -e TZ=Asia/Shanghai --privileged=true jdk bash
```

宿主和虚拟机映射端口：


```bash
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3307 connectaddress=192.168.114.4 connectport=3307
```

然后进入到这个容器中，去创建/root/ss， 将ShardingSphere.zip解压缩在这个文件夹中，然后给bin目录chmod 777权限，然后执行./start. sh。即可完成搭建。

### 四、MINIO搭建


```
docker run -it -p 9000:9000 -p 9001:9001 -d --net mynet --ip 172.18.0.10 --restart=always -e "MINIO_ACCESS_KEY=root" -e "MINIO_SECRET_KEY=12345678" -v /root/minio/data:/data -v /root/minio/config:/root/.minio minio/minio server /data --console-address ":9000" --address ":9001"
```

### 五、Redis搭建


```
docker run -d --name redis -p 6379:6379 --privileged=true --restart=always --net mynet --ip 172.18.0.9 -v /root/redis/conf/redis.conf:/redis.conf -v /root/redis/data:/data redis redis-server --appendonly yes
```

### 六、MongoDB搭建


```
docker run -it -d --name mongo -p 27017:27017 --net mynet --ip 172.18.0.8 -v /root/mongo:/etc/mongo -v /root/mongo/data/db:/data/db -m 400m --privileged=true -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=123456 -e TZ=Asia/Shanghai --restart=always mongo
```

### 七、Nacos的搭建


```
docker run -it -d -p 8848:8848 --env MODE=standalone --net mynet --ip 172.18.0.12 -e TZ=Asia/Shanghai --name nacos nacos/nacos-server
```

### 八、Rabbitmq的搭建


```
docker run -it -d --name mq --net mynet --ip 172.18.0.11 -p 5672:5672 -e TZ=Asia/Shanghai --privileged=true rabbitmq                      
```

## 第二章 项目简介

该项目，主要功能是实现代驾小程序。项目的核心角色就是乘客、司机和平台管理员。本项目使用Spring Cloud Alibaba技术栈实现系统微服务开发。

微服务之间能相互调用，首先需要相互发现，所以我们必须把微服务注册到Nacos上面。这样服务A想调用服务B，就可以从Nacos上面拿到服务B的IP地址和端口号了。即便某个服务B宕机了，也没有问题，因为心跳检测机制，Nacos很快就发现了宕机。所以服务A再调用服务B的时候，Nacos会从现有可用的服务B里面挑出一个给服务A。于是服务A和服务B之间调用，并不会因为一个服务B宕机而失败。

在Alibaba SpringCloud架构中，服务之间调用可以采用Feign技术。Feign是Netflix开发的声明式、模板化的HTTP客户端， 它可以帮助我们快捷优雅地调用HTTP API接口。比如说A服务想要调用B服务，需要我们在B服务中正常声明Controller中的Web方法。然后在A项目中定义Feign接口，然后业务层调用这个Feign接口就可以实现对B服务的Web方法调用，而且代码上写几个注解就可以了，挺简单的。

### 一、分布式事务

在分布式系统中，事务参与者在不同的分布式节点上或事务操作的数据源不是同一个，这些情况产生的事务都叫做分布式事务。例如：项目A实现Tb_item表新增、项目B实现tb_item_param新增，现在需要实现商品新增，需要把项目A和项目B两个项目新增的方法组成一个事务，这个事务就是分布式事务。

项目采用了ShardingSphere 5.0 + MySQL 8.0的组合方案。TX-LCN分布式事务可以完美兼容我们的数据库集群。LCN早期设计时，1.0版本和2.0版本设计步骤如下：

- 1）锁定事务单元（Lock）
- 2）确认事务模块状态（Confirm）
- 3）通知事务（Notify）

取各自首字母后名称为LCN。LCN框架从5.0开始兼容了LCN、TCC、TXC三种事务模式，每种模式在实际使用时有着自己对应的注解。**LCN：@LcnTransaction**、**TCC：@TccTransaction**、**TXC：@TxcTransaction**。

为了和LCN框架区分，从5.0开始把LCN框架更名为：TX-LCN分布式事务框架。TX-LCN由两大模块组成，TxClient、TxManager。TxClient作为模块的依赖框架，提供了TX-LCN的标准支持，事务发起方和参与方都属于TxClient。TxManager作为分布式事务的控制方，控制整个事务。

**LCN模式：**

LCN模式是通过代理JDBC中Connection的方式实现对本地事务的操作，然后在由TxManager统一协调控制事务。当本地事务提交回滚或者关闭连接时将会执行假操作，该代理的连接将由LCN连接池管理。

**TCC模式：**

TCC事务机制相对于传统事务机制（X/Open XA Two-Phase-Commit），其特征在于它不依赖资源管理器(RM)对XA的支持，而是通过对（**由业务系统提供的**）**业务逻辑**的调度来实现分布式事务。主要由三步操作，Try: 尝试执行业务、 Confirm:确认执行业务、 Cancel: 取消执行业务。每个TCC事务处理方法可以额外包含confirmXxx和cancelXxx的方法（），出现失败问题，需要在cancel中通过业务逻辑把改变的数据还原回来。

**TXC模式：**

TXC模式命名来源于淘宝，实现原理是在执行SQL之前，先查询SQL的影响数据，然后保存执行的SQL信息和创建锁。当需要回滚的时候就采用这些记录数据回滚数据库，目前锁实现依赖redis分布式锁控制。（在使用lcn时必须要配置redis参数）。

推荐LCN模式。LCN模式，需要TM（TX-Manager）程序来协调分布式事务，所以我们要创建一个Java项目来充当TM节点。并且还要在MySQL 5中创`tx-manager`逻辑库导入数据表。

TM节点的启动类必须要加上`@EnableTransactionManagerServer`注解。代码如下：

```java
@SpringBootApplication
@EnableTransactionManagerServer
public class HxdsTmApplication {

    public static void main(String[] args) {
        SpringApplication.run(HxdsTmApplication.class, args);
    }
}
```

TM节点的配置文件是Properties格式的，里面既要配置数据库连接，还要配置Redis连接。

```properties
spring.application.name=TransactionManager
server.port=7970
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:12005/tx-manager?characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
spring.datasource.username=root
spring.datasource.password=123456
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
spring.jpa.hibernate.ddl-auto=update
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.database=10

tx-lcn.manager.admin-key=123456
tx-lcn.manager.host=127.0.0.1
tx-lcn.manager.port=8070
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.use-generated-keys=true
```

YML文件里面还要设置连接TM节点的配置。

```
tx-lcn:
  client:
    manager-address: 127.0.0.1:8070
```

### 二、鉴权

在微服务架构中，一个大项目被拆分成很多零碎的服务，那么调用这些服务的时候必须要做权限验证。有些微服务项目弄了个全局的鉴权中心，所有微服务接收HTTP调用之后，先去全局鉴权中心验证用户的身份和权限，然后再允许用户调用微服务。

客户端——》网关——》分发到不同子系统——》各个子系统到鉴权中心鉴权——》返回数据

这样做有很多个缺点：

- 一次请求，调用多个微服务，每个都去鉴权中心去鉴权，相当于一次请求要鉴权多次。
- 一次请求，请求了多个微服务，返回了多个数据，前端需要重新组合这些数组，加大了前端的业务逻辑
- 每个微服务都要去鉴权中心鉴权，那么每个方法上都要写上不同角色的权限注解。比如说`test()`这个Web方法，司机只要登陆了小程序就可以调用，但是系统管理人员，必须具备相关权限才能调用，而代驾客户是永远无法看到这个方法的。你说说，我每写一个Web方法，都要思考到底有多少种角色和权限组合方案可以调用当前方法。

**BFF鉴权：**BFF全称是Backends For Frontends，直译过来就是“服务于前端的后端”。 简而言之，BFF就是设计后端微服务API接口时，考虑到不同设备的需求，为不同的设备提供不同的API接口。客户端不是直接访问服务器的公共接口，而是调用BFF层提供的接口，BFF层再调用基础的服务，不同的客户端拥有不同的BFF层，它们定制客户端需要的API接口。有了BFF层之后，客户端只需要发起一次HTTP请求，BFF层就能调用不同的服务，然后把汇总后的数据返回给客户端，这样就减少了外网的HTTP请求，响应速度也就更快。在BFF项目中设置了鉴权模块后，我们只需要在BFF项目的Web方法上面设置权限验证的注解即可，而且也不用考虑不同客户端的权限叠加。毕竟一种BFF项目，只服务于一种客户端。在司机BFF项目的代码上，我们只需要考虑司机端小程序的权限验证，代驾客户根本不可能访问到司机BFF层。

客户端——》网关——》鉴权——》分发到不同子系统——》返回数据

本项目使用SaToken进行认证和授权。

### 三、项目构成

#### 1. 使用了哪些云服务

|序号|云服务名称|具体用途|
|--|--|--|
|1|对象存储服务（COS）|存储司机实名认证的身份证和驾驶证照片|
|2|人脸识别（AiFace）|每天司机接单前的身份核实，并且具备静态活体检测功能|
|3|人员库管理（face-lib）|云端存储司机注册时候的人脸模型，用于身份比对使用|
|4|数据万象（ci）|用于监控大家录音文本内容，判断是否包含暴力和色情|
|5|OCR证件识别插件|用于OCR识别和扫描身份证和驾驶证的信息|
|6|微信同声传译插件|把文字合成语音，播报订单；把录音转换成文本，用于安全监控|
|7|路线规划插件|用于规划司机下班回家的线路，或者小程序订单显示的路线|
|8|地图选点插件|用于小程序上面地图选点操作|
|9|腾讯位置服务|路线规划、定位导航、里程和时间预估|

#### 2. 后端系统构成
|序号|子系统|端口号|具体用途|
|--|--|--|--|
|1|hxds-tm|7970和8070|分布式事务管理节点|
|2|hxds-dr|8001|司机子系统|
|3|hxds-odr|8002|订单子系统|
|4|hxds-snm|8003|消息通知子系统|
|5|hxds-mps|8004|地图子系统|
|6|hxds-oes|8005|订单执行子系统|
|7|hxds-rule|8006|规则子系统|
|8|hxds-cst|8007|客户子系统|
|9|hxds-vhr|8008|代金券子系统|
|10|hxds-nebula|8009|大数据子系统|
|11|hxds-mis-api|8010|MIS子系统|
|12|hxds-workflow|8011|工作流子系统|
|13|bff-driver|8101|司机bff子系统|
|14|bff-customer|8102|客户bff子系统|
|15|gateway|8080|网关子系统|
#### 3. 前端系统构成
|序号|客户端|端口号|具体用途|
|--|--|--|--|
|1|hxds-driver-wx|–|司机端小程序|
|2|hxds-customer-wx|–|客户端小程序|
|3|hxds-mis-vue|3000|MIS系统前端项目|

#### 4. 技术栈

|序号|技术栈|具体用途|
|--|--|--|
|1|SpringBoot|用于创建微服务子系统|
|2|SpringMVC|Web层框架|
|3|MyBatis|持久层框架|
|4|Feign|远程调用|
|5|TX-LCN|分布式事务|
|6|RabbitMQ|系统消息收发|
|7|Swagger|在线调试Web方法|
|8|QLExpress|规则引擎，计算预估费用、取消费用等等|
|9|Quartz|定时器，销毁过期未接单订单、定时自动分账等等|
|10|Phoenix|HBase数据存储|
|11|Minio|私有云存储|
|12|GEO|GPS分区定位计算|
|13|SaToken|认证与授权框架|
|14|VUE3.0|前端框架|
|15|UniAPP|移动端框架|

## 第三章、司机微服务

### 一、相关表结构

`tb_driver`数据表：

| 字段              | 类型     | 是否空值 | 备注信息                                                     |
| ----------------- | -------- | :------- | ------------------------------------------------------------ |
| id                | bigint   | True     | 主键                                                         |
| open_id           | varchar  | True     | 小程序长期授权                                               |
| nickname          | varchar  | True     | 昵称（来自微信）                                             |
| name              | varchar  | False    | 姓名                                                         |
| sex               | char     | False    | 性别（来自微信）                                             |
| photo             | varchar  | False    | 头像（来自微信）                                             |
| pid               | varchar  | False    | 身份证号码                                                   |
| birthday          | date     | False    | 生日                                                         |
| tel               | char     | False    | 电话                                                         |
| email             | varchar  | False    | 邮箱                                                         |
| mail_address      | varchar  | False    | 收信地址                                                     |
| contact_name      | varchar  | False    | 应急联系人                                                   |
| contact_tel       | char     | False    | 应急联系人电话                                               |
| real_auth         | tinyint  | False    | 1未认证，2已认证，3审核中                                    |
| idcard_address    | varchar  | False    | 身份证地址                                                   |
| idcard_expiration | date     | False    | 身份证有效期                                                 |
| idcard_front      | varchar  | False    | 身份证正面                                                   |
| idcard_back       | varchar  | False    | 身份证背面                                                   |
| idcard_holding    | varchar  | False    | 手持身份证                                                   |
| drcard_type       | varchar  | False    | 驾驶证类型                                                   |
| drcard_expiration | date     | False    | 驾驶证有效期                                                 |
| drcard_issue_date | date     | False    | 驾驶证初次领证日期                                           |
| drcard_front      | varchar  | False    | 驾驶证正面                                                   |
| drcard_back       | varchar  | False    | 驾驶证背面                                                   |
| drcard_holding    | varchar  | False    | 手持驾驶证                                                   |
| home              | json     | False    | 家庭地址，包含地址和定位，用于回家导航                       |
| summary           | json     | False    | 摘要信息，level等级，totalOrder接单数，weekOrder周接单，weekComment周好评，appeal正在申诉量 |
| archive           | tinyint  | True     | 是否在腾讯云归档存放司机面部信息                             |
| status            | tinyint  | True     | 状态，1正常，2禁用，3.降低接单量                             |
| create_time       | datetime | True     | 注册时间                                                     |

其中，主键id是ShardingSphere用雪花算法生成，所以即便记录插入成功，我们也不知道主键值是多少。于是我们只能通过司机的openId查询司机的主键值。因为给司机颁发Token令牌是根据司机ID来生成的，所以我们插入司机记录之后，必须要拿到主键值。

`tb_driver_settings`数据表：

| 字段      | 类型   | 禁止空值 | 备注信息     |
| --------- | ------ | -------- | ------------ |
| id        | bigint | True     | 主键         |
| driver_id | bigint | True     | 司机ID       |
| settings  | json   | True     | 司机个人设置 |

其中settings中的内容是：保存的司机在小程序上的设定，比如说是否自动抢单；附近多少公里以内的订单，自己才会接单；代驾订单预估里程在多少公里范围，自己才会接单等等。

```json
{
  "autoAccept": 1, //自动抢单
  "orientation": "", //定向接单
  "listenService": true,  //自动听单
  "orderDistance": 0, //代驾订单预估里程不限，司机不挑订单
  "rangeDistance": 5  //接收距离司机5公里以内的代驾单
}
```

`tb_wallet`数据表：司机钱包表，司机缴纳罚款或者获得系统奖励的时候，都会跟这张表有关系。

| 字段      | 类型    | 禁止空值 | 备注信息                                           |
| --------- | ------- | -------- | -------------------------------------------------- |
| id        | bigint  | True     | 主键                                               |
| driver_id | bigint  | True     | 司机ID                                             |
| balance   | decimal | True     | 钱包金额                                           |
| password  | varchar | False    | 支付密码，如果为空，不能支付，提示用户设置支付密码 |

### 二、司机注册

#### 1.查询要注册的司机是否已经存在记录

司机注册的业务中，我们要保证禁止重复注册，所以先根据openId查询数据表是否存在要注册的司机记录，如果不存在，就允许注册，否则不能注册。本项目中实现根据openId或者driverId查询是否存在司机记录。

微信小程序获取openId的业务逻辑。首先定义好wx的openId请求地址url，然后将我们自己的微信开发者appId、appSecret等内容作为参数，传给这个url，使用hutool提供的post方法进行参数传递，然后将响应转化为json对象，然后获取到openId。

```java
/**
     * 通过微信小程序临时授权code获取openId
     */
public String getOpenId(String code) {
    String url = "https://api.weixin.qq.com/sns/jscode2session";
    HashMap map = new HashMap();
    map.put("appid", appId);
    map.put("secret", appSecret);
    map.put("js_code", code);
    map.put("grant_type", "authorization_code");
    // hutool和JSONUtil都是 hutool包里面的
    String response = HttpUtil.post(url, map);
    JSONObject json = JSONUtil.parseObj(response);
    String openId = json.getStr("openid");
    if (openId == null || openId.length() == 0) {
        throw new RuntimeException("临时登陆凭证错误");
    }
    return openId;
}
```

```sql
<select id="hasDriver" parameterType="Map" resultType="long">
    SELECT COUNT(id) AS ct FROM tb_driver
    <WHERE>
        <if test="openId!=null">
            AND open_id = #{openId}
        </if>
        <if test="driverId!=null">
            AND id = #{driverId}
        </if>
    </where>
</select>
```

Hutool库里面有个函数可以将Java对象转换成Map对象，代码如下：

```java
Map param = BeanUtil.beanToMap(form);

// 这里的form是
@Data
@Schema(description = "新司机注册表单")
public class RegisterNewDriverForm {

    @NotBlank(message = "code不能为空")
    @Schema(description = "微信小程序临时授权")
    private String code;

    @NotBlank(message = "nickname不能为空")
    @Schema(description = "用户昵称")
    private String nickname;

    @NotBlank(message = "photo不能为空")
    @Schema(description = "用户头像")
    private String photo;
}
```

#### 2.插入要注册的司机

通过上面的查询之后，如果不存在就可以插入新的数据了。

```xml
<insert id="registerNewDriver" parameterType="Map">
    INSERT INTO tb_driver
    
    SET open_id = #{openId},
        nickname = #{nickname},
        photo = #{photo},
        real_auth = 1,
        summary = '{"level":0,"totalOrder":0,"weekOrder":0,"weekComment":0,"appeal":0}',
        archive = false,
        `status` = 1
</insert>
```

#### 3.添加司机设定

首先要查询出司机的主键，因为司机主键是ShardingSphere用雪花算法生成，所以，先使用openId查询出司机的主键。然后再添加司机的设定。

```sql
<insert id="insertDriverSettings" parameterType="com.example.hxds.dr.db.pojo.DriverSettingsEntity">
    INSERT INTO tb_driver_settings
    SET driver_id = #{driverId},
        settings = #{settings}
</insert>
```

其中settings是json的格式，里面包含的内容如下：

```json
{
  "autoAccept": 1, //自动抢单
  "orientation": "", //定向接单
  "listenService": true,  //自动听单
  "orderDistance": 0, //代驾订单预估里程不限，司机不挑订单
  "rangeDistance": 5  //接收距离司机5公里以内的代驾单
}
```

#### 4.添加司机钱包记录

注册时，要顺便给司机添加上钱包记录。tb_wallet表是司机钱包表，司机缴纳罚款或者获得系统奖励的时候，都会跟这张表有关系。

```java
<insert id="insert" parameterType="com.example.hxds.dr.db.pojo.WalletEntity">
    INSERT INTO tb_wallet
    SET driver_id = #{driverId},
        balance = #{balance},
        password = #{password}
</insert>
```
### 三、司机bff层用户注册功能

上节所写的内容都是司机子系统中的业务逻辑，但是在本系统中采用的微服务方式，且使用bff层对不同客户端进行鉴权，因此，bff层收到客户端发送的请求后，去调用司机子系统的业务逻辑，而这一调用过程需要使用到feign接口。

#### 1.创建feign接口

在bff中声明一个接口，并使用@FeignClient(value = "")指明要调用的子系统，然后在接口中定义要调用的方法已经请求地址。然后定义service接口、serviceImpl实现类、以及controller类。

```java
// 定义feign接口
@FeignClient(value = "hxds-dr")
public interface DrServiceApi {

    @PostMapping("/driver/registerNewDriver")
    public R registerNewDriver(RegisterNewDriverForm form);
}

// 定义controller层
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    @Resource
    private DriverService driverService;

    @PostMapping("/registerNewDriver")
    @Operation(summary = "新司机注册")
    public R registerNewDriver(@RequestBody @Valid RegisterNewDriverForm form) {
        long driverId = driverService.registerNewDriver(form);
        //在SaToken上面执行登陆，实际上就是缓存userId，然后才有资格拿到令牌
        StpUtil.login(driverId);
        //生成Token令牌字符串（已加密）
        String token = StpUtil.getTokenInfo().getTokenValue();
        return R.ok().put("token", token);
    }
}

// 定义service接口
public interface DriverService {
    public long registerNewDriver(RegisterNewDriverForm form);
}

// 定义serviceImpl实现类
@Service
public class DriverServiceImpl implements DriverService {

    // 注意：这里注入的是feign接口
    @Resource
    private DrServiceApi drServiceApi;

    @Override
    @Transactional
    @LcnTransaction
    public long registerNewDriver(RegisterNewDriverForm form) {
        // feign接口实际上调用的子系统中方法，这里会传回来一个userId字符串
        R r = drServiceApi.registerNewDriver(form);
        // 获取userId，并转为long类型
        long userId = Convert.toLong(r.get("userId"));
        return userId;
    }
}
```

#### 2.前端执行注册

前端提交Ajax请求到bff-driver层。首先需要在main.js中定义方法的url地址，

```js
let baseUrl = "http://192.168.99.106:8201/hxds-driver"

Vue.prototype.url = {
    registerNewDriver: `${baseUrl}/driver/registerNewDriver`
}
```

其次定义button以及执行的方法

```js
<button class="btn" open-type="getUserInfo" @tap="register()">立即注册</button>
register: function() {
    let that = this;
    // 登录方法，由微信提供临时授权码
    uni.login({
        provider: 'weixin',
        success: function(resp) {
            let code = resp.code;
            that.code = code;
        }
    });
    // 获取用户信息的方法
    uni.getUserProfile({
        desc: '获取用户信息',
        success: function(resp) {
            let nickname = resp.userInfo.nickName;
            let avatarUrl = resp.userInfo.avatarUrl;
            console.log(nickname);
            console.log(avatarUrl);
            let data = {
                code: that.code,
                nickname: nickname,
                photo: avatarUrl
            };
            that.ajax(that.url.registerNewDriver, 'POST', data, function(resp) {
                console.log(resp);
                let token = resp.data.token;
                uni.setStorageSync('token', token);
                uni.setStorageSync('realAuth', 1);
                that.$refs.uToast.show({
                    title: '注册成功',
                    type: 'success',
                    callback: function() {
                        uni.redirectTo({
                            url: '../../identity/filling/filling?mode=create'
                        });
                    }
                });
            });
        }
    });
}
```

### 六、腾讯云文件存储服务

#### 1.开通腾讯云对象存储服务

可领取半年免费存储。

为什么要开通存储服务呢？因为微信小程序打包后的体积有限制，因此需要将小程序使用的图片存放在桶中，其次，本次项目需要用到腾讯提供的众多云服务，使用存储桶方便操作。

首先，先创建密钥，主要记录APPID、SecretId、Secret Key。创建各种存储桶，设置不同类型的桶保存自己系统中不同的数据。存储桶可以设置访问用户。系统配置：首先在yml文件中设置如下内容。

```yaml
tencent:
  cloud:
    appId: 腾讯云APPID
    secretId: 腾讯云SecretId
    secretKey: 腾讯云SecretKey
    region-public: 公有存储桶所在地域
    bucket-public: 公有存储桶名称
    region-private: 私有存储桶所在地域
    bucket-private: 私有存储桶名称
```

引入maven
```xml
<dependency>
       <groupId>com.qcloud</groupId>
       <artifactId>cos_api</artifactId>
       <version>5.6.89</version>
</dependency>
```
创建存储桶，分别创建hxds-private，hxds-public。private用来存放身份证和驾驶证这些密度较高的图片，hxds-public用来保存对外公开的图片或者文件。系统会自动在设置的存储桶名称后面添加一串随机数字。

#### 2.开通腾讯数据万象服务

腾讯云提供的数据万象服务，主要用来使用AI监测上传的图片是否存在暴力、色情

注意：免费两个月，后期建议购买按量付费。

提供ImageAuditingRequest和ImageAuditingResponse类

```java
// 1. 先创建请求对象
ImageAuditingRequest request = new ImageAuditingRequest();
// 2. 设置对应的存储桶bucket
request.setBucketName(“存储桶的id”);
// 3. 设置审核类型
request.setDetectType("porn,ads");
// 4. 设置bucket中图片的位置
request.setObjectKey("1.png");
// 5. 调用接口，获取任务响应对象
ImageAuditingResponse response = client.imageAuditing(request);
```

#### 3.封装的云存储操作代码

```java
package com.example.hxds.common.util;

import cn.hutool.core.date.DateUtil;
import cn.hutool.core.util.IdUtil;
import com.example.hxds.common.exception.HxdsException;
import com.qcloud.cos.COSClient;
import com.qcloud.cos.ClientConfig;
import com.qcloud.cos.auth.BasicCOSCredentials;
import com.qcloud.cos.auth.COSCredentials;
import com.qcloud.cos.http.HttpMethodName;
import com.qcloud.cos.http.HttpProtocol;
import com.qcloud.cos.model.*;
import com.qcloud.cos.model.ciModel.auditing.ImageAuditingRequest;
import com.qcloud.cos.model.ciModel.auditing.ImageAuditingResponse;
import com.qcloud.cos.region.Region;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;
import java.net.URL;
import java.util.*;

@Component
public class CosUtil {
    @Value("${tencent.cloud.appId}")
    private String appId;

    @Value("${tencent.cloud.secretId}")
    private String secretId;

    @Value("${tencent.cloud.secretKey}")
    private String secretKey;

    @Value("${tencent.cloud.region-public}")
    private String regionPublic;

    @Value("${tencent.cloud.bucket-public}")
    private String bucketPublic;

    @Value("${tencent.cloud.region-private}")
    private String regionPrivate;

    @Value("${tencent.cloud.bucket-private}")
    private String bucketPrivate;

    //获取访问公有存储桶的连接
    private COSClient getCosPublicClient() {
        // 1. 权限认证，将id和私钥进行认证
        COSCredentials cred = new BasicCOSCredentials(secretId, secretKey);
        // 2. 设置ClientConfig，配置好公有存储桶的地域
        ClientConfig clientConfig = new ClientConfig(new Region(regionPublic));
        // 3. 再设置Https请求
        clientConfig.setHttpProtocol(HttpProtocol.https);
        // 4. 最后将认证对象和config对象封装成CosClient对象
        COSClient cosClient = new COSClient(cred, clientConfig);
        return cosClient;
    }

    //获得访问私有存储桶的连接
    private COSClient getCosPrivateClient() {
        COSCredentials cred = new BasicCOSCredentials(secretId, secretKey);
        ClientConfig clientConfig = new ClientConfig(new Region(regionPrivate));
        clientConfig.setHttpProtocol(HttpProtocol.https);
        COSClient cosClient = new COSClient(cred, clientConfig);
        return cosClient;
    }

    /**
     * 向公有存储桶上传文件
     */
    public HashMap uploadPublicFile(MultipartFile file, String path) throws IOException {
        //上传文件的名字
        String fileName = file.getOriginalFilename(); 
        //文件后缀名
        String fileType = fileName.substring(fileName.lastIndexOf(".")); 
        //避免重名图片在云端覆盖，所以用UUID作为文件名
        path += IdUtil.simpleUUID() + fileType; 
        //设置元数据信息：文件长度、编码、文件内容类型（application/octet-stream或者zip）
        ObjectMetadata meta = new ObjectMetadata();
        meta.setContentLength(file.getSize());
        meta.setContentEncoding("UTF-8");
        meta.setContentType(file.getContentType());

        //向存储桶中保存文件，设置存储桶名称、路径、文件流、元数据
        PutObjectRequest putObjectRequest = new PutObjectRequest(bucketPublic, path, file.getInputStream(), meta);
        // 设置存储类型
        putObjectRequest.setStorageClass(StorageClass.Standard); 
        // 调用cosclient，上传内容
        COSClient client = getCosPublicClient();
        PutObjectResult putObjectResult = client.putObject(putObjectRequest);

        //合成外网访问地址，这个map是返回给前端用户的地址和内容
        HashMap map = new HashMap();
        map.put("url", "https://" + bucketPublic + ".cos." + regionPublic + ".myqcloud.com" + path);
        map.put("path", path);

        //如果保存的是图片，用数据万象服务对图片内容审核
        if (List.of(".jpg", ".jpeg", ".png", ".gif", ".bmp").contains(fileType)) {
            // 这里就是开通的数据万象服务，使用这个类就可以
            //审核图片内容
            ImageAuditingRequest request = new ImageAuditingRequest();
            request.setBucketName(bucketPublic);
            request.setDetectType("porn,terrorist,politics,ads"); //辨别黄色、暴利、政治和广告内容
            request.setObjectKey(path);
            ImageAuditingResponse response = client.imageAuditing(request); //执行审查
            /*
             * 这里去掉了政治内容的审核，因为身份证背面照片有国徽，
             * 会被腾讯云判定为政治内容，导致无法在存储桶保存身份证背面照片
             */
            if (!response.getPornInfo().getHitFlag().equals("0")
                    || !response.getTerroristInfo().getHitFlag().equals("0")
                    || !response.getAdsInfo().getHitFlag().equals("0")
            ) {
                //删除违规图片
                client.deleteObject(bucketPublic, path);
                throw new HxdsException("图片内容不合规");
            }
        }
        client.shutdown();
        return map;
    }

    /**
     * 向私有存储桶上传文件
     */
    public HashMap uploadPrivateFile(MultipartFile file, String path) throws IOException {
        String fileName = file.getOriginalFilename(); //上传文件的名字
        String fileType = fileName.substring(fileName.lastIndexOf(".")); //文件后缀名
        path += IdUtil.simpleUUID() + fileType; //避免重名图片在云端覆盖，所以用UUID作为文件名

        //元数据信息
        ObjectMetadata meta = new ObjectMetadata();
        meta.setContentLength(file.getSize());
        meta.setContentEncoding("UTF-8");
        meta.setContentType(file.getContentType());

        //向存储桶中保存文件
        PutObjectRequest putObjectRequest = new PutObjectRequest(bucketPrivate, path, file.getInputStream(), meta);
        putObjectRequest.setStorageClass(StorageClass.Standard);
        COSClient client = getCosPrivateClient();
        PutObjectResult putObjectResult = client.putObject(putObjectRequest); //上传文件

        HashMap map = new HashMap();
        map.put("path", path);

        //如果保存的是图片，用数据万象服务对图片内容审核
        if (List.of(".jpg", ".jpeg", ".png", ".gif", ".bmp").contains(fileType)) {
            //审核图片内容
            ImageAuditingRequest request = new ImageAuditingRequest();
            request.setBucketName(bucketPrivate);
            request.setDetectType("porn,terrorist,politics,ads"); //辨别黄色、暴利、政治和广告内容
            request.setObjectKey(path);
            ImageAuditingResponse response = client.imageAuditing(request); //执行审查

            if (!response.getPornInfo().getHitFlag().equals("0")
                    || !response.getTerroristInfo().getHitFlag().equals("0")
                    || !response.getPoliticsInfo().getHitFlag().equals("0")
                    || !response.getAdsInfo().getHitFlag().equals("0")
            ) {
                //删除违规图片
                client.deleteObject(bucketPrivate, path);
                throw new HxdsException("图片内容不合规");
            }
        }
        client.shutdown();
        return map;

    }

    /**
     * 获取私有读写文件的临时URL外网访问地址
     */
    public String getPrivateFileUrl(String path) {
        COSClient client = getCosPrivateClient();
        GeneratePresignedUrlRequest request =
                new GeneratePresignedUrlRequest(bucketPrivate, path, HttpMethodName.GET);
        Date expiration = DateUtil.offsetMinute(new Date(), 5);  //设置临时URL有效期为5分钟
        request.setExpiration(expiration);
        URL url = client.generatePresignedUrl(request);
        client.shutdown();
        return url.toString();
    }

    /**
     * 刪除公有存储桶的文件
     */
    public void deletePublicFile(String[] pathes) {
        COSClient client = getCosPublicClient();
        for (String path : pathes) {
            client.deleteObject(bucketPublic, path);
        }
        client.shutdown();
    }

    /**
     * 刪除私有存储桶的文件
     */
    public void deletePrivateFile(String[] pathes) {
        COSClient client = getCosPrivateClient();
        for (String path : pathes) {
            client.deleteObject(bucketPrivate, path);
        }
        client.shutdown();
    }
    
}

```

### 七、开通OCR识别证件信息

当司机用户注册完成后，需要引导用户进行实名认证，这需要使用到OCR和云数据存储功能了。司机需要提交的验证照片主要有，身份证正面、反面、手持身份证、驾驶证正面、反面、手持驾驶证。在小程序中使用OCR证件扫描，我们需要在微信公众平台上设置第三方插件，在menifest.json文件中设置好插件的version和provider号。

小程序ocr插件主要代码如下：

```js
<ocr-navigator @onSuccess="scanIdcardFront" certificateType="idCard" :opposite="false">
    <button class="camera"></button>
</ocr-navigator>

// 身份证正面
scanIdcardFront: function(resp) {
    let that = this;
    let detail = resp.detail;
    that.idcard.pid = detail.id.text;
    that.idcard.name = detail.name.text;
    that.idcard.sex = detail.gender.text;
    that.idcard.address = detail.address.text;
    //需要缩略身份证地址，文字太长页面显示不了
    that.idcard.shortAddress = detail.address.text.substr(0, 15) + '...';
    that.idcard.birthday = detail.birth.text;
    //OCR插件拍摄到的身份证正面照片存储地址
    that.idcard.idcardFront = detail.image_path;
    //让身份证View标签加载身份证正面照片
    that.cardBackground[0] = detail.image_path;
    //发送Ajax请求，上传身份证正面照片
    that.uploadCos(that.url.uploadCosPrivateFile, detail.image_path, 'driverAuth', function(resp) {
        let data = JSON.parse(resp.data);
        let path = data.path;  //身份证照片的云端URL地址
        that.currentImg['idcardFront'] = path; //页面持久层保存身份证云端URL地址
        /*
         * 本页面所有上传到云端的照片云端URL地址都保存到数组中，因为用户可以反复拍摄身份证
         * 照片，那么之前上传的照片到最后应该从云端删除掉。页面提交完整实名认证信息的时候，
         * 需要比对cosImg数组中哪些照片不需要了，让云端删除不需要的证件照片
         */
        that.cosImg.push(path); 
    });
},
// 身份证背面  
scanIdcardBack: function(resp) {
    let that = this;
    let detail = resp.detail;
    //OCR插件拍摄到的身份证背面照片存储地址
    that.idcard.idcardBack = detail.image_path;
    //View标签加载身份证背面照片
    that.cardBackground[1] = detail.image_path;
    //获取身份证过期日期
    let validDate = detail.valid_date.text.split('-')[1];
    //日期格式化成yyyy-MM-dd形式
    that.idcard.expiration = dayjs(validDate, 'YYYYMMDD').format('YYYY-MM-DD');
    //提交Ajax请求给后端
    that.uploadCos(that.url.uploadCosPrivateFile, detail.image_path, 'driverAuth', function(resp) {
        let data = JSON.parse(resp.data);
        let path = data.path;
        that.currentImg['idcardBack'] = path;
        that.cosImg.push(path);
    });
},
// 驾驶证正面
scanDrcardFront: function(resp) {
    let that = this;
    let detail = resp.detail;
    that.drcard.issueDate = detail.issue_date.text; //初次领证日期
    that.drcard.carClass = detail.car_class.text; //准驾车型
    that.drcard.validFrom = detail.valid_from.text; //驾驶证起始有效期
    that.drcard.validTo = detail.valid_to.text; //驾驶证截止有效期
    that.drcard.drcardFront = detail.image_path;
    that.cardBackground[3] = detail.image_path;
    //把驾驶证正面照片上传到云端
    that.uploadCos(that.url.uploadCosPrivateFile, detail.image_path, 'driverAuth', function(resp) {
        let data = JSON.parse(resp.data);
        let path = data.path;
        that.currentImg['drcardFront'] = path;
        that.cosImg.push(path);
    });
},
// 驾驶证背面、手持身份证和手持驾驶证略
```

在bff中声明一个接口，用于接收司机上传的身份证文件。

```java
package com.example.hxds.bff.driver.controller;

import cn.dev33.satoken.annotation.SaCheckLogin;
import com.example.hxds.bff.driver.controller.form.DeleteCosFileForm;
import com.example.hxds.common.exception.HxdsException;
import com.example.hxds.common.util.CosUtil;
import com.example.hxds.common.util.R;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.repository.query.Param;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;
import javax.annotation.Resource;
import javax.validation.Valid;
import java.io.IOException;
import java.util.HashMap;

@RestController
@RequestMapping("/cos")
@Slf4j
@Tag(name = "CosController", description = "对象存储Web接口")
public class CosController {
    @Resource
    private CosUtil cosUtil;

    @PostMapping("/uploadCosPublicFile")
    @SaCheckLogin
    @Operation(summary = "上传文件")
    public R uploadCosPublicFile(@Param("file") MultipartFile file, @Param("module") String module) {
        if (file.isEmpty()) {
            throw new HxdsException("上传文件不能为空");
        }
        try {
            String path = null;
            //TODO 此处应该有path路径赋值
            
            HashMap map = cosUtil.uploadPublicFile(file, path);
            return R.ok(map);
        } catch (IOException e) {
            log.error("文件上传到腾讯云错误", e);
            throw new HxdsException("文件上传到腾讯云错误");
        }
    }

    @PostMapping("/uploadCosPrivateFile")
    @SaCheckLogin
    @Operation(summary = "上传文件")
    public R uploadCosPrivateFile(@Param("file") MultipartFile file, @Param("module") String module) {
        if (file.isEmpty()) {
            throw new HxdsException("上传文件不能为空");
        }
        try {
            String path = null;
            if ("driverAuth".equals(module)) {
                path = "/driver/auth/";
            } else {
                throw new HxdsException("module错误");
            }
            HashMap map = cosUtil.uploadPrivateFile(file, path);
            return R.ok(map);
        } catch (IOException e) {
            log.error("文件上传到腾讯云错误", e);
            throw new HxdsException("文件上传到腾讯云错误");
        }
    }

    @PostMapping("/deleteCosPublicFile")
    @SaCheckLogin
    @Operation(summary = "删除文件")
    public R deleteCosPublicFile(@Valid @RequestBody DeleteCosFileForm form) {
        cosUtil.deletePublicFile(form.getPathes());
        return R.ok();
    }

    @PostMapping("/deleteCosPrivateFile")
    @SaCheckLogin
    @Operation(summary = "删除文件")
    public R deleteCosPrivateFile(@Valid @RequestBody DeleteCosFileForm form) {
        cosUtil.deletePrivateFile(form.getPathes());
        return R.ok();
    }
}
```

除了上述必要的六张照片以外，我们还需要司机填写一些联络方式和紧急联系人。

联络方式主要有：手机号码、电子邮箱、收信地址、紧急联系人、紧急联系人电话

### 八、开通腾讯人脸识别活体检测

新注册的司机，需要进行面部信息采集，以便司机每天第一次接单的时候做身份核验。腾讯云人脸识别既能够进行面部识别，又能够进行活体检测。在腾讯云人脸识别工具上新建一个人员库，输入人员库的名称（通常为项目的名称）、人员库id（通常为项目的英文）、算法模型版本。调用该库的API接口如下：

```java
CreatePersonRequest req = new CreatePersonRequest();
req.setGroupId(groupName); //人员库ID
req.setPersonId(driverId + ""); //人员ID
long gender = sex.equals("男") ? 1L : 2L;
req.setGender(gender);
req.setQualityControl(4L); //照片质量等级
req.setUniquePersonControl(4L); //重复人员识别等级
req.setPersonName(name); //姓名
req.setImage(photo); //base64图片
CreatePersonResponse resp = client.CreatePerson(req);
```

注意，需要在yml中设置如下内容：

```yml
tencent:
    face:
        groupName: hxds
        region: ap-beijing
```

业务逻辑

```java
@Service
@Slf4j
public class DriverServiceImpl implements DriverService {
    @Value("${tencent.cloud.secretId}")
    private String secretId;

    @Value("${tencent.cloud.secretKey}")
    private String secretKey;

    @Value("${tencent.cloud.face.groupName}")
    private String groupName;

    @Value("${tencent.cloud.face.region}")
    private String region;
    
    
    @Override
    @Transactional
    @LcnTransaction
    public String createDriverFaceModel(long driverId, String photo) {
        //查询司机的姓名和性别
        HashMap map = driverDao.searchDriverNameAndSex(driverId);
        String name = MapUtil.getStr(map, "name");
        String sex = MapUtil.getStr(map, "sex");
        
        //腾讯云端创建司机面部档案
        Credential cred = new Credential(secretId, secretKey);
        IaiClient client = new IaiClient(cred, region);
        try {
            CreatePersonRequest req = new CreatePersonRequest();
            req.setGroupId(groupName);   //人员库ID
            req.setPersonId(driverId + "");   //人员ID
            long gender = sex.equals("男") ? 1L : 2L;
            req.setGender(gender);
            req.setQualityControl(4L);   //照片质量等级
            req.setUniquePersonControl(4L);   //重复人员识别等级
            req.setPersonName(name);   //姓名
            req.setImage(photo);   //base64图片
            CreatePersonResponse resp = client.CreatePerson(req);
            if (StrUtil.isNotBlank(resp.getFaceId())) {
                //更新司机表的archive字段值
                int rows = driverDao.updateDriverArchive(driverId);
                if (rows != 1) {
                    return "更新司机归档字段失败";
                }
            }
        } catch (TencentCloudSDKException e) {
            log.error("创建腾讯云端司机档案失败", e);
            return "创建腾讯云端司机档案失败";
        }
        return "";
    }
}
```
### 九、司机登录

司机登陆的时候，我们要判断司机有没有做实名认证，面部信息采集了没，所以登陆业务的步骤还是挺多的。根据司机的OpenId（微信提供OpenID）查询司机的注册信息，其中包含主键、实名认证和面部录入的情况。

```java
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    @PostMapping("/login")
    @Operation(summary = "登陆系统")
    public R login(@RequestBody @Valid LoginForm form) {
        // form表单中只有一个微信临时授权id
        HashMap map = driverService.login(form);
        if (map != null) {
            long driverId = MapUtil.getLong(map, "id");
            // 是否已认证
            byte realAuth = Byte.parseByte(MapUtil.getStr(map, "realAuth"));
            // 是否已经录入面部识别
            boolean archive = MapUtil.getBool(map, "archive");
            // 登录，并发送token给前端
            StpUtil.login(driverId);
            String token = StpUtil.getTokenInfo().getTokenValue();
            // 将token和是否已经认证、是否录入面部识别作为结果返回
            return R.ok().put("token", token).put("realAuth", realAuth).put("archive", archive);
        }
        return R.ok();
    }
}
```

#### 1.登陆时可能出现的几种情况：

- 正常登陆

之前我们注册了司机，并且填写了实名认证，也录入了面部信息。这里我们测试点击登陆按钮，如果能正常跳转到工作台页面，就说明没有问题。

- 没有做面部录入

因为实名认证之后，自动跳转到面部录入页面，这时候如果司机关闭了小程序，那么就会出现有注册信息和实名认证，但是没有面部信息。所以登陆的时候，我们要跳转到面部录入页面。

- 没有做实名认证

用户注册司机之后，在实名认证页面关闭了小程序，没有完成实名认证。我们在登陆环节中要判断出来，然后跳转到实名认证页面。

#### 2.个人中心信息获取

```java
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    @PostMapping("/searchDriverBaseInfo")
    @Operation(summary = "查询司机基本信息")
    @SaCheckLogin
    public R searchDriverBaseInfo() {
        long driverId = StpUtil.getLoginIdAsLong();
        SearchDriverBaseInfoForm form = new SearchDriverBaseInfoForm();
        form.setDriverId(driverId);
        HashMap map = driverService.searchDriverBaseInfo(form);
        return R.ok().put("result", map);
    }
}
```

#### 3.退出登录

```java
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    @GetMapping("/logout")
    @Operation(summary = "退出系统")
    @SaCheckLogin
    public R logout() {
        StpUtil.logout();
        return R.ok();
    }
}
```

### 十、司机小程序端工作台

司机首页工作台要显示的信息非常多，例如代驾时长、今日收入、今日成单。在订单表中查询当天代驾总时长、总收入和订单数。

#### 1.相关表结构 tb_order

| 字段名               | 类型     | 非空约束 | 备注信息                                                     |
| -------------------- | -------- | -------- | ------------------------------------------------------------ |
| id                   | bigint   | True     | 主键                                                         |
| uuid                 | varchar  | True     | 订单序列号                                                   |
| customer_id          | bigint   | True     | 客户ID                                                       |
| start_place          | varchar  | True     | 起始地点                                                     |
| start_place_location | json     | True     | 起始地点坐标信息                                             |
| end_place            | varchar  | True     | 结束地点                                                     |
| end_place_location   | json     | True     | 结束地点坐标信息                                             |
| expects_mileage      | decimal  | True     | 预估里程                                                     |
| real_mileage         | decimal  | False    | 实际里程                                                     |
| return_mileage       | decimal  | False    | 返程里程                                                     |
| expects_fee          | decimal  | True     | 预估订单价格                                                 |
| favour_fee           | decimal  | True     | 顾客好处费                                                   |
| incentive_fee        | decimal  | False    | 系统奖励费                                                   |
| real_fee             | decimal  | False    | 实际订单价格                                                 |
| driver_id            | bigint   | False    | 司机ID                                                       |
| date                 | date     | True     | 订单日期                                                     |
| create_time          | datetime | True     | 订单创建时间                                                 |
| accept_time          | datetime | False    | 司机接单时间                                                 |
| arrive_time          | datetime | False    | 司机到达时间                                                 |
| start_time           | datetime | False    | 代驾开始时间                                                 |
| end_time             | datetime | False    | 代驾结束时间                                                 |
| waiting_minute       | smallint | False    | 代驾等时分钟数                                               |
| prepay_id            | varchar  | False    | 微信预支付单ID                                               |
| pay_id               | varchar  | False    | 微信支付单ID                                                 |
| pay_time             | datetime | False    | 微信付款时间                                                 |
| charge_rule_id       | bigint   | True     | 费用规则ID                                                   |
| cancel_rule_id       | bigint   | False    | 订单取消规则ID                                               |
| car_plate            | varchar  | True     | 车牌号                                                       |
| car_type             | varchar  | True     | 车型                                                         |
| status               | tinyint  | True     | 1等待接单，2已接单，3司机已到达，4开始代驾，5结束代驾，6未付款，7已付款，8订单已结束，9顾客撤单，10司机撤单，11事故关闭，12其他 |
| remark               | varchar  | False    | 订单备注信息                                                 |

#### 1.查询当天代驾总时长、总收入和订单数

```sql
<select id="searchDriverTodayBusinessData" parameterType="long" resultType="HashMap">
    SELECT SUM(TIMESTAMPDIFF(HOUR,end_time, start_time)) AS duration,
           SUM(real_fee)                                 AS income,
           COUNT(id)                                     AS orders
    FROM tb_order
    WHERE driver_id = #{driverId}
      AND `status` IN (5, 6, 7, 8)
      AND date = CURRENT_DATE
</select>
```

其中，status有 1等待接单，2已接单，3司机已到达，4开始代驾，5结束代驾，6未付款，7已付款，8订单已结束，9顾客撤单，10司机撤单，11事故关闭，12其他 这些状态

**业务层逻辑：**

```java
@Service
public class DriverServiceImpl implements DriverService {
    @Resource
    private OdrServiceApi odrServiceApi;
        
    @Override
    public HashMap searchWorkbenchData(long driverId) {
        //查询司机当天业务数据
        SearchDriverTodayBusinessDataForm form_1 = new SearchDriverTodayBusinessDataForm();
        form_1.setDriverId(driverId);
        R r = odrServiceApi.searchDriverTodayBusinessData(form_1);
        HashMap business = (HashMap) r.get("result");

        //查询司机的设置
        SearchDriverSettingsForm form_2 = new SearchDriverSettingsForm();
        form_2.setDriverId(driverId);
        r = drServiceApi.searchDriverSettings(form_2);
        HashMap settings = (HashMap) r.get("result");
        HashMap result = new HashMap() {{
            put("business", business);
            put("settings", settings);
        }};
        return result;
    }
}
```

**Controller层逻辑：**

```java
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    ……
        
    @PostMapping("/searchWorkbenchData")
    @Operation(summary = "查询司机工作台数据")
    @SaCheckLogin
    public R searchWorkbenchData() {
        long driverId = StpUtil.getLoginIdAsLong();
        HashMap result = driverService.searchWorkbenchData(driverId);
        return R.ok().put("result", result);
    }
}
```

### 十一、本章总结

#### 1.在司机端小程序实现新司机注册

在这一章，我们最先开发的功能是司机注册，我们把采集到的昵称、头像和OpenID，写入到数据库里面，还在钱包表和司机设置表里面添加了新纪录。

#### 2.实现司机实名认证信息采集

这些仅仅是新司机注册的第一个步骤，接下来我们需要让司机填写实名认证信息，这里用上了OCR证件扫描插件。身份证和驾驶证的信息是通过扫描获得的，联络方式则需要司机手动填写。文字信息我们就直接保存到数据库，证件照片，我们是保存到腾讯云对象存储服务里面，这些过程大家还都有印象是吧。

#### 3.利用腾讯云实现司机面部信息采集

填写好实名认证，接下来要做面部采集。我们用上了腾讯云的面部识别服务，可以把采集到的司机面部照片保存到人员库里面，将来司机每天第一次接单的时候，需要把刚采集的面部照片和人员库里面的照片做比对，如果是同一个人，那么就允许司机接单。

#### 4.完成司机端用户登陆

写好了以上三个功能，算是完成了新司机注册的流程。那么接下来，司机登陆的功能咱们必须给做出来。其实也没什么难度，就是用openId到数据库里面查找司机记录，然后根据driverId生成令牌返回给司机端小程序，小程序把令牌保存到storage里面，有了登陆凭证也就算登陆成功了。以后再发起Ajax请求，在请求头中附带令牌，后端程序就知道你是已经登陆过的司机。包括我们在测试Web方法的时候，也要在请求头里面设置上令牌，要不然就没法测试Web方法。

#### 5.实现司机实名认证信息的修改

如果司机更换了住址，想要修改通信地址。或者领取了新的身份证，想要重新修改实名认证信息也是可以的。重新进入到filling.vue页面，需要把司机原有的实名信息加载出来，这里就有个问题。我们要从云端加载证件照片，势必要生成临时的URL地址，腾讯云早就为我们提供这个功能了，我在代码里设定的过期时间是5分钟，超过5分钟，证件照片的URL地址就自动失效，在一定程度上能保护我们的隐私。至于说修改实名信息，就是重新执行一遍UPDATE语句，我们早就实现了，不需要我们额外写新代码。

#### 6.实现司机个人页面的数据统计

司机在修改自己实名认证的过程中，先要进入到mine.vue页面，这个页面上有一些统计数据需要我们查询出来，所以在这一章里面我们写了代码，把这些统计数据显示到了mine.vue页面上

#### 7.实现工作台页面数据的统计

除此之外，工作台页面上的统计数据我们也要查询出来，这个并不复杂，只是要调用多个子系统才能完成，这个也是我们第一次看到bff-driver的业务层，调用多个子系统查询数据。

#### 8.审批司机实名认证申请

在这一章的最后，我们做了实名认证的审批。首先要在Web页面上加载出分页数据，然后我们展开面板可以看到更详细的信息，包括证件照片，甚至说你点击证件照片还能自动放大，这个功能我没给大家演示，你可以自己实验一下，非常有意思。如果你认为司机的实名信息没有问题，那么就可以通过该申请，其实就是把real_auth字段更新成2状态。只不过我们现在还没做到消息通知模块，等将来写到这个模块的时候，我们应该给司机端发送审批结果的通知消息。

## 第四章 乘客下单和司机抢单

流程：先是乘客端下单，然后代驾系统预估代驾线路、里程和费用，并且把订单推送给附近符合条件的司机。

### 一、开通腾讯位置服务

在腾讯云控制面板上，创建一个地图应用，名字可以随便定义，类型选择“出行”，然后就能拿到地图应用的密钥了。拿到密钥，点击编辑授权自己的微信小程序APP ID，然后开启WebServiceAPI域名白名单。腾讯位置服务给开发者提供了免费的调用额度，只要你每天调用位置服务各种API次数不超过限额，那么就是免费的，对于开发者来说是足够用了。

在代驾的移动端、Web端和后端Java项目中，都用到了腾讯位置服务。**为什么要这么做？**比如说乘客下单的时候，虽然小程序可以直接调用地图API计算出代驾行程的里程，但是你把这个里程用Ajax交给后端的Java程序，是绝对不被承认的。因为你自己用POSTMAN也能模拟Ajax发送请求，传过来的一个随意编造的里程，那怎么能行，所以后端Java项目必须要调用Java版本的腾讯位置API，重新计算代驾的里程。

在yml文件中配置好map服务：

```java
tencent:
  map:
    key: 填写你自己地图应用的Key
```

用户小程序端需要将起点和重点的经纬度传递给后端。传递的form表单如下所示：

```java
@Data
@Schema(description = "预估里程和时间的表单")
public class EstimateOrderMileageAndMinuteForm {
    @NotBlank(message = "mode不能为空")
    @Pattern(regexp = "^driving$|^walking$|^bicycling$")
    @Schema(description = "计算方式")
    private String mode;

    @NotBlank(message = "startPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "startPlaceLatitude内容不正确")
    @Schema(description = "订单起点的纬度")
    private String startPlaceLatitude;

    @NotBlank(message = "startPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "startPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String startPlaceLongitude;

    @NotBlank(message = "endPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "endPlaceLatitude内容不正确")
    @Schema(description = "订单终点的纬度")
    private String endPlaceLatitude;

    @NotBlank(message = "endPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "endPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String endPlaceLongitude;
}
```

#### 1. 后端封装预估里程和时间

```java
@Service
@Slf4j
public class MapServiceImpl implements MapService {
    //预估里程的API地址
    private String distanceUrl = "https://apis.map.qq.com/ws/distance/v1/matrix/";
    @Value("${tencent.map.key}")
    private String key;
    /**
     * mode 是计算方式的意思  有以下取值：driving：驾车、walking：步行、bicycling：自行车
     */
    public HashMap estimateOrderMileageAndMinute(String mode, 
                                                 String startPlaceLatitude, 
                                                 String startPlaceLongitude,
                                                 String endPlaceLatitude, 
                                                 String endPlaceLongitude) {
        HttpRequest req = new HttpRequest(distanceUrl);
        req.form("mode", mode);
        req.form("from", startPlaceLatitude + "," + startPlaceLongitude);
        req.form("to", endPlaceLatitude + "," + endPlaceLongitude);
        req.form("key", key);
        HttpResponse resp = req.execute();
        JSONObject json = JSONUtil.parseObj(resp.body());
        // 预估里程的状态，如果为0表示正常，如果不为0则表示有异常
        int status = json.getInt("status");
        // 异常信息
        String message = json.getStr("message");
        System.out.println(message);
        if (status != 0) {
            log.error(message);
            throw new HxdsException("预估里程异常：" + message);
        }
        JSONArray rows = json.getJSONObject("result").getJSONArray("rows");
        JSONObject element = rows.get(0, JSONObject.class).getJSONArray("elements").get(0, JSONObject.class);
        // 距离，单位米
        int distance = element.getInt("distance");
        // 将距离转化为字符串里程为公里
        String mileage = new BigDecimal(distance).divide(new BigDecimal(1000)).toString();
        // 时长，单位秒
        int duration = element.getInt("duration");
        String temp = new BigDecimal(duration).divide(new BigDecimal(60), 0, RoundingMode.CEILING).toString();
        // 将时长转换为分钟
        int minute = Integer.parseInt(temp);
        HashMap map = new HashMap() {{
            put("mileage", mileage);
            put("minute", minute);
        }};
        return map;
    }
}
```

#### 2. 后端封装查询最佳路线

最佳路线需要写在后端程序中，因为调用腾讯位置服务需用到密钥。如果是mis管理系统，把密钥写到JS代码中，普通用户查看网页源码就能看到，拿到你的密钥之后，任何人都可以使用这个密钥调用位置服务的API接口。但是，如果是小程序客户端，那可以将最佳路线的业务逻辑和密钥写在JS代码，因为普通用户在小程序上无法获取到这些信息。

```java
@Service
@Slf4j
public class MapServiceImpl implements MapService {
    //规划行进路线的API地址
    private String directionUrl = "https://apis.map.qq.com/ws/direction/v1/driving/";

    public HashMap calculateDriveLine(String startPlaceLatitude, 
                                      String startPlaceLongitude,
                                      String endPlaceLatitude, 
                                      String endPlaceLongitude) {
        HttpRequest req = new HttpRequest(directionUrl);
        req.form("from", startPlaceLatitude + "," + startPlaceLongitude);
        req.form("to", endPlaceLatitude + "," + endPlaceLongitude);
        req.form("key", key);

        HttpResponse resp = req.execute();
        JSONObject json = JSONUtil.parseObj(resp.body());
        int status = json.getInt("status");
        if (status != 0) {
            throw new HxdsException("执行异常");
        }
        JSONObject result = json.getJSONObject("result");
        HashMap map = result.toBean(HashMap.class);
        return map;
    }
}
```

#### 3.小程序端开启GPS实时定位

有两种开启小程序实时定位的方法：**startLocationUpdate()**和**startLocationUpdateBackground()**。其中后者是在小程序被挂到后台，也能获取实时定位。这个我们不太需要，我们只需要前者就可以了，只要小程序正常使用就能获得实时定位，挂到后台就不获取定位，这样还能节省手机电量。在App.vue页面中定义JS代码，开启实时获得GPS定位。**因为小程序启动之后会运行onLaunch()函数，所以我们就把开启实时定位的代码写到onLaunch()函数里面。**


```js
onLaunch: function() {
    //开启GPS后台刷新
    wx.startLocationUpdate({
        success(resp) {
            console.log('开启定位成功');
        },
        fail(resp) {
            console.log('开启定位失败');
        }
    });
    //GPS定位变化就自动提交给后端
    wx.onLocationChange(function(resp) {
        let latitude = resp.latitude;
        let longitude = resp.longitude;
        let location = { latitude: latitude, longitude: longitude };
        //触发自定义事件
        uni.$emit('updateLocation', location);
    });
},
```

**小程序每次获得最新GPS定位坐标之后，都会执行onLocationChange()函数，然后我们就能拿到最新的经纬度坐标。**但是我们想要把坐标数据传递给工作台页面，很多人会想到Storage机制。但是Storage在频繁读写的条件下会丢失数据，特别是小程序一秒钟会获得很多GPS定位，读写Storage很多次，必然会丢失数据，所以千万不能使用Storage机制。

为了在页面之间快速高频传递数据，我们要用自定义事件。比如在App.vue页面抛出一个自定义事件，可以在其中绑定数据，然后让工作台页面捕获到这个事件，于是数据就传递给了工作台页面。


#### 4.捕获定位事件

在onShow()回调函数中，定义JS代码，捕获自定义事件，更新地图定位的坐标，并且还要把定位坐标计算出对应的地址信息，然后设置成默认的上车点。把坐标转换成地址的功能叫做“逆地址解析”。

```js
onShow: function() {
    let that = this;
    that.map = uni.createMapContext('map');
    let qqmapsdk = new QQMapWX({
        key: that.tencent.map.key
    });
    //实时获取定位
    uni.$on('updateLocation', function(location) {
        //console.log(location);
        //避免地图选点的内容被逆地址解析覆盖
      	if(that.flag!=null){
				    return
		}
        let latitude = location.latitude;
        let longitude = location.longitude;
        that.latitude = latitude;
        that.longitude = longitude;
        that.from.latitude = latitude;
        that.from.longitude = longitude;
        //把坐标解析成地址
        qqmapsdk.reverseGeocoder({
            location: {
                latitude: latitude,
                longitude: longitude
            },
            success: function(resp) {
                //console.log(resp);
                that.from.address = resp.result.address;
            },
            fail: function(error) {
                console.log(error);
            }
        });
    });
},
onHide: function() {
    uni.$off('updateLocation');
},
```

#### 5.点击回位

回位图标的点击事件回调函数`returnLocationHandle()`需要我们来声明一下，内容还是很简单的。


```js
returnLocationHandle: function() {
    this.map.moveToLocation();
}
```

#### 6.选择起点和终点

点击工作台页面上的起点或者终点标签会自动跳转到地图选点的页面，我们可以输入查找具体的上车点或者终点位置。这个地图选点页面不需要我们设计，是腾讯位置服务小程序插件自带的。在乘客端小程序的`manifest.json`文件中引用了地图选点插件。

声明点击起点或者终点标签点击事件的回调函数，`chooseLocationHandle()`函数内容如下：

```javascript
chooseLocationHandle: function(flag) {
    let that = this;
    let key = that.tencent.map.key; //使用在腾讯位置服务申请的key
    let referer = that.tencent.map.referer; //调用插件的app的名称
    let latitude = that.latitude;
    let longitude = that.longitude;
    that.flag = flag;
    let data = JSON.stringify({
        latitude: latitude,
        longitude: longitude
    });
    uni.navigateTo({
        url: `plugin://chooseLocation/index?key=${key}&referer=${referer}&location=${data}`
    });
},
```

当我们设置好地图选点地址之后，小程序会自动跳回到工作台页面。我们在工作台页面的`onShow()`函数中要获取用户的选点地址.

```js
onShow: function() {
    let location = chooseLocation.getLocation();
    if (location != null) {
        let place = location.name;
        let latitude = location.latitude;
        let longitude = location.longitude;
        if (that.flag == 'from') {
            that.from.address = place;
            that.from.latitude = latitude;
            that.from.longitude = longitude;
        } else {
            that.to.address = place;
            that.to.latitude = latitude;
            that.to.longitude = longitude;
            //跳转到线路页面
            uni.setStorageSync("from",that.from)
            uni.setStorageSync("to",that.to)
            uni.navigateTo({
                url: `../create_order/create_order`
            });
        }
    }
},
onHide: function() {
    uni.$off('updateLocation');
    //清除地图选点的结果
    chooseLocation.setLocation(null);
},
onUnload: function() {
    //清除地图选点的结果
    chooseLocation.setLocation(null);
    this.flag = null;
}
```

### 二、乘客下单

#### 1.前端计算最佳路线

在前端中计算最佳线路、预估里程和时长的代码我们可以用一个函数给封装起来，将来使用的时候调用起来非常方便。

```js
calculateLine: function(ref) {
    qqmapsdk.direction({
        mode: 'driving',
        from: {
            latitude: ref.from.latitude,
            longitude: ref.from.longitude
        },
        to: {
            latitude: ref.to.latitude,
            longitude: ref.to.longitude
        },
        success: function(resp) {
            if (resp.status != 0) {
                uni.showToast({
                    icon: 'error',
                    title: resp.message
                });
                return;
            }
            // 默认选择第一条路线，因为最佳路线会推送好几条
            let route = resp.result.routes[0];
            // 距离
            let distance = route.distance;
            // 时长（含路况）
            let duration = route.duration;
            // 方案坐标点串，是个压缩数组
            let polyline = route.polyline;
            ref.distance = Math.ceil((distance / 1000) * 10) / 10;
            ref.duration = duration;
            // 获取坐标点
            let points = ref.formatPolyline(polyline);

            ref.polyline = [
                {
                    points: points,
                    width: 6,
                    color: '#05B473',
                    arrowLine: true
                }
            ];
            ref.markers = [
                {
                    id: 1,
                    latitude: ref.from.latitude,
                    longitude: ref.from.longitude,
                    width: 25,
                    height: 35,
                    anchor: {
                        x: 0.5,
                        y: 0.5
                    },
                    iconPath: 'https://mapapi.qq.com/web/lbs/javascriptGL/demo/img/start.png'
                },
                {
                    id: 2,
                    latitude: ref.to.latitude,
                    longitude: ref.to.longitude,
                    width: 25,
                    height: 35,
                    anchor: {
                        x: 0.5,
                        y: 0.5
                    },
                    iconPath: 'https://mapapi.qq.com/web/lbs/javascriptGL/demo/img/end.png'
                }
            ];
        }
    });
},
```

在选点完成后，小程序会跳转同时将起始地址传给创建订单的页面create_order.vue，同时还需要调用腾讯位置服务的API，显示计算最佳线路，预估里程和时间。接下来，乘客设置代驾车型和车牌了。代驾系统要做存档，将来理赔的时候也有依据。乘客既可以添加车辆，也能查询乘客的车辆列表，甚至还能删除车辆信息。

#### 2.添加乘客车辆

**添加车辆信息表单**

```java
@Data
@Schema(description = "添加客户车辆的表单")
public class InsertCustomerCarForm {
    @NotNull(message = "customerId不能为空")
    @Min(value = 1, message = "customerId不能小于1")
    @Schema(description = "客户ID")
    private Long customerId;

    @NotBlank(message = "carPlate不能为空")
    @Pattern(regexp = "^([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}(([0-9]{5}[DF])|([DF]([A-HJ-NP-Z0-9])[0-9]{4})))|([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}[A-HJ-NP-Z0-9]{4}[A-HJ-NP-Z0-9挂学警港澳]{1})$",
            message = "carPlate内容不正确")
    @Schema(description = "车牌号")
    private String carPlate;

    @NotBlank(message = "carType不能为空")
    @Pattern(regexp = "^[\\u4e00-\\u9fa5A-Za-z0-9\\-\\_\\s]{2,20}$", message = "carType内容不正确")
    @Schema(description = "车型")
    private String carType;
}
```

**查询客户车辆的表单**

```java
@Data
@Schema(description = "查询客户车辆的表单")
public class SearchCustomerCarListForm {
    @NotNull(message = "customerId不能为空")
    @Min(value = 1, message = "customerId不能小于1")
    @Schema(description = "客户ID")
    private Long customerId;
}
```

**删除客户车辆的表单**

```java
@Data
@Schema(description = "删除客户车辆的表单")
public class DeleteCustomerCarByIdForm {
    @NotNull(message = "id不能为空")
    @Min(value = 1, message = "id不能小于1")
    @Schema(description = "车辆ID")
    private Long id;
}
```

**业务逻辑**

```java
@RestController
@RequestMapping("/customer/car")
@Tag(name = "CustomerCarController", description = "客户车辆Web接口")
public class CustomerCarController {
    @Resource
    private CustomerCarService customerCarService;

    @PostMapping("/insertCustomerCar")
    @Operation(summary = "添加客户车辆")
    public R insertCustomerCar(@RequestBody @Valid InsertCustomerCarForm form) {
        CustomerCarEntity entity = BeanUtil.toBean(form, CustomerCarEntity.class);
        customerCarService.insertCustomerCar(entity);
        return R.ok();
    }

    @PostMapping("/searchCustomerCarList")
    @Operation(summary = "查询客户车辆列表")
    public R searchCustomerCarList(@RequestBody @Valid SearchCustomerCarListForm form) {
        ArrayList<HashMap> list = customerCarService.searchCustomerCarList(form.getCustomerId());
        return R.ok().put("result", list);
    }

    @PostMapping("/deleteCustomerCarById")
    @Operation(summary = "删除客户车辆")
    public R deleteCustomerCarById(@RequestBody @Valid DeleteCustomerCarByIdForm form) {
        int rows = customerCarService.deleteCustomerCarById(form.getId());
        return R.ok().put("rows", rows);
    }
}
```

在`add_car.vue`页面上，我们可以设置车辆的车型和车牌号码，然后点击保存按钮，发起Ajax请求，车辆记录就保存到数据库里面了。

在`car_list.vue`页面上，应该先把查询到的车辆列表一次性显示出来，因为记录并不多，所以我们就不用分页功能了。然后用户长按某个车辆会出现对话框，用户确认后就会删除该车辆。

#### 3.后端重写计算最佳路线

乘客提交创建订单的请求之后，需要后端Java程序重新计算最佳线路、里程和时间。这是为了避免有人用POSTMAN模拟客户端，随便输入代驾的起点、终点、里程和时长，所以后端程序必须重新计算这些数据，并且写入到订单记录中

```java
@Data
@Schema(description = "预估里程和时间的表单")
public class EstimateOrderMileageAndMinuteForm {
    @NotBlank(message = "mode不能为空")
    @Pattern(regexp = "^driving$|^walking$|^bicycling$")
    @Schema(description = "计算方式")
    private String mode;

    @NotBlank(message = "startPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "startPlaceLatitude内容不正确")
    @Schema(description = "订单起点的纬度")
    private String startPlaceLatitude;

    @NotBlank(message = "startPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "startPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String startPlaceLongitude;

    @NotBlank(message = "endPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "endPlaceLatitude内容不正确")
    @Schema(description = "订单终点的纬度")
    private String endPlaceLatitude;

    @NotBlank(message = "endPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "endPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String endPlaceLongitude;
}

// 调用已经写好的预估里程
@FeignClient(value = "hxds-mps")
public interface MpsServiceApi {
    @PostMapping("/map/estimateOrderMileageAndMinute")
    public R estimateOrderMileageAndMinute(EstimateOrderMileageAndMinuteForm form);
}
```

#### 4.规则引擎预估金额

由于每笔代驾都要记录订单的预估金额，所以我们要根据里程和计费规则，预估出代驾的费用，这就需要用到规则引擎。

**为什么要使用规则引擎？**

例如遇到雨雪天气，代驾费用就得上调一些。如果是业务淡季，代驾费用可以下调一点。既然代驾费的算法经常要变动，我们肯定不能把算法写死到程序里面。我们要把算法从程序中抽离，保存到MySQL里面。将来我们要改动计费算法，直接添加一个新纪录就行了，原有记录不需要删改，系统默认使用最新的计费方式。本项目选用的规则引擎是带有阿里血统的QLExpress，作为一个嵌入式规则引擎在业务系统中使用，让业务规则定义简便而不失灵活。QLExpress支持标准的JAVA语法，还可以支持自定义操作符号、操作符号重载、函数定义、宏定义、数据延迟加载、线程安全。

数据库中的计费规则数据表如下所示：在hxds_rule逻辑库的tb_charge_rule表中，保存的是计费规则，也就是程序中剥离的算法。

|字段|类型|非空|描述信息|
|--|--|--|--|
|id |	bigint| 	True |	主键ID|
|code| 	varchar| 	True |	规则编码|
|name| 	varchar| 	True |	规则名称|
|rule| 	text |	True |	规则代码|
|type| 	tinyint |	True |	1司机取消规则，2乘客取消规则|
|status| 	tinyint |	True |	状态代码，1有效，2关闭|
|create_time| 	datetime |	True |	添加时间|

具体的记录如下，其中rule字段是具体的算法。但是为了避免规则被数据库管理员随便改动，我们要对rule字段加密，这样外界就没法随便修改规则了，必须由专业人员来维护。

在hxds-rule子系统中，ChargeRuleController中定义了estimateOrderCharge()方法用来预估代驾费费用，返回的结果绑定在R对象中。

```java
@RestController
@RequestMapping("/charge")
@Tag(name = "ChargeRuleController", description = "代驾费用的Web接口")
public class ChargeRuleController{

    @PostMapping("/estimateOrderCharge")
    public R estimateOrderCharge(@RequestBody @Valid EstimateOrderChargeForm form) {
        HashMap map = chargeRuleService.calculateOrderCharge(form.getMileage(), form.getTime(), 0, key);
        return R.ok().put("result", map);
    }
}
```

调用该Web方法需要我们传入的参数有两个：里程（公里）和下单时间（HH:mm:ss）
```java
@Data
@Schema(description = "预估代驾费用的表单")
public class EstimateOrderChargeForm {
    @NotBlank(message = "mileage不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d+$|^0\\.\\d*[1-9]\\d*$|^[1-9]\\d*$", message = "mileage内容不正确")
    @Schema(description = "代驾公里数")
    private String mileage;

    @NotBlank(message = "time不能为空")
    @Pattern(regexp = "^(20|21|22|23|[0-1]\\d):[0-5]\\d:[0-5]\\d$", message = "time内容不正确")
    @Schema(description = "代驾开始时间")
    private String time;
}

```

调用该方法，传入里程为12.5，时间为凌晨1点整时，该web方法返回的结果如下：

```json
{
  "msg": "success",
  "result": {
    "amount": "115.00",  //总金额
    "chargeRuleId": "714601916034166785",  //使用的规则ID
    
    "baseMileage": "8",  //代驾基础里程
    "baseMileagePrice": "85",  // 基础里程费
    "exceedMileagePrice": "3.5",  //超出规定里程后每公里3.5元
    "mileageFee": "102.50",  //本订单里程费
    
    "baseMinute": "10",  //免费等时10分钟
    "exceedMinutePrice": "1.0",  //超出10分钟后，每分钟1元
    "waitingFee": "0.00",  //本订单等时费
    
    "baseReturnMileage": "8",  //总里程超过8公里后，要加收返程费
    "exceedReturnPrice": "1.0",  //返程里程是每公里1元
    "returnMileage": "12.5",  //本订单的返程里程
    "returnFee": "12.50",  //本订单返程费
  },
  "code": 200
}

```

#### 5.计算费用标准

| 时间段                | 基础里程 | 收费 |
| --------------------- | -------- | ---- |
| 06:00 - 22:00         | 8公里    | 35元 |
| 22:00 - 23:00         | 8公里    | 50元 |
| 23:00 - 00:00（次日） | 8公里    | 65元 |
| 00:00 - 06:00         | 8公里    | 85元 |

#### 6.乘客端下单业务逻辑编写

**下单表单**

```java
@Data
@Schema(description = "预估订单数据的表单")
public class CreateNewOrderForm {
    @Schema(description = "客户ID")
    private Long customerId;

    @NotBlank(message = "startPlace不能为空")
    @Pattern(regexp = "[\\(\\)0-9A-Z#\\-_\\u4e00-\\u9fa5]{2,50}", message = "startPlace内容不正确")
    @Schema(description = "订单起点")
    private String startPlace;

    @NotBlank(message = "startPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "startPlaceLatitude内容不正确")
    @Schema(description = "订单起点的纬度")
    private String startPlaceLatitude;

    @NotBlank(message = "startPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "startPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String startPlaceLongitude;

    @NotBlank(message = "endPlace不能为空")
    @Pattern(regexp = "[\\(\\)0-9A-Z#\\-_\\u4e00-\\u9fa5]{2,50}", message = "endPlace内容不正确")
    @Schema(description = "订单终点")
    private String endPlace;

    @NotBlank(message = "endPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "endPlaceLatitude内容不正确")
    @Schema(description = "订单终点的纬度")
    private String endPlaceLatitude;

    @NotBlank(message = "endPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "endPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String endPlaceLongitude;

    @NotBlank(message = "favourFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "favourFee内容不正确")
    @Schema(description = "顾客好处费")
    private String favourFee;

    @NotBlank(message = "carPlate不能为空")
    @Pattern(regexp = "^([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}(([0-9]{5}[DF])|([DF]([A-HJ-NP-Z0-9])[0-9]{4})))|([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}[A-HJ-NP-Z0-9]{4}[A-HJ-NP-Z0-9挂学警港澳]{1})$",
            message = "carPlate内容不正确")
    @Schema(description = "车牌号")
    private String carPlate;

    @NotBlank(message = "carType不能为空")
    @Pattern(regexp = "^[\\u4e00-\\u9fa5A-Za-z0-9\\-\\_\\s]{2,20}$", message = "carType内容不正确")
    @Schema(description = "车型")
    private String carType;
    
}
```

**乘客端下单业务逻辑**


```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    @Resource
    private OdrServiceApi odrServiceApi;

    @Resource
    private MpsServiceApi mpsServiceApi;

    @Resource
    private RuleServiceApi ruleServiceApi;

    @Resource
    private SnmServiceApi snmServiceApi;
    
    @Override
    @Transactional
    @LcnTransaction
    public int createNewOrder(CreateNewOrderForm form) {
        Long customerId = form.getCustomerId();
        String startPlace = form.getStartPlace();
        String startPlaceLatitude = form.getStartPlaceLatitude();
        String startPlaceLongitude = form.getStartPlaceLongitude();
        String endPlace = form.getEndPlace();
        String endPlaceLatitude = form.getEndPlaceLatitude();
        String endPlaceLongitude = form.getEndPlaceLongitude();
        String favourFee = form.getFavourFee();
        /**
         *  在日常生活中，我们经常遇到，在家先尝试下单，输入好起始点，然后系统返回最佳路线和预估车费和预估时间。
         *  但是，走到楼下，我们重新输入起始点或者，在楼下下单的时候，系统返回的车费和时间就变了。
         *	因此，我们需要重新预估里程和时间。
         * 【重新预估里程和时间】
         * 虽然下单前，系统会预估里程和时长，但是有可能顾客在下单页面停留时间过长，
         * 然后再按下单键，这时候路线和时长可能都有变化，所以需要重新预估里程和时间
         */
        EstimateOrderMileageAndMinuteForm form_1 = new EstimateOrderMileageAndMinuteForm();
        form_1.setMode("driving");
        form_1.setStartPlaceLatitude(startPlaceLatitude);
        form_1.setStartPlaceLongitude(startPlaceLongitude);
        form_1.setEndPlaceLatitude(endPlaceLatitude);
        form_1.setEndPlaceLongitude(endPlaceLongitude);
        R r = mpsServiceApi.estimateOrderMileageAndMinute(form_1);
        HashMap map = (HashMap) r.get("result");
        String mileage = MapUtil.getStr(map, "mileage");
        int minute = MapUtil.getInt(map, "minute");

        /**
         * 重新估算订单金额
         */
        EstimateOrderChargeForm form_2 = new EstimateOrderChargeForm();
        form_2.setMileage(mileage);
        form_2.setTime(new DateTime().toTimeStr());
        r = ruleServiceApi.estimateOrderCharge(form_2);
        map = (HashMap) r.get("result");
        String expectsFee = MapUtil.getStr(map, "amount");
        String chargeRuleId = MapUtil.getStr(map, "chargeRuleId");
        short baseMileage = MapUtil.getShort(map, "baseMileage");
        String baseMileagePrice = MapUtil.getStr(map, "baseMileagePrice");
        String exceedMileagePrice = MapUtil.getStr(map, "exceedMileagePrice");
        short baseMinute = MapUtil.getShort(map, "baseMinute");
        String exceedMinutePrice = MapUtil.getStr(map, "exceedMinutePrice");
        short baseReturnMileage = MapUtil.getShort(map, "baseReturnMileage");
        String exceedReturnPrice = MapUtil.getStr(map, "exceedReturnPrice");

        return 0;

    }
}
```

**订单表是`tb_order`，这张数据表的结构如下：**

| **字段**             | **类型** | **非空** | **描述信息**                                                 |
| -------------------- | -------- | -------- | ------------------------------------------------------------ |
| id                   | bigint   | True     | 主键                                                         |
| uuid                 | varchar  | True     | 订单序列号                                                   |
| customer_id          | bigint   | True     | 客户ID                                                       |
| start_place          | varchar  | True     | 起始地点                                                     |
| start_place_location | json     | True     | 起始地点坐标信息                                             |
| end_place            | varchar  | True     | 结束地点                                                     |
| end_place_location   | json     | True     | 结束地点坐标信息                                             |
| expects_mileage      | decimal  | True     | 预估里程                                                     |
| real_mileage         | decimal  | False    | 实际里程                                                     |
| return_mileage       | decimal  | False    | 返程里程                                                     |
| expects_fee          | decimal  | True     | 预估订单价格                                                 |
| favour_fee           | decimal  | True     | 顾客好处费                                                   |
| incentive_fee        | decimal  | False    | 系统奖励费                                                   |
| real_fee             | decimal  | False    | 实际订单价格                                                 |
| driver_id            | bigint   | False    | 司机ID                                                       |
| date                 | date     | True     | 订单日期                                                     |
| create_time          | datetime | True     | 订单创建时间                                                 |
| accept_time          | datetime | False    | 司机接单时间                                                 |
| arrive_time          | datetime | False    | 司机到达时间                                                 |
| start_time           | datetime | False    | 代驾开始时间                                                 |
| end_time             | datetime | False    | 代驾结束时间                                                 |
| waiting_minute       | smallint | False    | 代驾等时分钟数                                               |
| prepay_id            | varchar  | False    | 微信预支付单ID                                               |
| pay_id               | varchar  | False    | 微信支付单ID                                                 |
| pay_time             | datetime | False    | 微信付款时间                                                 |
| charge_rule_id       | bigint   | True     | 费用规则ID                                                   |
| cancel_rule_id       | bigint   | False    | 订单取消规则ID                                               |
| car_plate            | varchar  | True     | 车牌号                                                       |
| car_type             | varchar  | True     | 车型                                                         |
| status               | tinyint  | True     | 1等待接单，2已接单，3司机已到达，4开始代驾，5结束代驾，6未付款，7已付款，8订单已结束，9顾客撤单，10司机撤单，11事故关闭，12其他 |
| remark               | varchar  | False    | 订单备注信息                                                 |

**账单表是`tb_order_bill`，这个表的结构如下：**

| **字段**             | **类型** | **非空** | **描述信息**           |
| -------------------- | -------- | -------- | ---------------------- |
| id                   | bigint   | True     | 主键                   |
| order_id             | bigint   | True     | 订单ID                 |
| total                | decimal  | False    | 总金额                 |
| real_pay             | decimal  | False    | 实付款金额             |
| mileage_fee          | decimal  | False    | 里程费                 |
| tel_fee              | decimal  | False    | 通话费                 |
| waiting_fee          | decimal  | False    | 等时费用               |
| toll_fee             | decimal  | False    | 路桥费                 |
| parking_fee          | decimal  | False    | 停车费                 |
| other_free           | decimal  | False    | 其他费用               |
| return_fee           | decimal  | False    | 返程费                 |
| favour_fee           | decimal  | False    | 顾客好处费             |
| incentive_fee        | decimal  | False    | 系统奖励费             |
| voucher_fee          | decimal  | False    | 代金券面额             |
| detail               | json     | False    | 详情                   |
| base_mileage         | smallint | True     | 基础里程（公里）       |
| base_mileage_price   | decimal  | True     | 基础里程价格           |
| exceed_mileage_price | decimal  | True     | 超出基础里程的价格     |
| base_minute          | smallint | True     | 基础分钟               |
| exceed_minute_price  | decimal  | True     | 超出基础分钟的价格     |
| base_return_mileage  | smallint | True     | 基础返程里程（公里）   |
| exceed_return_price  | decimal  | True     | 超出基础返程里程的价格 |

#### 7.插入订单表和账单表

**顾客下单的表单**

```java
@Data
@Schema(description = "顾客下单的表单")
public class InsertOrderForm {
    @NotBlank(message = "uuid不能为空")
    @Pattern(regexp = "^[0-9A-Za-z]{32}$", message = "uuid内容不正确")
    @Schema(description = "uuid")
    private String uuid;

    @NotNull(message = "customerId不能为空")
    @Min(value = 1, message = "customerId不能小于1")
    @Schema(description = "客户ID")
    private Long customerId;

    @NotBlank(message = "startPlace不能为空")
    @Pattern(regexp = "[\\(\\)0-9A-Z#\\-_\\u4e00-\\u9fa5]{2,50}", message = "startPlace内容不正确")
    @Schema(description = "订单起点")
    private String startPlace;

    @NotBlank(message = "startPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "startPlaceLatitude内容不正确")
    @Schema(description = "订单起点的纬度")
    private String startPlaceLatitude;

    @NotBlank(message = "startPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "startPlaceLongitude内容不正确")
    @Schema(description = "订单起点的经度")
    private String startPlaceLongitude;

    @NotBlank(message = "endPlace不能为空")
    @Pattern(regexp = "[\\(\\)0-9A-Z#\\-_\\u4e00-\\u9fa5]{2,50}", message = "endPlace内容不正确")
    @Schema(description = "订单终点")
    private String endPlace;

    @NotBlank(message = "endPlaceLatitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "endPlaceLatitude内容不正确")
    @Schema(description = "订单终点的纬度")
    private String endPlaceLatitude;

    @NotBlank(message = "endPlaceLongitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "endPlaceLongitude内容不正确")
    @Schema(description = "订单终点的经度")
    private String endPlaceLongitude;

    @NotBlank(message = "expectsMileage不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d+$|^0\\.\\d*[1-9]\\d*$|^[1-9]\\d*$", message = "expectsMileage内容不正确")
    @Schema(description = "预估代驾公里数")
    private String expectsMileage;

    @NotBlank(message = "expectsFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "expectsFee内容不正确")
    @Schema(description = "预估代驾费用")
    private String expectsFee;

    @NotBlank(message = "favourFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "favourFee内容不正确")
    @Schema(description = "顾客好处费")
    private String favourFee;


    @NotNull(message = "chargeRuleId不能为空")
    @Min(value = 1, message = "chargeRuleId不能小于1")
    @Schema(description = "规则ID")
    private Long chargeRuleId;

    @NotBlank(message = "carPlate不能为空")
    @Pattern(regexp = "^([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}(([0-9]{5}[DF])|([DF]([A-HJ-NP-Z0-9])[0-9]{4})))|([京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领A-Z]{1}[A-Z]{1}[A-HJ-NP-Z0-9]{4}[A-HJ-NP-Z0-9挂学警港澳]{1})$",
            message = "carPlate内容不正确")
    @Schema(description = "车牌号")
    private String carPlate;

    @NotBlank(message = "carType不能为空")
    @Pattern(regexp = "^[\\u4e00-\\u9fa5A-Za-z0-9\\-\\_\\s]{2,20}$", message = "carType内容不正确")
    @Schema(description = "车型")
    private String carType;

    @NotBlank(message = "date不能为空")
    @Pattern(regexp = "^((((1[6-9]|[2-9]\\d)\\d{2})-(0?[13578]|1[02])-(0?[1-9]|[12]\\d|3[01]))|(((1[6-9]|[2-9]\\d)\\d{2})-(0?[13456789]|1[012])-(0?[1-9]|[12]\\d|30))|(((1[6-9]|[2-9]\\d)\\d{2})-0?2-(0?[1-9]|1\\d|2[0-8]))|(((1[6-9]|[2-9]\\d)(0[48]|[2468][048]|[13579][26])|((16|[2468][048]|[3579][26])00))-0?2-29-))$", message = "date内容不正确")
    @Schema(description = "日期")
    private String date;

    @NotNull(message = "baseMileage不能为空")
    @Min(value = 1, message = "baseMileage不能小于1")
    @Schema(description = "基础里程（公里）")
    private Short baseMileage;

    @NotBlank(message = "baseMileagePrice不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "baseMileagePrice内容不正确")
    @Schema(description = "基础里程价格")
    private String baseMileagePrice;

    @NotBlank(message = "exceedMileagePrice不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "exceedMileagePrice内容不正确")
    @Schema(description = "超出基础里程的价格")
    private String exceedMileagePrice;

    @NotNull(message = "baseMinute不能为空")
    @Min(value = 1, message = "baseMinute不能小于1")
    @Schema(description = "基础分钟")
    private Short baseMinute;

    @NotBlank(message = "exceedMinutePrice不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "exceedMinutePrice内容不正确")
    @Schema(description = "超出基础分钟的价格")
    private String exceedMinutePrice;

    @NotNull(message = "baseReturnMileage不能为空")
    @Min(value = 1, message = "baseReturnMileage不能小于1")
    @Schema(description = "基础返程里程（公里）")
    private Short baseReturnMileage;

    @NotBlank(message = "exceedReturnPrice不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "exceedReturnPrice内容不正确")
    @Schema(description = "超出基础返程里程的价格")
    private String exceedReturnPrice;

}
```

**顾客下单**

```java
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    @PostMapping("/insertOrder")
    @Operation(summary = "顾客下单")
    public R insertOrder(@RequestBody @Valid InsertOrderForm form) {
        OrderEntity orderEntity = new OrderEntity();
        orderEntity.setUuid(form.getUuid());
        orderEntity.setCustomerId(form.getCustomerId());
        orderEntity.setStartPlace(form.getStartPlace());
        JSONObject json = new JSONObject();
        json.set("latitude", form.getStartPlaceLatitude());
        json.set("longitude", form.getStartPlaceLongitude());
        orderEntity.setStartPlaceLocation(json.toString());
        orderEntity.setEndPlace(form.getEndPlace());
        json = new JSONObject();
        json.set("latitude", form.getEndPlaceLatitude());
        json.set("longitude", form.getEndPlaceLongitude());
        orderEntity.setEndPlaceLocation(json.toString());
        orderEntity.setExpectsMileage(new BigDecimal(form.getExpectsMileage()));
        orderEntity.setExpectsFee(new BigDecimal(form.getExpectsFee()));
        orderEntity.setFavourFee(new BigDecimal(form.getFavourFee()));
        orderEntity.setChargeRuleId(form.getChargeRuleId());
        orderEntity.setCarPlate(form.getCarPlate());
        orderEntity.setCarType(form.getCarType());
        orderEntity.setDate(form.getDate());

        OrderBillEntity billEntity = new OrderBillEntity();
        billEntity.setBaseMileage(form.getBaseMileage());
        billEntity.setBaseMileagePrice(new BigDecimal(form.getBaseMileagePrice()));
        billEntity.setExceedMileagePrice(new BigDecimal(form.getExceedMileagePrice()));
        billEntity.setBaseMinute(form.getBaseMinute());
        billEntity.setExceedMinutePrice(new BigDecimal(form.getExceedMinutePrice()));
        billEntity.setBaseReturnMileage(form.getBaseReturnMileage());
        billEntity.setExceedReturnPrice(new BigDecimal(form.getExceedReturnPrice()));

        String id = orderService.insert(orderEntity, billEntity);
        return R.ok().put("result", id);
    }
}
```

#### 8.司机接单

创建订单后，要寻找附近适合接单的司机。司机端的小程序实时将司机的GPS定位上传，然后将定位信息缓存到Redis中，利用Redis的Geo功能（主要用于存储地理位置信息，并对存储信息进行操作），GEO是redis3.2以后才有的功能。可使用如下命令添加地理位置信息和查询多少半径内的地理内容。

```
GEOADD china 116.403963 39.915119 tiananmen 116.417876 39.915411 wangfujing 116.404354 39.904748 qianmen
```
然后我们`GEORADIUS`命令查询距离某个定位点1公里范围以内的景点有哪些。

```
GEORADIUS china 116.4000 39.9000 1 km WITHDIST
```

Redis的GEO命令可以帮我们提取出某个坐标点指定距离以内的景点，如果Redis里面缓存的是司机的定位信息，那么我们用代驾单的起点坐标来查询附近几公里以内的司机。

##### 1.司机定位缓存

**有个需要特别注意的问题就是Geo中的定位我们没办法设置超时时间，所以我们想要知道哪些司机上线中，就得额外创建带有超时的缓存。我的设想是缓存司机信息用的Key是`driver_online#driverId`，对应的Value是`接单距离#订单里程范围#定向接单的坐标`，超时时间为1分钟。当系统接到订单之后，到Redis上面根据driverId查找缓存，找到了就是在线，找不到就是不在线。**

```java
// 首先把司机定位信息缓存到GEO，然后额外创建司机上线缓存。
@Service
public class DriverLocationServiceImpl implements DriverLocationService {
    @Resource
    private RedisTemplate redisTemplate;

    @Override
    public void updateLocationCache(Map param) {
        long driverId = MapUtil.getLong(param, "driverId");
        String latitude = MapUtil.getStr(param, "latitude");
        String longitude = MapUtil.getStr(param, "longitude");

        //接单范围
        int rangeDistance = MapUtil.getInt(param, "rangeDistance");
        //订单里程范围
        int orderDistance = MapUtil.getInt(param, "orderDistance");
        //当前司机坐标，封装成Point对象才能缓存到Redis里面
        Point point = new Point(Convert.toDouble(longitude), Convert.toDouble(latitude));
        /*
         * 把司机实时定位缓存到Redis里面，便于Geo定位计算
         * Geo是集合形式，如果设置过期时间，所有司机的定位缓存就全都失效了
         * 正确做法是司机上线后，更新GEO中的缓存定位
         */
        redisTemplate.opsForGeo().add("driver_location", point, driverId + "");

        //定向接单地址的经度
        String orientateLongitude = null;
        if (param.get("orientateLongitude") != null) {
            orientateLongitude = MapUtil.getStr(param, "orientateLongitude");
        }
        //定向接单地址的纬度
        String orientateLatitude = null;
        if (param.get("orientateLatitude") != null) {
            orientateLatitude = MapUtil.getStr(param, "orientateLatitude");
        }
        //定向接单经纬度的字符串
        String orientation = "none";
        if (orientateLongitude != null && orientateLatitude != null) {
            orientation = orientateLatitude + "," + orientateLongitude;
        }

        /*
        * 为了解决判断哪些司机在线，我们还要单独弄一个上线缓存
        * 缓存司机的接单设置（定向接单、接单范围、订单总里程），便于系统判断该司机是否符合接单条件
         */
        String temp = rangeDistance + "#" + orderDistance + "#" + orientation;
        redisTemplate.opsForValue().set("driver_online#" + driverId, temp, 60, TimeUnit.SECONDS);
    }

    @Override
    public void removeLocationCache(long driverId) {
        //删除司机定位缓存
        redisTemplate.opsForGeo().remove("driver_location", driverId + "");
        //删除司机上线缓存
        redisTemplate.delete("driver_online#" + driverId);
    }
}
```

##### 2.上传司机实时GPS定位

前端微信小程序GPS后台刷新代码以及GPS定位变化后自动提交后端的代码如下：

```js
onLaunch: function() {
    let gps = [];
    //保持屏幕常亮，避免手机休眠
    wx.setKeepScreenOn({
        keepScreenOn: true
    });

    //TODO 每隔3分钟触发自定义事件，接受系统消息
    
    //开启GPS后台刷新
    uni.startLocationUpdate({
        success(resp) {
            console.log('开启定位成功');
        },
        fail(resp) {
            console.log('开启定位失败');
            uni.$emit('updateLocation', null);
        }
    });
  
    //GPS定位变化就自动提交给后端
    wx.onLocationChange(function(resp) {
        let latitude = resp.latitude;
        let longitude = resp.longitude;
        let speed = resp.speed;
        // console.log(resp)
        let location = { latitude: latitude, longitude: longitude };

        
        let workStatus = uni.getStorageSync('workStatus');
        //TODO 先暂时写死，以后要去掉这句话
        workStatus = '开始接单'
        /*
         * 上传司机GPS定位信息
         */
        let baseUrl = 'http://192.168.99.106:8201/hxds-driver';
        if (workStatus == '开始接单') {
            // TODO 只在每分钟的前10秒上报定位信息，减小服务器压力
            // let current = new Date();
            // if (current.getSeconds() > 10) {
            // 	return;
            // }
            let settings = uni.getStorageSync('settings');
            //TODO 先暂时写死，以后要去掉这句话
            settings = {
                orderDistance: 0,
                rangeDistance: 5,
                orientation: ''
            }
            let orderDistance = settings.orderDistance;
            let rangeDistance = settings.rangeDistance;
            let orientation = settings.orientation;
            uni.request({
                url: `${baseUrl}/driver/location/updateLocationCache`,
                method: 'POST',
                header: {
                    token: uni.getStorageSync('token')
                },
                data: {
                    latitude: latitude,
                    longitude: longitude,
                    orderDistance: orderDistance,
                    rangeDistance: rangeDistance,
                    orientateLongitude: orientation != '' ? orientation.longitude : null,
                    orientateLatitude: orientation != '' ? orientation.latitude : null
                },
                success: function(resp) {
                    if (resp.statusCode == 401) {
                        uni.redirectTo({
                            url: '/pages/login/login'
                        });
                    } else if (resp.statusCode == 200 && resp.data.code == 200) {
                        let data = resp.data;
                        if (data.hasOwnProperty('token')) {
                            let token = data.token;
                            uni.setStorageSync('token', token);
                        }
                        console.log("定位更新成功")
                    } else {
                        console.error('更新GPS定位信息失败', resp.data);
                    }
                },
                fail: function(error) {
                    console.error('更新GPS定位信息失败', error);
                }
            });
        } else if (workStatus == '开始代驾') {
            //TODO 每凑够20个定位就上传一次，减少服务器的压力
        }

        //触发自定义事件
        uni.$emit('updateLocation', location);
    });
},

```

##### 3.搜索5公里内的司机

创建订单的过程中，我们要查找附近适合接单的司机。如果有这样的司机，代驾系统才会创建订单，否则就拒绝创建订单。

```java
@Service
public class DriverLocationServiceImpl implements DriverLocationService {
    ……
        
    @Override
    public ArrayList searchBefittingDriverAboutOrder(double startPlaceLatitude, 
                                                     double startPlaceLongitude,
                                                     double endPlaceLatitude, 
                                                     double endPlaceLongitude, 
                                                     double mileage) {
        //搜索订单起始点5公里以内的司机
        // 将用户起始点封装为左标
        Point point = new Point(startPlaceLongitude, startPlaceLatitude);
        //设置GEO距离单位为千米
        Metric metric = RedisGeoCommands.DistanceUnit.KILOMETERS;
        Distance distance = new Distance(5, metric);
        // 设置圆，以左标为中心，半径为5公里
        Circle circle = new Circle(point, distance);

        //创建GEO参数
        RedisGeoCommands.GeoRadiusCommandArgs args = RedisGeoCommands
                .GeoRadiusCommandArgs
                .newGeoRadiusArgs() // radius函数
                .includeDistance() //结果中包含距离
                .includeCoordinates() //结果中包含坐标
                .sortAscending(); //升序排列

        // 执行GEO计算，获得查询结果
        GeoResults<RedisGeoCommands.GeoLocation<String>> radius = redisTemplate.opsForGeo()
                .radius("driver_location", circle, args);
        // 需要通知的司机列表
        ArrayList list = new ArrayList(); 

        if (radius != null) {
            // 遍历
            Iterator<GeoResult<RedisGeoCommands.GeoLocation<String>>> iterator = radius.iterator();
            while (iterator.hasNext()) {
                GeoResult<RedisGeoCommands.GeoLocation<String>> result = iterator.next();
                RedisGeoCommands.GeoLocation<String> content = result.getContent();
                String driverId = content.getName();
                //Point memberPoint = content.getPoint(); // 对应的经纬度坐标
                double dist = result.getDistance().getValue(); // 距离中心点的距离

                //排查掉不在线的司机
                if (!redisTemplate.hasKey("driver_online#" + driverId)) {
                    continue;
                }

                //查找该司机的在线缓存
                Object obj = redisTemplate.opsForValue().get("driver_online#" + driverId);
                //如果查找的那一刻，缓存超时被置空，那么就忽略该司机
                if (obj == null) {
                    continue;
                }

                String value = obj.toString();
                String[] temp = value.split("#");
                int rangeDistance = Integer.parseInt(temp[0]);
                int orderDistance = Integer.parseInt(temp[1]);
                String orientation = temp[2];

                //判断是否符合接单范围
                boolean bool_1 = dist <= rangeDistance;

                //判断订单里程是否符合
                boolean bool_2 = false;
                if (orderDistance == 0) {
                    bool_2 = true;
                } else if (orderDistance == 5 && mileage > 0 && mileage <= 5) {
                    bool_2 = true;
                } else if (orderDistance == 10 && mileage > 5 && mileage <= 10) {
                    bool_2 = true;
                } else if (orderDistance == 15 && mileage > 10 && mileage <= 15) {
                    bool_2 = true;
                } else if (orderDistance == 30 && mileage > 15 && mileage <= 30) {
                    bool_2 = true;
                }

                //判断定向接单是否符合
                boolean bool_3 = false;
                if (!orientation.equals("none")) {
                    double orientationLatitude = Double.parseDouble(orientation.split(",")[0]);
                    double orientationLongitude = Double.parseDouble(orientation.split(",")[1]);
                    //把定向点的火星坐标转换成GPS坐标
                    double[] location = CoordinateTransform.transformGCJ02ToWGS84(orientationLongitude, orientationLatitude);
                    GlobalCoordinates point_1 = new GlobalCoordinates(location[1], location[0]);
                    //把订单终点的火星坐标转换成GPS坐标
                    location = CoordinateTransform.transformGCJ02ToWGS84(endPlaceLongitude, endPlaceLatitude);
                    GlobalCoordinates point_2 = new GlobalCoordinates(location[1], location[0]);
                    //这里不需要Redis的GEO计算，直接用封装函数计算两个GPS坐标之间的距离
                    GeodeticCurve geoCurve = new GeodeticCalculator().calculateGeodeticCurve(Ellipsoid.WGS84, point_1, point_2);

                    //如果定向点距离订单终点距离在3公里以内，说明这个订单和司机定向点是顺路的
                    if (geoCurve.getEllipsoidalDistance() <= 3000) {
                        bool_3 = true;
                    }

                } else {
                    bool_3 = true;
                }
                
                //匹配接单条件
                if (bool_1 && bool_2 && bool_3) {
                    HashMap map = new HashMap() {{
                        put("driverId", driverId);
                        put("distance", dist);
                    }};
                    list.add(map); //把该司机添加到需要通知的列表中
                }
            }
        }
        return list;
    }
}
```

##### 4.修改下单逻辑

```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    ……
        
    @Override
    @Transactional
    @LcnTransaction
    public HashMap createNewOrder(CreateNewOrderForm form) {
        Long customerId = form.getCustomerId();
        String startPlace = form.getStartPlace();
        String startPlaceLatitude = form.getStartPlaceLatitude();
        String startPlaceLongitude = form.getStartPlaceLongitude();
        String endPlace = form.getEndPlace();
        String endPlaceLatitude = form.getEndPlaceLatitude();
        String endPlaceLongitude = form.getEndPlaceLongitude();
        String favourFee = form.getFavourFee();
        /**
         *  在日常生活中，我们经常遇到，在家先尝试下单，输入好起始点，然后系统返回最佳路线和预估车费和预估时间。
         *  但是，走到楼下，我们重新输入起始点或者，在楼下下单的时候，系统返回的车费和时间就变了。
         *	因此，我们需要重新预估里程和时间。
         * 【重新预估里程和时间】
         * 虽然下单前，系统会预估里程和时长，但是有可能顾客在下单页面停留时间过长，
         * 然后再按下单键，这时候路线和时长可能都有变化，所以需要重新预估里程和时间
         */
        EstimateOrderMileageAndMinuteForm form_1 = new EstimateOrderMileageAndMinuteForm();
        form_1.setMode("driving");
        form_1.setStartPlaceLatitude(startPlaceLatitude);
        form_1.setStartPlaceLongitude(startPlaceLongitude);
        form_1.setEndPlaceLatitude(endPlaceLatitude);
        form_1.setEndPlaceLongitude(endPlaceLongitude);
        R r = mpsServiceApi.estimateOrderMileageAndMinute(form_1);
        HashMap map = (HashMap) r.get("result");
        String mileage = MapUtil.getStr(map, "mileage");
        int minute = MapUtil.getInt(map, "minute");

        /**
         * 重新估算订单金额
         */
        EstimateOrderChargeForm form_2 = new EstimateOrderChargeForm();
        form_2.setMileage(mileage);
        form_2.setTime(new DateTime().toTimeStr());
        r = ruleServiceApi.estimateOrderCharge(form_2);
        map = (HashMap) r.get("result");
        String expectsFee = MapUtil.getStr(map, "amount");
        String chargeRuleId = MapUtil.getStr(map, "chargeRuleId");
        short baseMileage = MapUtil.getShort(map, "baseMileage");
        String baseMileagePrice = MapUtil.getStr(map, "baseMileagePrice");
        String exceedMileagePrice = MapUtil.getStr(map, "exceedMileagePrice");
        short baseMinute = MapUtil.getShort(map, "baseMinute");
        String exceedMinutePrice = MapUtil.getStr(map, "exceedMinutePrice");
        short baseReturnMileage = MapUtil.getShort(map, "baseReturnMileage");
        String exceedReturnPrice = MapUtil.getStr(map, "exceedReturnPrice");
            
        /*
         * 搜索适合接单的司机
         */
        SearchBefittingDriverAboutOrderForm form_3 = new SearchBefittingDriverAboutOrderForm();
        form_3.setStartPlaceLatitude(startPlaceLatitude);
        form_3.setStartPlaceLongitude(startPlaceLongitude);
        form_3.setEndPlaceLatitude(endPlaceLatitude);
        form_3.setEndPlaceLongitude(endPlaceLongitude);
        form_3.setMileage(mileage);
        r = mpsServiceApi.searchBefittingDriverAboutOrder(form_3);
        ArrayList<HashMap> list = (ArrayList<HashMap>) r.get("result");
        
        HashMap result=new HashMap(){{
            put("count",0);
        }};
        if (list.size() > 0) {
            /*
             * 生成订单记录
             */
            InsertOrderForm form_4 = new InsertOrderForm();
            form_4.setUuid(IdUtil.simpleUUID());
            form_4.setCustomerId(customerId);
            form_4.setStartPlace(startPlace);
            form_4.setStartPlaceLatitude(startPlaceLatitude);
            form_4.setStartPlaceLongitude(startPlaceLongitude);
            form_4.setEndPlace(endPlace);
            form_4.setEndPlaceLatitude(endPlaceLatitude);
            form_4.setEndPlaceLongitude(endPlaceLongitude);
            form_4.setExpectsMileage(mileage);
            form_4.setExpectsFee(expectsFee);
            form_4.setFavourFee(favourFee);
            form_4.setDate(new DateTime().toDateStr());
            form_4.setChargeRuleId(Long.parseLong(chargeRuleId));
            form_4.setCarPlate(form.getCarPlate());
            form_4.setCarType(form.getCarType());
            form_4.setBaseMileage(baseMileage);
            form_4.setBaseMileagePrice(baseMileagePrice);
            form_4.setExceedMileagePrice(exceedMileagePrice);
            form_4.setBaseMinute(baseMinute);
            form_4.setExceedMinutePrice(exceedMinutePrice);
            form_4.setBaseReturnMileage(baseReturnMileage);
            form_4.setExceedReturnPrice(exceedReturnPrice);

            r = odrServiceApi.insert(form_4);
            String orderId = MapUtil.getStr(r, "result");

            //TODO  发送订单消息给相关的司机
            
            result.put("orderId",orderId);
            result.replace("count",list.size());
        } 
        return result;
    }
}
```

### 三、消息收发

有很多人拿RabbitMQ和Kafka做对比，其实仅仅考虑性能还是不够的，我们还要权衡业务场景。例如RabbitMQ具有对消息的过滤功能，而Kafka则不能对Topic中的消息做过滤。也就是说消费者要接受该Topic中所有的消息。RabbitMQ允许我们对消息设置TTL过期时间，如果超过TTL时间，那么RabbitMQ会自动销毁队列或者主题中的消息。这个功能特别实用，例如一个用户10年前注册了微博帐户，在这10年中，微博会给该用户发送非常多的公告通知，但是该用户迟迟不上线。想这样的幽灵用户，微博系统中还有上百万。你说微博系统有必要把这些幽灵用户的公告消息永久保存吗？没必要是吧，所以我们用RabbitMQ缓存公告消息，超过1年就自动销毁。这样微博系统就不用长期持久保存那些幽灵用户的公告通知了。

#### 1.RabbitMQ的六种模式

简单模式：生产者-》队列-》消费者，完全是一对一的关系

工作队列模式：生产者-》消息队列-》多个消费者，但是消费者是竞争关系，每条消息只能被一个消费者消费

发布订阅模式：该模式需要使用到交换机，交换机与消息队列绑定，一个消息可以被多个消费者接收到，交换机可以控制消息到底是发送给所有绑定的队列，还是发送给特定的队列。我们自己是可以设置的。 生产者-》交换机-》多个消息队列-》每个消息队列又有多个消费者。交换机有多个类型，我们选择Direct类型。Fanout：广播类型，Direct：定向，把消息交给符合指定routing key的队列，Topic：通配符，把消息交给routing pattern（路由模式）的队列。

路由模式：这是发布订阅模式的增强版，我们可以把多个routing key分配给同一个消息队列，只要其中的一个routing key满足，交换机就会转发消息。

通配符模式：这是路由模式的增强版，我们给routing key设置了通配符。只要符合某个通配符，交换机就会转发消息。例如a.#这个消息就会被转发给两个消息队列，如果是b.#这个消息只能发送给消息队列2，这就是通配符的效果。

RPC模式：即客户端远程调用服务端的方法，使用MQ可以实现RPC的异步调用，基于Direct交换机实现，流程如下：
1. 客户端即生产者就是消费者，向RPC请求队列发送RPC调用信息，同时监听RPC响应队列。
2. 服务端监听RPC请求队列的消息，收到消息后执行服务端的方法，得到方法返回的结果
3. 服务端将RPC方法的结果发送到RPC响应队列
4. 客户端（RPC调用方）监听RPC响应队列，接收到RPC调用结果

#### 2.本项目的订单队列设计方案

本项目中使用RabbitMQ采用的是发布订阅模式，每个司机都有自己的消息队列，绑定的名字就是司机的driverId，当我们要发送抢单消息给某个司机的时候，交换机会根据driverId把消息路由给对应的消息队列。本项目用阻塞式来接收RabbitMQ的消息，阻塞式顾名思义就是Java没收发完消息，绝对不往下执行其他代码。直到收完消息，然后把消息打包成R对象返回给移动端。发送新订单消息给适合接单的司机，我倾向于是用异步发送的方式。这是因为有可能附近适合接单的司机比较多，Java程序给这些司机的队列发送消息可能需要一定的耗时，这就会导致createNewOrder()执行时间太长，乘客端迟迟得不到响应，也不知道订单创建成功没有。如果采用异步发送消息就好多了，createNewOrder()函数把发送新订单消息的任务委派给某个空闲线程，自己可以继续往下执行，这样就不会让乘客端小程序等待太长时间，用户体验更好。

新订单消息代码如下：

```java
@Data
public class NewOrderMessage {
    // 用户id
    private String userId;
    // 订单id
    private String orderId;
    // 起点
    private String from;
    // 终点
    private String to;
    // 预估费用
    private String expectsFee;
    // 里程
    private String mileage;
    // 时长
    private String minute;
    // 距离
    private String distance;
    // 乘客优惠费用
    private String favourFee;

}
```

消息发送代码如下：

```java
@Component
@Slf4j
public class NewOrderMassageTask {
    @Resource
    private ConnectionFactory factory;

    /**
     * 同步发送新订单消息
     */
    public void sendNewOrderMessage(ArrayList<NewOrderMessage> list) {
        int ttl = 1 * 60 * 1000; //新订单消息缓存过期时间1分钟
        String exchangeName = "new_order_private"; //交换机的名字
        try (
                Connection connection = factory.newConnection();
                Channel channel = connection.createChannel();
        ) {
            //定义交换机，根据routing key路由消息
            channel.exchangeDeclare(exchangeName, BuiltinExchangeType.DIRECT);
            HashMap param = new HashMap();
            for (NewOrderMessage message : list) {
                //MQ消息的属性信息
                HashMap map = new HashMap();
                map.put("orderId", message.getOrderId());
                map.put("from", message.getFrom());
                map.put("to", message.getTo());
                map.put("expectsFee", message.getExpectsFee());
                map.put("mileage", message.getMileage());
                map.put("minute", message.getMinute());
                map.put("distance", message.getDistance());
                map.put("favourFee", message.getFavourFee());
                //创建消息属性对象
                AMQP.BasicProperties properties = new AMQP.BasicProperties().builder().contentEncoding("UTF-8")
                        .headers(map).expiration(ttl + "").build();

                String queueName = "queue_" + message.getUserId(); //队列名字
                String routingKey = message.getUserId(); //routing key
                //声明队列（持久化缓存消息，消息接收不加锁，消息全部接收完并不删除队列）
                channel.queueDeclare(queueName, true, false, false, param);
                channel.queueBind(queueName,exchangeName,routingKey);
                //向交换机发送消息，并附带routing key
                channel.basicPublish(exchangeName, routingKey, properties, ("新订单" + message.getOrderId()).getBytes());
                log.debug(message.getUserId() + "的新订单消息发送成功");
            }

        } catch (Exception e) {
            log.error("执行异常", e);
            throw new HxdsException("新订单消息发送失败");
        }
    }

    /**
     * 异步发送新订单消息
     */
    @Async
    public void sendNewOrderMessageAsync(ArrayList<NewOrderMessage> list) {
        sendNewOrderMessage(list);
    }
}

```

消息接收必须使用同步方式

```java
@Component
@Slf4j
public class NewOrderMassageTask {
    ……
        
    /**
     * 同步接收新订单消息
     */
    public List<NewOrderMessage> receiveNewOrderMessage(long userId) {
        String exchangeName = "new_order_private"; //交换机名字
        String queueName = "queue_" + userId; //队列名字
        String routingKey = userId + ""; //routing key

        List<NewOrderMessage> list = new ArrayList();
        try (Connection connection = factory.newConnection();
             Channel privateChannel = connection.createChannel();
        ) {
            //定义交换机，routing key模式
            privateChannel.exchangeDeclare(exchangeName, BuiltinExchangeType.DIRECT);
            //声明队列（持久化缓存消息，消息接收不加锁，消息全部接收完并不删除队列）
            privateChannel.queueDeclare(queueName, true, false, false, null);
            //绑定要接收的队列
            privateChannel.queueBind(queueName, exchangeName, routingKey);
            //为了避免一次性接收太多消息，我们采用限流的方式，每次接收10条消息，然后循环接收
            privateChannel.basicQos(0, 10, true);

            while (true) {
                //从队列中接收消息
                GetResponse response = privateChannel.basicGet(queueName, false);
                if (response != null) {
                    //消息属性对象
                    AMQP.BasicProperties properties = response.getProps();
                    Map<String, Object> map = properties.getHeaders();
                    String orderId = MapUtil.getStr(map, "orderId");
                    String from = MapUtil.getStr(map, "from");
                    String to = MapUtil.getStr(map, "to");
                    String expectsFee = MapUtil.getStr(map, "expectsFee");
                    String mileage = MapUtil.getStr(map, "mileage");
                    String minute = MapUtil.getStr(map, "minute");
                    String distance = MapUtil.getStr(map, "distance");
                    String favourFee = MapUtil.getStr(map, "favourFee");

                    //把新订单的消息封装到对象中
                    NewOrderMessage message = new NewOrderMessage();
                    message.setOrderId(orderId);
                    message.setFrom(from);
                    message.setTo(to);
                    message.setExpectsFee(expectsFee);
                    message.setMileage(mileage);
                    message.setMinute(minute);
                    message.setDistance(distance);
                    message.setFavourFee(favourFee);

                    list.add(message);

                    byte[] body = response.getBody();
                    String msg = new String(body);
                    log.debug("从RabbitMQ接收的订单消息：" + msg);

                    //确认收到消息，让MQ删除该消息
                    long deliveryTag = response.getEnvelope().getDeliveryTag();
                    privateChannel.basicAck(deliveryTag, false);
                } else {
                    break;
                }
            }
            ListUtil.reverse(list); //消息倒叙，新消息排在前面
            return list;
        } catch (Exception e) {
            log.error("执行异常", e);
            throw new HxdsException("接收新订单失败");
        }
    }
         
    /**
     * 同步删除新订单消息队列
     */
    public void deleteNewOrderQueue(long userId) {
        String exchangeName = "new_order_private"; //交换机名字
        String queueName = "queue_" + userId; //队列名字
        try (Connection connection = factory.newConnection();
             Channel privateChannel = connection.createChannel();
        ) {
            //定义交换机
            privateChannel.exchangeDeclare(exchangeName, BuiltinExchangeType.DIRECT);
            //删除队列
            privateChannel.queueDelete(queueName);
            log.debug(userId + "的新订单消息队列成功删除");
        } catch (Exception e) {
            log.error(userId + "的新订单队列删除失败", e);
            throw new HxdsException("新订单队列删除失败");
        }
    }

    /**
     * 异步删除新订单消息队列
     */
    @Async
    public void deleteNewOrderQueueAsync(long userId) {
        deleteNewOrderQueue(userId);
    }

    /**
     * 同步清空新订单消息队列
     */
    public void clearNewOrderQueue(long userId) {
        String exchangeName =  "new_order_private";
        String queueName = "queue_" + userId;
        try (Connection connection = factory.newConnection();
             Channel privateChannel = connection.createChannel();
        ) {
            privateChannel.exchangeDeclare(exchangeName, BuiltinExchangeType.DIRECT);
            privateChannel.queuePurge(queueName);
            log.debug(userId + "的新订单消息队列清空删除");
        } catch (Exception e) {
            log.error(userId + "的新订单队列清空失败", e);
            throw new HxdsException("新订单队列清空失败");
        }
    }

    /**
     * 异步清空新订单消息队列
     */
    @Async
    public void clearNewOrderQueueAsync(long userId) {
        clearNewOrderQueue(userId);
    }
}
```

#### 3.修改下单业务逻辑

```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    ……
        
    @Override
    @Transactional
    @LcnTransaction
    public HashMap createNewOrder(CreateNewOrderForm form) {
        Long customerId = form.getCustomerId();
        String startPlace = form.getStartPlace();
        String startPlaceLatitude = form.getStartPlaceLatitude();
        String startPlaceLongitude = form.getStartPlaceLongitude();
        String endPlace = form.getEndPlace();
        String endPlaceLatitude = form.getEndPlaceLatitude();
        String endPlaceLongitude = form.getEndPlaceLongitude();
        String favourFee = form.getFavourFee();
        /**
         *  在日常生活中，我们经常遇到，在家先尝试下单，输入好起始点，然后系统返回最佳路线和预估车费和预估时间。
         *  但是，走到楼下，我们重新输入起始点或者，在楼下下单的时候，系统返回的车费和时间就变了。
         *	因此，我们需要重新预估里程和时间。
         * 【重新预估里程和时间】
         * 虽然下单前，系统会预估里程和时长，但是有可能顾客在下单页面停留时间过长，
         * 然后再按下单键，这时候路线和时长可能都有变化，所以需要重新预估里程和时间
         */
        EstimateOrderMileageAndMinuteForm form_1 = new EstimateOrderMileageAndMinuteForm();
        form_1.setMode("driving");
        form_1.setStartPlaceLatitude(startPlaceLatitude);
        form_1.setStartPlaceLongitude(startPlaceLongitude);
        form_1.setEndPlaceLatitude(endPlaceLatitude);
        form_1.setEndPlaceLongitude(endPlaceLongitude);
        R r = mpsServiceApi.estimateOrderMileageAndMinute(form_1);
        HashMap map = (HashMap) r.get("result");
        String mileage = MapUtil.getStr(map, "mileage");
        int minute = MapUtil.getInt(map, "minute");

        /**
         * 重新估算订单金额
         */
        EstimateOrderChargeForm form_2 = new EstimateOrderChargeForm();
        form_2.setMileage(mileage);
        form_2.setTime(new DateTime().toTimeStr());
        r = ruleServiceApi.estimateOrderCharge(form_2);
        map = (HashMap) r.get("result");
        String expectsFee = MapUtil.getStr(map, "amount");
        String chargeRuleId = MapUtil.getStr(map, "chargeRuleId");
        short baseMileage = MapUtil.getShort(map, "baseMileage");
        String baseMileagePrice = MapUtil.getStr(map, "baseMileagePrice");
        String exceedMileagePrice = MapUtil.getStr(map, "exceedMileagePrice");
        short baseMinute = MapUtil.getShort(map, "baseMinute");
        String exceedMinutePrice = MapUtil.getStr(map, "exceedMinutePrice");
        short baseReturnMileage = MapUtil.getShort(map, "baseReturnMileage");
        String exceedReturnPrice = MapUtil.getStr(map, "exceedReturnPrice");
            
        /*
         * 搜索适合接单的司机
         */
        SearchBefittingDriverAboutOrderForm form_3 = new SearchBefittingDriverAboutOrderForm();
        form_3.setStartPlaceLatitude(startPlaceLatitude);
        form_3.setStartPlaceLongitude(startPlaceLongitude);
        form_3.setEndPlaceLatitude(endPlaceLatitude);
        form_3.setEndPlaceLongitude(endPlaceLongitude);
        form_3.setMileage(mileage);
        r = mpsServiceApi.searchBefittingDriverAboutOrder(form_3);
        ArrayList<HashMap> list = (ArrayList<HashMap>) r.get("result");
        
        HashMap result=new HashMap(){{
            put("count",0);
        }};
        if (list.size() > 0) {
            /*
             * 生成订单记录
             */
            InsertOrderForm form_4 = new InsertOrderForm();
            form_4.setUuid(IdUtil.simpleUUID());
            form_4.setCustomerId(customerId);
            form_4.setStartPlace(startPlace);
            form_4.setStartPlaceLatitude(startPlaceLatitude);
            form_4.setStartPlaceLongitude(startPlaceLongitude);
            form_4.setEndPlace(endPlace);
            form_4.setEndPlaceLatitude(endPlaceLatitude);
            form_4.setEndPlaceLongitude(endPlaceLongitude);
            form_4.setExpectsMileage(mileage);
            form_4.setExpectsFee(expectsFee);
            form_4.setFavourFee(favourFee);
            form_4.setDate(new DateTime().toDateStr());
            form_4.setChargeRuleId(Long.parseLong(chargeRuleId));
            form_4.setCarPlate(form.getCarPlate());
            form_4.setCarType(form.getCarType());
            form_4.setBaseMileage(baseMileage);
            form_4.setBaseMileagePrice(baseMileagePrice);
            form_4.setExceedMileagePrice(exceedMileagePrice);
            form_4.setBaseMinute(baseMinute);
            form_4.setExceedMinutePrice(exceedMinutePrice);
            form_4.setBaseReturnMileage(baseReturnMileage);
            form_4.setExceedReturnPrice(exceedReturnPrice);

            r = odrServiceApi.insert(form_4);
            String orderId = MapUtil.getStr(r, "result");

            /*
             * 发送新订单通知给相关司机
             */
            SendNewOrderMessageForm form_5 = new SendNewOrderMessageForm();
            String[] driverContent = new String[list.size()];
            for (int i = 0; i < list.size(); i++) {
                HashMap one = list.get(i);
                String driverId = MapUtil.getStr(one, "driverId");
                String distance = MapUtil.getStr(one, "distance");
                distance = new BigDecimal(distance).setScale(1, RoundingMode.CEILING).toString();
                driverContent[i] = driverId + "#" + distance;
            }
            form_5.setDriversContent(driverContent);
            form_5.setOrderId(Long.parseLong(orderId));
            form_5.setFrom(startPlace);
            form_5.setTo(endPlace);
            form_5.setExpectsFee(expectsFee);
            //里程转化成保留小数点后一位
            mileage = new BigDecimal(mileage).setScale(1, RoundingMode.CEILING).toString();
            form_5.setMileage(mileage);
            form_5.setMinute(minute);
            form_5.setFavourFee(favourFee);
            snmServiceApi.sendNewOrderMessageAsync(form_5); //异步发送消息
            result.put("orderId",orderId);
            result.replace("count",list.size());
        } 
        return result;
    }
}
```

#### 4.司机开始接单和停止接单

司机在工作台页面上点击开始接单或者停止接单按钮之后，我们要清空司机的定位缓存、上线缓存，以及消息队列。为什么要做两次清空，因为有可能司机上线点击了开始接单，然后，手机退出了小程序，那么小程序这时候可能存放了很多定位缓存、上线缓存以及有可能存在的消息队列。

##### 1.开始接单和停止接单的业务逻辑

```java
@RestController
@RequestMapping("/driver")
@Tag(name = "DriverController", description = "司机模块Web接口")
public class DriverController {
    ……
        
    @PostMapping("/startWork")
    @Operation(summary = "开始接单")
    @SaCheckLogin
    public R startWork() {
        long driverId = StpUtil.getLoginIdAsLong();
        
        //删除司机定位缓存
        RemoveLocationCacheForm form_1 = new RemoveLocationCacheForm();
        form_1.setDriverId(driverId);
        locationService.removeLocationCache(form_1);

        //清空新订单消息列表
        ClearNewOrderQueueForm form_2 = new ClearNewOrderQueueForm();
        form_2.setUserId(driverId);
        newOrderMessageService.clearNewOrderQueue(form_2);

        return R.ok();
    }
    
    @PostMapping("/stopWork")
    @Operation(summary = "停止接单")
    @SaCheckLogin
    public R stopWork() {
        long driverId = StpUtil.getLoginIdAsLong();
        //删除司机定位缓存
        RemoveLocationCacheForm form_1 = new RemoveLocationCacheForm();
        form_1.setDriverId(driverId);
        locationService.removeLocationCache(form_1);

        //清空新订单消息列表
        ClearNewOrderQueueForm form_2 = new ClearNewOrderQueueForm();
        form_2.setUserId(driverId);
        newOrderMessageService.clearNewOrderQueue(form_2);

        return R.ok();
    }
}
```

##### 2.前端司机开始接单

我们声明`startWorkHandle()`函数，让司机开始接单。声明`stopWorkHandle()`函数，让司机停止接单。

```js
startWorkHandle: function() {
    ……
    uni.showModal({
        title: '提示消息',
        content: '你要开始接收代驾订单信息？',
        success: function(resp) {
            if (resp.confirm) {
                ……
                that.$refs.uToast.show({
                    title: '开始接单了',
                    type: 'success',
                    callback: function() {
                        ……
                        //初始化新订单和列表变量
                        that.newOrder = null;
                        that.newOrderList.length = 0;
                        that.executeOrder = {};
                        //创建接收新订单消息的定时器，每隔5秒钟接收一次新订单消息
                        if (that.reciveNewOrderTimer == null) {
                        	that.reciveNewOrderTimer = that.createTimer(that);
                        }
                    }
                });
            }
        }
    });
},

stopWorkHandle: function() {
    let that = this;
    uni.showModal({
        title: '提示消息',
        content: '你要停止接收代驾订单信息？',
        success: function(resp) {
            if (resp.confirm) {
                ……
                that.$refs.uToast.show({
                    title: '停止接单了',
                    type: 'default',
                    callback: function() {
                        ……
                        //初始化新订单和列表变量
                        that.newOrder = null;
                        that.newOrderList.length = 0;
                        that.executeOrder = {};
                        //销毁定时器
                        clearInterval(that.reciveNewOrderTimer);
                        that.reciveNewOrderTimer = null;
                        that.playFlag = false;
                    }
                });
            }
        }
    });
},
```

#### 5.Redis事务解决超售问题

##### 1.超售问题

超售就是卖出了超过预期数量的商品。比如说A商品库存是100个，但是秒杀的过程中，一共卖出去500个A商品。对于卖家来说，这就是超售。对于买家来说，这属于超买。

`案例：`

A顾客抢购商品，电商系统先去判断，商品有没有库存，假设现在某个商品库存为1，是可以抢购的，于是电商程序，开启事物，生成UPDATE语句，但是也许是线程被挂起的原因，或者网络延迟的原因，反正就是没有把UPDATE交给数据库执行。

这时候B顾客来抢购商品，电商系统也是先去判断有没有库存，因为A顾客抢购商品的SQL语句并没有执行，但是B顾客抢购商品执行的很顺利，电商系统开启了事务，然后生成库存减1的UPDATE语句，并且提交给数据库运行，事务提交之后，商品库存变成0。这时候A顾客的抢购商品的UPDATE语句，传递给数据库执行，于是数据库对库存又减1，这就形成了超售现象。原本库存只有1件商品，却卖出去两份订单。

##### 2.乐观锁机制解决问题

我们在数据表上面添加一个乐观锁字段，数据类型是整数的，用来记录数据更新的版本号，这个跟SVN机制很像。乐观锁是一种逻辑锁，他是通过版本号来判定有没有更新冲突出现。比如说，现在A商品的乐观锁版本号是0，现在有事务1来抢购商品了。事务1记录下版本号是0，等到执行修改库存的时候，就把乐观锁的版本号设置成1。但是事务1在执行的过程中，还没来得及执行UPDATE语句修改库存。这个时候事务2进来了，他执行的很快，直接把库存修改成99，然后把版本号变成了1。这时候，事务1开始执行UPDATE语句，但是发现乐观锁的版本号变成了1，这说明，肯定有人抢在事务1之前，更改了库存，所以事务1就不能更新，否则就会出现超售现象。

##### 3.Redis事务

Redis的事务就是个批处理机制，底层的原理和乐观锁相似，只不过把乐观锁拿到内存中实现。

比如说现在客户端A，在修改数据之前，会先要观察要修改的数据，相当于记下了数据的版本号。我们可以开启事务机制，所有的命令都不会立即发送给Redis，而是先缓存到客户端本地。等我们提交事务的时候，一次性，把这个命令发送给Redis。Redis分析版本号之后，发现没有问题，这时候就会执行这些批处理的命令，而且在执行过程中不允许打断，不会处理其他客户端的命令，这样就不会出现超售现象。

在客户端A在本地缓存命令的时候，这个时候客户端B修改了数据，版本号同时也更新了。这个时候客户端A提交事物，Redis发现客户端A的版本号，跟现有的数据版本号有逻辑冲突，所以就禁止执行客户端A的命令，这样事物就失败了。失败归失败，但是没有产生超售的现象。

##### 4.Redis的AOF模式

由于Redis使用内存缓存数据，如果Redis宕机，重启Redis之后，原本内存中缓存的数据就全都消失了。为了在宕机之后能有效恢复之前缓存的数据，我们可以开启Redis的持久化功能。

Redis有RDB和AOF两种持久化方式。RDB会根据指定的规则定时将内存中的数据保存到硬盘中，容易因为持久化不及时，导致恢复的时候丢失一部分缓存数据。AOF会将每次执行的命令及时保存到硬盘中，实时性更好，丢失的数据更少。

redis的config需要添加如下配置：

```bash
appendonly yes
appendfilename "appendonly.aof"
appendfsync always
```

##### 5.抢单逻辑（这部分比较重要）

redis事务代码如下：

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Resource
    private RedisTemplate redisTemplate;

    @Override
    @Transactional
    @LcnTransaction
    public String acceptNewOrder(long driverId, long orderId) {
        //Redis不存在抢单的新订单就代表抢单失败
        if (!redisTemplate.hasKey("order#" + orderId)) {
            return "抢单失败";
        }
        //执行Redis事务
        redisTemplate.execute(new SessionCallback() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                //获取新订单记录的Version
                operations.watch("order#" + orderId);
                //本地缓存Redis操作
                operations.multi();
                //把新订单缓存的Value设置成抢单司机的ID
                operations.opsForValue().set("order#" + orderId, driverId);
                //执行Redis事务，如果事务提交失败会自动抛出异常
                return operations.exec();
            }
        });
        //抢单成功之后，删除Redis中的新订单，避免让其他司机参与抢单
        redisTemplate.delete("order#" + orderId);
        //更新订单记录，添加上接单司机ID和接单时间
        HashMap param = new HashMap() {{
            put("driverId", driverId);
            put("orderId", orderId);
        }};
        int rows = orderDao.acceptNewOrder(param);
        if (rows != 1) {
            throw new HxdsException("接单失败，无法更新订单记录");
        }
        return "接单成功";

    }
}
```

### 四、自动抢单问题

自动抢单的原理非常简单，就是当语音播报订单结束之后，JS代码就立即发出抢单Ajax请求。如果是手动抢单，就是司机点击抢单按钮之后再发出Ajax请求。

在工作台页面的showNewOrder()函数中，我们补充上自动抢单和手动抢单的代码。

```js
showNewOrder: function(ref) {
    ……
    /*
     * 执行自动抢单
     */
    if (ref.settings.autoAccept) {
        ref.ajax(ref.url.acceptNewOrder, 'POST', { orderId: orderId },function(resp) {
                let result = resp.data.result;
                //自动抢单成功
                if (result == '接单成功') {
                    uni.showToast({
                        title: '接单成功'
                    });
                    audio = uni.createInnerAudioContext();
                    ref.audio = audio;
                    audio.src = '/static/voice/voice_3.mp3';
                    audio.play();
                    audio.onEnded(function() {
                        //停止接单
                        ref.audio = null;
                        ref.ajax(ref.url.stopWork, 'POST', null, function(resp) {});
                        //初始化新订单和列表变量
                        ref.newOrder = null;
                        ref.newOrderList.length = 0;
                        ref.executeOrder.id = orderId;
                        clearInterval(ref.reciveNewOrderTimer);
                        ref.reciveNewOrderTimer = null;
                        ref.playFlag = false;
                        //隐藏了工作台页面底部操作条之后，需要重新计算订单执行View的高度
                        ref.contentStyle = `width: 750rpx;height:${ref.windowHeight - 200 - 0}px;`;
                        //TODO 加载订单执行数据
                    });
                } else {
                    //自动抢单失败
                    audio = uni.createInnerAudioContext();
                    ref.audio = audio;
                    audio.src = '/static/voice/voice_4.mp3';
                    audio.play();
                    audio.onEnded(function() {
                        //抢单失败就切换下一个订单
                        ref.playFlag = false;
                        if (ref.newOrderList.length > 0) {
                            ref.showNewOrder(ref); //递归调用
                        } else {
                            ref.newOrder = null;
                        }
                    });
                }
            },
            false
        );
    } else {
        /**
         * 每个订单在页面上停留3秒钟，等待司机手动抢单
         */
        ref.playFlag = false;
        setTimeout(function() {
            //如果用户不是正在手动抢单中，就播放下一个新订单
            if (!ref.accepting) {
                ref.canAcceptOrder = false;
                if (ref.newOrderList.length > 0) {
                    ref.showNewOrder(ref); //递归调用
                } else {
                    ref.newOrder = null;
                }
            }
        }, 3000);
    }
},
```
#### 1.手动抢单

手动抢单代码如下：

```js
acceptHandle: function() {
    let that = this;
    if (!that.canAcceptOrder || that.accepting) {
        return;
    }

    that.accepting = true; //正在抢单中，该变量可以避免多次重复抢单
    uni.vibrateShort({});
    that.ajax(that.url.acceptNewOrder, 'POST', { orderId: that.newOrder.orderId }, function(resp) {
        let audio = uni.createInnerAudioContext();
        let result = resp.data.result;
        //手动抢单成功
        if (result == '接单成功') {
            uni.showToast({
                title: '接单成功'
            });
            that.audio = audio;
            audio.src = '/static/voice/voice_3.mp3';
            audio.play();
            audio.onEnded(function() {
                //停止接单
                that.audio = null;
                that.ajax(that.url.stopWork, 'POST', null, function(resp) {});

                //初始化新订单和列表变量
                that.executeOrder.id = that.newOrder.orderId;
                that.newOrder = null;
                that.newOrderList.length = 0;
                clearInterval(that.reciveNewOrderTimer);
                that.reciveNewOrderTimer = null;
                that.playFlag = false;
                that.accepting = false;
                that.canAcceptOrder = false;
                //隐藏了工作台页面底部操作条之后，需要重新计算订单执行View的高度
                that.contentStyle = `width: 750rpx;height:${that.windowHeight - 200 - 0}px;`;
                //TODO 加载订单执行数据
            });
        } else {
            that.audio = audio;
            audio.src = '/static/voice/voice_4.mp3';
            audio.play();
            that.playFlag = false;
            setTimeout(function() {
                that.accepting = false;
                that.canAcceptOrder = false;
                if (that.newOrderList.length > 0) {
                    that.showNewOrder(that); //递归调用
                } else {
                    that.newOrder = null;
                }
            }, 3000);
        }
    });
},
```
#### 2.前端显示执行的订单

查询司机正在执行的订单业务逻辑：

```xml
<select id="searchDriverExecuteOrder" parameterType="Map" resultType="HashMap">
    SELECT CAST(id AS CHAR)                              AS id,
           customer_id                                   AS customerId,
           start_place                                   AS startPlace,
           start_place_location                          AS startPlaceLocation,
           end_place                                     AS endPlace,
           end_place_location                            AS endPlaceLocation,
           CAST(favour_fee AS CHAR)                      AS favourFee,
           car_plate                                     AS carPlate,
           car_type                                      AS carType,
           DATE_FORMAT(create_time, '%Y-%m-%d %H:%i:%s') AS createTime
    FROM tb_order
    WHERE id = #{orderId}
      AND driver_id = #{driverId}
</select>	
```

```java
@Service
public class OrderServiceImpl implements OrderService {   
    @Override
    public HashMap searchDriverExecuteOrder(SearchDriverExecuteOrderForm form) {
        //查询订单信息
        R r = odrServiceApi.searchDriverExecuteOrder(form);
        HashMap orderMap = (HashMap) r.get("result");

        //查询代驾客户信息
        long customerId = MapUtil.getLong(orderMap, "customerId");
        SearchCustomerInfoInOrderForm infoInOrderForm = new SearchCustomerInfoInOrderForm();
        infoInOrderForm.setCustomerId(customerId);
        r = cstServiceApi.searchCustomerInfoInOrder(infoInOrderForm);
        HashMap cstMap = (HashMap) r.get("result");

        HashMap map = new HashMap();
        map.putAll(orderMap);
        map.putAll(cstMap);
        return map;
    }
}
```

前端页面加载数据：

```js
loadExecuteOrder: function(ref) {
    ref.ajax(ref.url.searchDriverExecuteOrder, 'POST', { orderId: ref.executeOrder.id }, function(resp) {
        let result = resp.data.result;
        // console.log(result);
        ref.executeOrder = {
            id: ref.executeOrder.id,
            photo: result.photo,
            title: result.title,
            tel: result.tel,
            customerId: result.customerId,
            startPlace: result.startPlace,
            startPlaceLocation: JSON.parse(result.startPlaceLocation),
            endPlace: result.endPlace,
            endPlaceLocation: JSON.parse(result.endPlaceLocation),
            favourFee: result.favourFee,
            carPlate: result.carPlate,
            carType: result.carType,
            createTime: result.createTime
        };

        ref.workStatus = '接客户';
        uni.setStorageSync('workStatus', '接客户');
        uni.setStorageSync('executeOrder', ref.executeOrder);
    });
},
```

电话拨号

```js
callServiceHandle: function() {
    uni.makePhoneCall({
        phoneNumber: '10086'
    });
},
```
### 五、乘客端取消订单

因为乘客端的等待司机接单的倒计时为15分钟，现在后端Redis里面的抢单缓存也是15分钟。如果移动端倒计时到15分钟恰好结束，这时候发起Ajax请求取消订单，那么就会给后端程序带来困难，因为在Redis里面已经不存在抢单缓存了，后端程序无法判断到底是司机抢单成功后删除了抢单缓存，还是因为没有司机抢单，缓存过期自动删除了。

有的同学觉得我们查询数据库的订单状态不就知道有没有人接单了么？这是不行的。因为恰好司机抢单成功之后，修改了订单的状态为2，但是事务还没来的及提交，这时候关闭订单的Ajax请求发过来了。Java程序通过查询数据库，发现订单的状态是1，以为没有司机接单，实际上已经有司机接单了。

为了避免以上的情况，我们把Redis抢单缓存延长到16分钟。如果关闭订单的Ajax请求发送给后端，这时候抢单缓存还存在，说明没有司机抢单，那么我们就删除抢单缓存和订单记录就可以了。如果抢单缓存不存在，说明已经有司机抢单成功了，这时候我们返回给乘客端关闭订单失败，已有司机抢单成功即可。

#### 1.后端业务逻辑

```java
@Data
@Schema(description = "查询订单状态的表单")
public class SearchOrderStatusForm {
    @NotNull(message = "orderId不能为空")
    @Schema(description = "订单ID")
    private Long orderId;

    @Min(value = 0, message = "driverId不能小于0")
    @Schema(description = "司机ID")
    private Long driverId;

    @Min(value = 0, message = "customerId不能小于0")
    @Schema(description = "乘客ID")
    private Long customerId;
}

@Data
@Schema(description = "更新订单状态的表单")
public class DeleteUnAcceptOrderForm {
    @NotNull(message = "orderId不能为空")
    @Schema(description = "订单ID")
    private Long orderId;

    @Min(value = 0, message = "driverId不能小于0")
    @Schema(description = "司机ID")
    private Long driverId;

    @Min(value = 0, message = "customerId不能小于0")
    @Schema(description = "乘客ID")
    private Long customerId;

}

@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    ……
        
    @PostMapping("/searchOrderStatus")
    @Operation(summary = "查询订单状态")
    public R searchOrderStatus(@RequestBody @Valid SearchOrderStatusForm form) {
        Map param = BeanUtil.beanToMap(form);
        Integer status = orderService.searchOrderStatus(param);
        return R.ok().put("result", status);
    }

    @PostMapping("/deleteUnAcceptOrder")
    @Operation(summary = "删除没有司机接单的订单")
    public R deleteUnAcceptOrder(@RequestBody @Valid DeleteUnAcceptOrderForm form) {
        Map param = BeanUtil.beanToMap(form);
        String result = orderService.deleteUnAcceptOrder(param);
        return R.ok().put("result", result);
    }
}

@Service
public class OrderServiceImpl implements OrderService {    
    @Override
    public Integer searchOrderStatus(Map param) {
        Integer status = orderDao.searchOrderStatus(param);
        if (status == null) {
            throw new HxdsException("没有查询到数据，请核对查询条件");
        }
        return status;
    }

    @Override
    @Transactional
    @LcnTransaction
    public String deleteUnAcceptOrder(Map param) {
        long orderId = MapUtil.getLong(param, "orderId");
        if (!redisTemplate.hasKey("order#" + orderId)) {
            return "订单取消失败";
        }
        redisTemplate.execute(new SessionCallback() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                operations.watch("order#" + orderId);
                operations.multi();
                operations.opsForValue().set("order#" + orderId, "none");
                return operations.exec();
            }
        });

        redisTemplate.delete("order#" + orderId);
        int rows = orderDao.deleteUnAcceptOrder(param);
        if (rows != 1) {
            return "订单取消失败";
        }
        return "订单取消成功";
    }
}
```

#### 2.前端逻辑

我们把查询订单状态的代码封装起来，这样每隔5秒钟轮询的时候，以及倒计时结束取消订单失败后，我们再次查询订单状态时候，都要调用这个封装函数。

```java
searchOrderStatus: function(ref) {
    let data = {
        orderId: ref.orderId
    };
    ref.ajax(
        ref.url.searchOrderStatus,
        'POST',
        data,
        function(resp) {
            // console.log(resp.data.result)
            if (resp.data.result == 2) {
                ref.showPopup = false;
                ref.timestamp = null;
                uni.showToast({
                    icon: 'success',
                    title: '司机已接单'
                });
                uni.vibrateShort({});
                setTimeout(function() {
                    uni.redirectTo({
                        url: '../move/move?orderId=' + ref.orderId
                    });
                }, 3000);
            }
        },
        false
    );
},
```

然后关闭没有司机接单的订单，这部分代码我们也给封装起来。

```java
deleteUnAcceptOrder: function(ref) {
    ref.showPopup = false;
    ref.timestamp = null;
    let data = {
        orderId: ref.orderId
    };
    ref.ajax(ref.url.deleteUnAcceptOrder, 'POST', data, function(resp) {
        let result = resp.data.result;
        if (result == '订单取消成功') {
            uni.showToast({
                icon: 'success',
                title: '订单取消成功'
            });
            setTimeout(function() {
                uni.redirectTo({
                    url: '../workbench/workbench'
                });
            }, 3000);
        } else {
            ref.searchOrderStatus(ref);
        }
    });
},
```

轮询是否接单和关闭订单。倒计时组件有两个属性：`@change`和`@end`。倒计时数字每次变化的时候触发的是`@change`，倒计时结束后触发的是`@end`，这两个事件的回调函数我们都要写一下。

```js
<u-count-down
    ref="uCountDown"
    :timestamp="timestamp"
    :autoplay="false"
    separator="zh"
    :show-hours="false"
    :show-border="true"
    bg-color="#DDF0FF"
    separator-color="#0083F3"
    border-color="#0D90FF"
    color="#0D90FF"
    font-size="32"
    @end="countEndHandle"
    @change="countChangeHandle"
></u-count-down>


countChangeHandle: function(s) {
    let that = this;
    if (s != 0 && s % 5 == 0) {
        that.searchOrderStatus(that);
    }
},


countEndHandle: function() {
    //显示没有人接单，关闭订单
    let that = this;
    that.deleteUnAcceptOrder(that);
},

//在倒计时弹窗中，有手动关闭订单的按钮。
<button class="btn" @tap="cancelHandle">取消订单</button>
//该按钮的点击事件回调函数我们给实现一下，其实就是发出关闭订单的请求。
cancelHandle: function() {
    let that = this;
    that.deleteUnAcceptOrder(that);
}
```

### 六、本章总结

#### 1.开通腾讯位置服务，封装地图服务接口

在这一章，我们首先开通了腾讯位置服务。利用腾讯位置服务，我们能计算最佳线路、里程和时间。腾讯位置服务给我们提供了Java语言的API，还有JS的API，以及小程序的地图选点插件。很多人把腾讯位置服务跟小程序页面上的地图组件给弄混了。其实它们完全是两种东西，腾讯位置服务侧重于提供服务，我们拿到最佳线路的导航数据之后，必须解压处理，才能传递给小程序的地图组件显示出来，我们可以给地图组件设定线路的颜色，起点和终点的图标等等。所以地图组件不提供任何复杂的地图运算服务，它只是负责显示，真正负责运算的是腾讯位置服务。

#### 2.实现小程序地图选点功能，选择起点和终点

乘客端创建订单的时候，我们用上了腾旭位置服务提供的地图选点插件，在页面上可以很容易的设定好代驾订单的起点和终点。

#### 3.利用腾讯位置服务规划最佳线路

然后接下来该调用腾讯位置服务，帮我们规划出最佳的线路，并且在页面上显示出来。注意，这里我们是用腾讯位置服务里面JavaScript的API来做的，这也是为了减轻后端项目的压力。

#### 4.实现乘客端的车辆管理

乘客创建代驾订单必须要选择代驾车辆，在小程序页面上可以添加、删除和选中车辆，这部分功能比较简单，就是普通的CRUD操作。

#### 5.创建代驾订单和抢单缓存

乘客端提交Ajax请求创建代驾订单，后端Java程序要做的事情非常多，比如说调用腾讯位置服务重新估算订单的距离，还要查询附近有没有适合接单的司机。如果有符合接单的司机，才会创建代驾订单保存到MySQL里面，并且往Redis添加抢单缓存。

#### 6.实时缓存司机定位和线上缓存

关于司机定位缓存这块，我们在司机端开启了实时定位功能，GPS定位信息上传到后端会自动缓存起来，并且创建司机的上线缓存。

#### 7.利用GEO运算，查找适合接单的司机

查找周围适合接单司机，我们先是用上了Redis的GEO运算，计算方圆5公里以内的司机。然后再看看订单能不能跟司机的接单设置匹配上。

#### 8.利用RabbitMQ实现消息收发

如果能匹配上，系统就创建订单，并且把消息发送给相关司机。消息收发这个功能，我们是用的RabbitMQ来实现的。特别是RabbitMQ的阻塞和非阻塞收发消息模式，以及Java程序同步执行和异步执行方式，大家一定要区分清楚了。

#### 9.司机端小程序轮询新订单，自动/手动抢单

司机端小程序为了能即使获取到新订单，我们采用了轮询的方式，从RabbitMQ中接收消息。收到新订单之后，司机端会语音播报这个新订单的内容。等到语音播报结束，司机可以手动抢单，也可以由系统自动抢单。如果在手动抢单模式下，司机超过3秒钟没有抢单，就自动切换到下一个订单。

#### 10.利用Redis事务解决抢单超售

如果同时有多个司机抢同一个订单，就会出现超售的现象。一个订单被多个司机抢到，这种结果我们肯定不能接受。为了避免超售，我们要用上Redis事务机制。

#### 11.开启Redis的AOF模式，减少缓存数据丢失

除此之外，我们还要把Redis设置成AOF模式。Redis里面缓存的数据，会频繁的同步到硬盘上。如果Redis出现宕机，我们重新启动Redis，AOF文件的内容会自动加载到内存中，缓存就会恢复了。虽然不能100%恢复宕机前的缓存，但是这已经能找回绝大多数缓存信息了。

#### 12.乘客端轮询是否有司机接单

乘客端下单之后，要采用轮询的方式查看是否有司机接单。我们设定的是每隔5秒钟就轮询一次，其实就是看订单记录的状态变成2，就说明有司机接单了。

#### 13.乘客端自动或者手动关闭订单

如果倒计时结束也没有司机接单，那么就应该关闭订单了。就是删除订单和抢单缓存，然后乘客端从create_order页面跳回到工作台页面。当然了，不用等到倒计时结束，乘客也能手动关闭订单的。

## 第五章 订单执行和监控

无论司机端还是乘客端的小程序，如果遇到微信闪退，重新登录小程序之后，必须加载当前的订单。司机端和乘客端的小程序都有司乘同显页面，这也是我们本章要完成的工作。另外，订单在执行的过程中，小程序要实时采集录音，把录音和对话文本上传到后端系统。还有就是乘客和司机如果配合刷单，骗取平台补贴，这个事情我们要用程序做个判断，把骗取补贴这个漏洞给堵上。

### 一、司机端加载执行的订单

司机加载订单时，需要根据司机编号从订单表中查询出正在执行的订单，并根据订单表中的用户id，查询出用户信息。这个信息主要是乘客用户的名字、电话。

#### 1.Dao层xml文件

status状态有： 1等待接单，**2已接单，3司机已到达，4开始代驾**，5结束代驾，6未付款，7已付款，8订单已结束，9顾客撤单，10司机撤单，11事故关闭，12其他

```xml
<select id="searchDriverCurrentOrder" parameterType="long" resultType="HashMap">
    SELECT CAST(id AS CHAR)                              AS id,
           customer_id                                   AS customerId,
           start_place                                   AS startPlace,
           start_place_location                          AS startPlaceLocation,
           end_place                                     AS endPlace,
           end_place_location                            AS endPlaceLocation,
           CAST(favour_fee AS CHAR)                      AS favourFee,
           car_plate                                     AS carPlate,
           car_type                                      AS carType,
           DATE_FORMAT(create_time, '%Y-%m-%d %H:%i:%s') AS createTime,
           `status`
    FROM tb_order
    WHERE driver_id = #{driverId}
      AND `status` IN (2, 3, 4) LIMIT 1
</select>
```

#### 2.Web层代码

**form表单：**

```java
@Data
@Schema(description = "查询司机当前订单的表单")
public class SearchDriverCurrentOrderForm {
    @NotNull(message = "driverId不能为空")
    @Min(value = 1, message = "driverId不能小于1")
    @Schema(description = "司机ID")
    private Long driverId;
}
```

```java
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    ……
        
    @PostMapping("/searchDriverCurrentOrder")
    @Operation(summary = "查询司机当前订单")
    public R searchDriverCurrentOrder(@RequestBody @Valid SearchDriverCurrentOrderForm form) {
        HashMap map = orderService.searchDriverCurrentOrder(form.getDriverId());
        return R.ok().put("result", map);
    }
}
```

#### 3.Feign层代码

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Override
    public HashMap searchDriverCurrentOrder(SearchDriverCurrentOrderForm form) {
       	// 查询出当前代驾的订单
        R r = odrServiceApi.searchDriverCurrentOrder(form);
        HashMap orderMap = (HashMap) r.get("result");

        if (MapUtil.isNotEmpty(orderMap)) {
            HashMap map = new HashMap();
            //查询代驾客户信息
            long customerId = MapUtil.getLong(orderMap, "customerId");
            SearchCustomerInfoInOrderForm infoInOrderForm = new SearchCustomerInfoInOrderForm();
            infoInOrderForm.setCustomerId(customerId);
            r = cstServiceApi.searchCustomerInfoInOrder(infoInOrderForm);
            HashMap cstMap = (HashMap) r.get("result");
            // 当前执行订单的信息
            map.putAll(orderMap);
            // 当前执行订单的客户信息
            map.putAll(cstMap);
            return map;
        } else {
            return null;
        }
    }
}
```

#### 4.前端程序

```js
onLoad:function(){
    ……
    //查询当前执行的订单
    that.ajax(
        that.url.searchDriverCurrentOrder,
        'POST',
        null,
        function(resp) {
            if (resp.data.hasOwnProperty('result')) {
                let result = resp.data.result;
                // console.log(resp.data);
                that.executeOrder = {
                    id: result.id,
                    photo: result.photo,
                    title: result.title,
                    tel: result.tel,
                    customerId: result.customerId,
                    startPlace: result.startPlace,
                    startPlaceLocation: JSON.parse(result.startPlaceLocation),
                    endPlace: result.endPlace,
                    endPlaceLocation: JSON.parse(result.endPlaceLocation),
                    favourFee: result.favourFee,
                    carPlate: result.carPlate,
                    carType: result.carType,
                    createTime: result.createTime
                };
                let map = {
                    '2': '接客户',
                    '3': '到达代驾点',
                    '4': '开始代驾'
                };
                that.contentStyle = `width: 750rpx;height:${that.windowHeight - 200 - 0}px;`;
                that.workStatus = map[result.status + ''];
                uni.setStorageSync('workStatus', that.workStatus);
                uni.setStorageSync('executeOrder', that.executeOrder);
            }
        },
        false
    );

}
```

### 二、乘客端加载执行的订单

如果当前有正在执行的订单，就直接加载该订单的信息。这个功能应该在乘客端小程序上面也要实现一下。乘客端加载订单可比司机端复杂多了。如果是有未接单的订单，页面要跳转到create_order.vue页面，然后重新倒计时，但是由于可能会出现抢单缓存超时被销毁了，但是代驾系统以为有司机抢单了，实际上根本没有司机抢单。

如果乘客下单之后，微信出现了闪退，然后过了半个小时，乘客重新登陆小程序，因为数据库中存在没有接单的订单，小程序会跳转到create_order.vue页面，继续开始倒计时等待司机接单。由于抢单缓存早就销毁了，即便倒计时结束，发起Ajax请求关闭订单。但是业务层发现没有抢单缓存，那么可能就有司机接单了（实际上根本没有司机接单），又关闭不了订单，于是就僵持在这里了。为了避免上面情况的发生，我们要用Java程序监听抢单缓存的销毁事件，赶紧把关联的订单和账单记录给删除掉。即便像上面那样，乘客半小时后登陆小程序，因为没有订单了，所以乘客可以重新下单。

#### 1.修改RedisConfiguration类

在该类中，添加RedisMessageListenerContainer监听类类，同时设置缓存数据销毁通知队列，如果有缓存销毁，就自动往这个队列中发消息。

```java
@Configuration
public class RedisConfiguration {

    @Resource
    private RedisConnectionFactory redisConnectionFactory;

    @Bean
    public ChannelTopic expiredTopic() {
        /*
         * 自定义Redis队列的名字，如果有缓存销毁，就自动往这个队列中发消息
         * 每个子系统有各自的Redis逻辑库，订单子系统不会监听到其他子系统缓存数据销毁
         */
        return new ChannelTopic("__keyevent@5__:expired");  
    }

    // 实现Redis消息监听容器
    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer() {
        RedisMessageListenerContainer redisMessageListenerContainer = new RedisMessageListenerContainer();
        redisMessageListenerContainer.setConnectionFactory(redisConnectionFactory);
        return redisMessageListenerContainer;
    }
}

```

同时定义一个key过期时间消息监听器

```java
@Slf4j
@Component
public class KeyExpiredListener extends KeyExpirationEventMessageListener {
    @Resource
    private OrderDao orderDao;

    @Resource
    private OrderBillDao orderBillDao;

    public KeyExpiredListener(RedisMessageListenerContainer listenerContainer) {
        super(listenerContainer);
    }

    @Override
    @Transactional
    public void onMessage(Message message, byte[] pattern) {
        //从消息队列中接收消息
        if (new String(message.getChannel()).equals("__keyevent@5__:expired")) {
            
            //反序列化Key，否则出现乱码
            JdkSerializationRedisSerializer serializer = new JdkSerializationRedisSerializer();
            String key = serializer.deserialize(message.getBody()).toString();
            
            if (key.contains("order#")) {
                long orderId = Long.parseLong(key.split("#")[1]);
                HashMap param = new HashMap() {{
                    put("orderId", orderId);
                }};
                int rows = orderDao.deleteUnAcceptOrder(param);
                if (rows == 1) {
                    log.info("删除了无人接单的订单：" + orderId);
                }
                rows = orderBillDao.deleteUnAcceptOrderBill(orderId);
                if (rows == 1) {
                    log.info("删除了无人接单的账单：" + orderId);
                }
            }
        }
        super.onMessage(message, pattern);
    }
}

```

现在还有一种情况需要我们动脑子认真想想，比如说乘客下单成功之后，等待了5分钟，微信就闪退了。过了5分钟之后，他重新登录小程序。因为抢单缓存还没有被销毁，而且订单和账单记录也都在，小程序跳转到create_order.vue页面，重新从15分钟开始倒计时，但是倒计时过程中，抢单缓存会超时被销毁，同时订单和账单记录也都删除了。这时候乘客端小程序发来轮询请求，业务层发现倒计时还没结束，但是抢单缓存就没有了，说明有司机抢单了，于是就跳转到司乘同显页面，这明显是不对的。

于是我们要改造OrderServiceImpl类中的代码，把抛出异常改成返回状态码为0，在移动端轮询的时候如果发现状态码是0，说明订单已经被关闭了。所以就弹出提示消息即可。

```java
@Service
public class OrderServiceImpl implements OrderService {
    ……
        
    @Override
    public Integer searchOrderStatus(Map param) {
        Integer status = orderDao.searchOrderStatus(param);
        if (status == null) {
            //throw new HxdsException("没有查询到数据，请核对查询条件");
            status=0;
        }
        return status;
    }
}
```

### 三、司机乘客同显





