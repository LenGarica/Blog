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

现在还有一种情况需要我们动脑子认真想想，比如说乘客下单成功之后，等待了5分钟，微信就闪退了。过了5分钟之后，他重新登录小程序。因为抢单缓存还没有被销毁，而且订单和账单记录也都在，小程序跳转到create_order.vue页面，重新从15分钟开始倒计时，但是倒计时过程中，抢单缓存会超时被销毁，**同时订单和账单记录也都删除了**。这时候乘客端小程序发来轮询请求，业务层发现倒计时还没结束，但是抢单缓存就没有了，说明有司机抢单了，于是就跳转到司乘同显页面，这明显是不对的。

于是我们要改造OrderServiceImpl类中的代码，把抛出异常改成返回状态码为0，在移动端轮询的时候如果发现状态码是0，说明订单已经被关闭了，关闭倒计时，弹出提示消息即可。

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

根据orderId查询出属于某个司机或者乘客同显有关的订单信息。

```sql
<select id="searchOrderForMoveById" parameterType="Map" resultType="HashMap">
    SELECT start_place AS startPlace,
           start_place_location AS startPlaceLocation,
           end_place AS endPlace,
           end_place_location AS endPlaceLocation,
           `status`
    FROM tb_order
    WHERE id = #{orderId}
    <if test="customerId!=null">
        AND customer_id = #{customerId}
    </if>
    <if test="driverId!=null">
        AND driver_id = #{driverId}
    </if>
    LIMIT 1;
</select>
```

提供的接口：

```java
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    @PostMapping("/searchOrderForMoveById")
    @Operation(summary = "查询订单信息用于司乘同显功能")
    public R searchOrderForMoveById(@RequestBody @Valid SearchOrderForMoveByIdForm form) {
        Map param = BeanUtil.beanToMap(form);
        HashMap map = orderService.searchOrderForMoveById(param);
        return R.ok().put("result", map);
    }
}
```

#### 1.司机端最佳线路的显示

因为司乘同显页面上要显示最佳线路，所以我们要对从腾讯位置服务中查询到的导航坐标解压缩，我们必须在页面中声明解压函数。腾讯位置服务返回的数据是一个坐标数组，要将坐标数组拿出来进行解压。就是以前我们写乘客端小程序`create_order.vue`页面的时候，用过这个函数，这里可以直接复制过来。

解压算法：分两步

- 将腾讯返回的polyline数组，从第三个数开始，f(n) = [f(n - 2) + f(n)] / 1 千万 进行处理
- 然后再次遍历数组，从第一个数开始，每两个为一组放入新数组中，这样数组中每一组数据，就是经度和纬度。

```js
formatPolyline(polyline) {
    let coors = polyline;
    let pl = [];
    //坐标解压（返回的点串坐标，通过前向差分进行压缩）
    const kr = 1000000;
    for (let i = 2; i < coors.length; i++) {
        coors[i] = Number(coors[i - 2]) + Number(coors[i]) / kr;
    }
    //将解压后的坐标放入点串数组pl中
    for (let i = 0; i < coors.length; i += 2) {
        pl.push({
            longitude: coors[i + 1],
            latitude: coors[i]
        });
    }
    return pl;
},
```

#### 2.司机端地图终点的变化

在`move.vue`页面中，声明计算最佳线路的函数。因为司机接单之后要赶往上车点，所以最佳线路的终点是上车点。如果开始代驾了，那么最佳线路的终点应该是订单的终点。所以在`calculateLine()`函数中，最佳线路的终点是随着订单状态改变，而发生变化的。

```js
calculateLine: function(ref) {
    //如果还没有获得最新的定位，就不计算最佳线路
    if (ref.latitude == 0 || ref.longitude == 0) {
        return;
    }
    qqmapsdk.direction({
        mode: ref.mode, //可选值：'driving'（驾车）、'walking'（步行）、'bicycling'（骑行），不填默认：'driving',可不填
        //from参数不填默认当前地址
        from: {
            latitude: ref.latitude,
            longitude: ref.longitude
        },
        to: {
            latitude: ref.targetLatitude,
            longitude: ref.targetLongitude
        },
        success: function(resp) {
            let route = resp.result.routes[0];
            let distance = route.distance;
            let duration = route.duration;
            let polyline = route.polyline;
            ref.distance = Math.ceil((distance / 1000) * 10) / 10;
            ref.duration = duration;

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
                    latitude: ref.latitude,
                    longitude: ref.longitude,
                    width: 35,
                    height: 35,
                    anchor: {
                        x: 0.5,
                        y: 0.5
                    },
                    iconPath: '../static/move/driver-icon.png'
                }
            ];
        },
        fail: function(error) {
            console.log(error);
        }
    });
}
```

#### 3.乘客端司机位置显示

在司机赶往上车点的途中，这时候乘客的定位没什么用处，乘客端的司乘同显页面要根据司机的定位规划最佳线路。如果现在开始代驾了，这时候乘客的定位才会被用上，司乘同显根据乘客定位规划最佳线路。乘客端想要拿到司机的定位信息，司机端必须上传定位才可以。

- 司机上传定位

```java
@Service
public class DriverLocationServiceImpl implements DriverLocationService {
    // 司机上传自己的定位，同时将位置保存在redis中。这里使用的就是普通的redis缓存，没有使用GEO
    @Override
    public void updateOrderLocationCache(Map param) {
        long orderId = MapUtil.getLong(param, "orderId");
        String latitude = MapUtil.getStr(param, "latitude");
        String longitude = MapUtil.getStr(param, "longitude");
        String location = latitude + "#" + longitude;
        // 缓存 key : 订单编号， value ： 位置信息latitude + "#" + longitude， 过期时间： 10分钟
        redisTemplate.opsForValue().set("order_location#" + orderId, location, 10, TimeUnit.MINUTES);
    }

    // 查询订单缓存
    @Override
    public HashMap searchOrderLocationCache(long orderId) {
        // 根据key查询定位
        Object obj = redisTemplate.opsForValue().get("order_location#" + orderId);
        if (obj != null) {
            String[] temp = obj.toString().split("#");
            String latitude = temp[0];
            String longitude = temp[1];
            HashMap map = new HashMap() {{
                put("latitude", latitude);
                put("longitude", longitude);
            }};
            return map;
        }
        return null;
    }
}

// 接口代码
@RestController
@RequestMapping("/driver/location")
@Tag(name = "DriverLocationController", description = "司机定位服务Web接口")
public class DriverLocationController {
    @PostMapping("/updateOrderLocationCache")
    @Operation(summary = "更新订单定位缓存")
    @SaCheckLogin
    public R updateOrderLocationCache(@RequestBody @Valid UpdateOrderLocationCacheForm form) {
        driverLocationService.updateOrderLocationCache(form);
        return R.ok();
    }
}

// 前端代码
wx.onLocationChange(function(resp) {
    ……
    if (workStatus == '开始接单') {
        ……
    }
    else if (workStatus == '接客户') {
        let executeOrder = uni.getStorageSync('executeOrder');
        let orderId = executeOrder.id;
        let data = {
            orderId: orderId,
            latitude: latitude,
            longitude: longitude
        };
        uni.request({
            url: `${baseUrl}/driver/location/updateOrderLocationCache`,
            method: 'POST',
            header: {
                token: uni.getStorageSync('token')
            },
            data: data,
            success: function(resp) {
                if (resp.statusCode == 401) {
                    uni.redirectTo({
                        url: 'pages/login/login'
                    });
                } else if (resp.statusCode == 200 && resp.data.code == 200) {
                    let data = resp.data;
                    if (data.hasOwnProperty('token')) {
                        let token = data.token;
                        uni.setStorageSync('token', token);
                    }
                    console.log('订单定位更新成功');
                } else {
                    console.error('订单定位更新失败', resp.data);
                }
            },
            fail: function(error) {
                console.error('订单定位更新失败', error);
            }
        });
    } else if (workStatus == '开始代驾') {
        //TODO 每凑够20个定位就上传一次，减少服务器的压力
    }
    uni.$emit('updateLocation', location);
});
```

- 乘客端获取司机位置

```java
@RestController
@RequestMapping("/order/location")
@Tag(name = "OrderLocationController", description = "订单定位服务Web接口")
public class OrderLocationController {

    @Resource
    private OrderLocationService orderLocationService;

    @PostMapping("/searchOrderLocationCache")
    @Operation(summary = "查询订单定位缓存")
    @SaCheckLogin
    public R searchOrderLocationCache(@RequestBody @Valid SearchOrderLocationCacheForm form) {
        HashMap map = orderLocationService.searchOrderLocationCache(form);
        return R.ok().put("result", map);
    }
}
```

#### 4.到达代驾点确认

司机赶到上车点之后，点击“到达上车点”按钮，乘客点击“司机到达”按钮，然后司乘同显的路线，就变成了乘客当前定位到代驾终点的最佳线路。

```java
@Data
@Schema(description = "更新订单状态的表单")
public class ArriveStartPlaceForm {
    @NotNull(message = "orderId不能为空")
    @Min(value = 1, message = "orderId不能小于1")
    @Schema(description = "订单ID")
    private Long orderId;

    @NotNull(message = "driverId不能为空")
    @Min(value = 1, message = "driverId不能小于1")
    @Schema(description = "司机ID")
    private Long driverId;

}

// 业务接口
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    @PostMapping("/arriveStartPlace")
    @Operation(summary = "司机到达上车点")
    public R arriveStartPlace(@RequestBody @Valid ArriveStartPlaceForm form) {
        Map param = BeanUtil.beanToMap(form);
        param.put("status", 3);
        int rows = orderService.arriveStartPlace(param);
        return R.ok().put("rows", rows);
    }
}

// feign接口
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    @PostMapping("/arriveStartPlace")
    @Operation(summary = "司机到达上车点")
    @SaCheckLogin
    public R arriveStartPlace(@RequestBody @Valid ArriveStartPlaceForm form) {
        long driverId = StpUtil.getLoginIdAsLong();
        form.setDriverId(driverId);
        int rows = orderService.arriveStartPlace(form);
        return R.ok().put("rows", rows);
    }
}
```

有一点需要必须清楚，乘客端点击了司机已到达，并不更改订单状态，因为上节课司机已到达的时候就已经把订单改成了3状态，为什么不是乘客确认司机已到达之后，再把订单改成3状态呢？这是因为司机到达上车点之后，有10分钟的免费等时。10分钟之后，乘客还没有到达上车点，代驾账单中就会出现等时费（1分钟1元钱）。有时候明明司机已经到达了上车点，但是乘客拖拖拉拉半个小时才到上车点，然后才点击司机已到达，于是等待的半个小时就成了免费的，因为没有确认司机已到达上车点，那就不算等时。

乘客端点击确认司机已到达之后，并不会修改订单状态，修改Redis里面的标志位缓存。等到司机端点击开始代驾的时候，要确定Redis里面标志位的值为2，然后才能把订单更新成4状态

status  ：  1等待接单，2已接单，3司机已到达，4开始代驾，5结束代驾，6未付款，7已付款，8订单已结束，9顾客撤单，10司机撤单，11事故关闭，12其他

#### 5.本地app实现导航

司机点击了开始代驾后，由于小程序总体积不能超过32MB，因此需要调用本地地图App。

```js
// 点击开始代驾按钮，调起app
<view class="item" @tap="showNavigationHandle">
    <image src="../../static/workbench/other-icon-1.png" 
           mode="widthFix" 
           class="location-icon"/>
    <text class="location-text">定位导航</text>
</view>

showNavigationHandle: function() {
    let that = this;
    let latitude = null;
    let longitude = null;
    let destination = null;
    if (that.workStatus == '接客户') {
        latitude = Number(that.executeOrder.startPlaceLocation.latitude);
        longitude = Number(that.executeOrder.startPlaceLocation.longitude);
        destination = that.executeOrder.startPlace;
    } else {
        latitude = Number(that.executeOrder.endPlaceLocation.latitude);
        longitude = Number(that.executeOrder.endPlaceLocation.longitude);
        destination = that.executeOrder.endPlace;
    }
    //打开手机导航软件
    that.map.openMapApp({
        latitude: latitude,
        longitude: longitude,
        destination: destination
    });
},
```

### 四、搭建HBase

代驾过程中，我们需要保存驾驶途中的GPS定位，将来我们计算代驾真实里程的时候，就需要用到这些坐标点。那么这些定位点保存在MySQL中可以吗？当然不行，MySQL单表记录超过两千万就卡的不行。那么保存在MongoDB中可以吗？也不行，因为MongoDB里面的条件查询真的是超级蹩脚，所以我们想要用复杂条件检索数据，那么你还是打消用MongoDB的念头吧。

除了GPS定位数据之外，我们还要把代驾过程中的聊天对话的文字内容保存起来。这么看来，我们需要一个既能保存海量数据，又支持复杂条件检索的数据存储平台。那么HBase就再适合不过了。

HBase是一个分布式，版本化，面向列的开源数据库，构建在 Apache Hadoop和 Apache  ZooKeeper之上。HBase是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBase技术可在廉价PC  Server上搭建起大规模结构化存储集群。HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库，HBase基于列的而不是基于行的模式。

#### 1.HBase的特点

1. 海量数据存储，HBase中的表可以容纳上百亿行x上百万列的数据。
2. 列式存储，HBase中的数据是基于列进行存储的，能够动态的增加和删除列。
3. 准实时查询，HBase在海量的数据量下能够接近准实时的查询（百毫秒以内）
4. 多版本，HBase中每一列的数据都有多个版本。
5. 高可靠性，HBase中的数据存储于HDFS中且依赖于Zookeeper进行Master和RegionServer的协调管理。

#### 2.Phoenix

HBase的语法可读性差，复杂查询不方便。Phoenix是给HBase添加了一个语法表示层，允许我们用SQL语句读写HBase中的数据，可以做联机事务处理，拥有低延迟的特性，这就让我方便多了。Phoenix会把SQL编译成一系列的Hbase的scan操作，然后把scan结果生成标准的JDBC结果集，处理千万级行的数据也只用毫秒或秒级就搞定。而且Phoenix还支持MyBatis框架。

#### 3.表结构

- 创建逻辑库

```sql
CREATE SCHEMA hxds;
USE hxds;
```

- 创建数据表

接下来我们要创建`order_voice_text`、`order_monitoring`和`order_gps`数据表。

其中`order_voice_text`表用于存放司乘对话内容的文字内容。

| 序号 | 字段        | 类型    | 备注                          |
| ---- | ----------- | ------- | ----------------------------- |
| 1    | id          | BIGINT  | 主键                          |
| 2    | uuid        | VARCHAR | 唯一标识                      |
| 3    | order_id    | BIGINT  | 订单号                        |
| 4    | record_file | VARCHAR | 保存文件名                    |
| 5    | text        | VARCHAR | 文本内容                      |
| 6    | label       | VARCHAR | 审核结果（normal）            |
| 7    | suggestion  | VARCHAR | 后续建议（pass\block\review） |
| 8    | keywords    | VARCHAR | 关键字                        |
| 9    | create_time | DATE    | 创建日期时间                  |

```sql
CREATE TABLE hxds.order_voice_text(
    "id" BIGINT NOT NULL PRIMARY KEY, 
    "uuid" VARCHAR,
    "order_id" BIGINT,
    "record_file" VARCHAR,
    "text" VARCHAR,
    "label" VARCHAR,
    "suggestion" VARCHAR,
    "keywords" VARCHAR,
    "create_time" DATE
);


CREATE SEQUENCE hxds.ovt_sequence START WITH 1 INCREMENT BY 1;

CREATE INDEX ovt_index_1 ON hxds.order_voice_text("uuid");
CREATE INDEX ovt_index_2 ON hxds.order_voice_text("order_id");
CREATE INDEX ovt_index_3 ON hxds.order_voice_text("label");
CREATE INDEX ovt_index_4 ON hxds.order_voice_text("suggestion");
CREATE INDEX ovt_index_5 ON hxds.order_voice_text("create_time");
```

```java
@Data
public class OrderVoiceTextEntity {
    private Long id;
    private String uuid;
    private Long orderId;
    private String recordFile;
    private String text;
    private String label;
    private String suggestion;
    private String keywords;
    private String createTime;
}
```

`order_monitoring`表存储AI分析对话内容的安全评级结果。

| 序号 | 字段        | 类型    | 备注               |
| ---- | ----------- | ------- | ------------------ |
| 1    | id          | BIGINT  | 主键               |
| 2    | order_id    | BIGINT  | 订单号             |
| 3    | status      | TINYINT | 状态               |
| 4    | records     | INTEGER | 录音文件的数量     |
| 5    | safety      | VARCHAR | 安全等级           |
| 6    | reviews     | INTEGER | 需要人工审核的数量 |
| 7    | alarm       | TINYINT | 是否报警           |
| 8    | create_time | DATE    | 创建日期时间       |

```sql
CREATE TABLE hxds.order_monitoring
(
    "id"          BIGINT NOT NULL PRIMARY KEY,
    "order_id"    BIGINT,
    "status"      TINYINT,
    "records"     INTEGER,
    "safety"       VARCHAR,
    "reviews"     INTEGER,
    "alarm"       TINYINT,
    "create_time" DATE
);

CREATE INDEX om_index_1 ON hxds.order_monitoring("order_id");
CREATE INDEX om_index_2 ON hxds.order_monitoring("status");
CREATE INDEX om_index_3 ON hxds.order_monitoring("safety");
CREATE INDEX om_index_4 ON hxds.order_monitoring("reviews");
CREATE INDEX om_index_5 ON hxds.order_monitoring("alarm");
CREATE INDEX om_index_6 ON hxds.order_monitoring("create_time");

CREATE SEQUENCE hxds.om_sequence START WITH 1 INCREMENT BY 1;
```

```java
@Data
public class OrderMonitoringEntity {
    private Long id;
    private Long orderId;
    private Byte status;
    private Integer records;
    private String safety;
    private Integer reviews;
    private Byte alarm;
    private String createTime;
}
```

`order_gps`表保存的时候代驾过程中的GPS定位。

| 序号 | 字段        | 类型    | 备注         |
| ---- | ----------- | ------- | ------------ |
| 1    | id          | BIGINT  | 主键         |
| 2    | order_id    | BIGINT  | 订单号       |
| 3    | driver_id   | BIGINT  | 司机编号     |
| 4    | customer_id | BIGINT  | 乘客编号     |
| 5    | latitude    | VARCHAR | 纬度         |
| 6    | longitude   | VARCHAR | 经度         |
| 7    | speed       | VARCHAR | 速度         |
| 8    | create_time | DATE    | 创建日期时间 |

```sql
CREATE TABLE hxds.order_gps(
    "id" BIGINT NOT NULL PRIMARY KEY, 
    "order_id" BIGINT,
    "driver_id" BIGINT,
    "customer_id" BIGINT,
    "latitude" VARCHAR,
    "longitude" VARCHAR,
    "speed" VARCHAR,
    "create_time" DATE
);

CREATE SEQUENCE og_sequence START WITH 1 INCREMENT BY 1;

CREATE INDEX og_index_1 ON hxds.order_gps("order_id");
CREATE INDEX og_index_2 ON hxds.order_gps("driver_id");
CREATE INDEX og_index_3 ON hxds.order_gps("customer_id");
CREATE INDEX og_index_4 ON hxds.order_gps("create_time");
```

```java
@Data
public class OrderGpsEntity {
    private Long id;
    private Long orderId;
    private Long driverId;
    private Long customerId;
    private String latitude;
    private String longitude;
    private String speed;
    private String createTime;
}
```

- yml中整合Phoenix

```yml
datasource:
	driver-class-name: org.apache.phoenix.queryserver.client.Driver
	url: jdbc:phoenix:thin:url=http://127.0.0.1:8765;serialization=PROTOBUF
	type: com.alibaba.druid.pool.DruidDataSource
	druid:
		test-on-borrow: true
		test-while-idle: true
		max-active: 8
		min-idle: 4
```

#### 4.司乘对话保存后端逻辑

司乘聊天的文本，我们可以通过MyBatis框架保存到HBase里面。至于说司乘聊天的录音文件，我们可以保存到Minio里面。只要开始代驾，小程序就要录制司乘对话，直到代驾结束，才停止录音。

```sql
<insert id="insert" parameterType="com.example.hxds.nebula.db.pojo.OrderVoiceTextEntity">
    UPSERT INTO hxds.order_voice_text("id", "uuid", "order_id", "record_file", "text", "create_time")
    VALUES(NEXT VALUE FOR hxds.ovt_sequence, '${uuid}', #{orderId}, '${recordFile}', '${text}', NOW())
</insert>
```

```java
@Service
@Slf4j
public class MonitoringServiceImpl implements MonitoringService {
    @Resource
    private OrderVoiceTextDao orderVoiceTextDao;
    
    @Resource
    private VoiceTextCheckTask voiceTextCheckTask;

    @Value("${minio.endpoint}")
    private String endpoint;

    @Value("${minio.access-key}")
    private String accessKey;

    @Value("${minio.secret-key}")
    private String secretKey;

    @Override
    @Transactional
    public void monitoring(MultipartFile file, String name, String text) {
        //把录音文件上传到Minio
        try {
            MinioClient client = new MinioClient.Builder().endpoint(endpoint).credentials(accessKey, secretKey).build();
            client.putObject(
                    PutObjectArgs.builder().bucket("hxds-record").object(name)
                            .stream(file.getInputStream(), -1, 20971520)
                            .contentType("audio/x-mpeg")
                            .build());
        } catch (Exception e) {
            log.error("上传代驾录音文件失败", e);
            throw new HxdsException("上传代驾录音文件失败");
        }
        
        OrderVoiceTextEntity entity = new OrderVoiceTextEntity();
        
        //文件名格式例如:2156356656617-1.mp3，我们要解析出订单号
        String[] temp = name.substring(0, name.indexOf(".mp3")).split("-");
        Long orderId = Long.parseLong(temp[0]);

        String uuid = IdUtil.simpleUUID();
        entity.setUuid(uuid);
        entity.setOrderId(orderId);
        entity.setRecordFile(name);
        entity.setText(text);
        //把文稿保存到HBase
        int rows = orderVoiceTextDao.insert(entity);
        if (rows != 1) {
            throw new HxdsException("保存录音文稿失败");
        }

        //执行文稿内容审查
		voiceTextCheckTask.checkText(orderId,text,uuid);
    }
}
```

我们要对文字内容加以审核，看看对话的内容是否包含色情或者暴力的成分，我们要对聊天内容做一个安全评级。如果用人工去监视司机和乘客之间的对话，那成本可太高了，而且也雇不起那么多人，所以我们要用AI去监控对话的内容。

腾讯云的数据万象服务为我们提供了图像、视频和文本内容的审核。就拿文本来说把，数据万象可以对用户上传的文本进行内容安全识别，能够做到识别准确率高、召回率高，多维度覆盖对内容识别的要求，并实时更新识别服务的识别标准和能力。

1. 能够对文本文件进行多样化场景检测，精准识别文本中出现可能令人反感、不安全或不适宜的内容，有效降低内容违规风险与有害信息识别成本。
2. 能够精准识别涉黄等有害内容，支持用户配置词库，打击自定义的违规文本。文本内容安全服务能检测内容的危险等级，对于高危部分直接过滤，对于可疑部分提交人工复审，从而节省识别人力，降低业务风险。
3. 业务可用性不低于99.9%，专业团队7 × 24小时实时提供技术支持。
4. 请求毫秒级响应，结果秒级返回，超低延迟助力业务“快人一步”。
5. 多集群部署，每秒超万级并发，支持动态扩容，无需担心性能损耗。

数据万象的文本审核计费规则：每1000个utf8编码计算为一条，每种场景单独计算，比如勾选鉴黄、广告两种场景，那么会计算为审核2次。

调用数据万象接口审核文字内容，然后还要更新`order_monitoring`表中的记录，根据数据万象审核的结果更新订单的安全等级。这些操作都是需要消耗时间的，所以我们应该交给异步线程任务去做。

```java
@Component
@Slf4j
public class VoiceTextCheckTask {
    @Value("${tencent.cloud.appId}")
    private String appId;

    @Value("${tencent.cloud.secretId}")
    private String secretId;

    @Value("${tencent.cloud.secretKey}")
    private String secretKey;

    @Value("${tencent.cloud.bucket-public}")
    private String bucketPublic;

    @Resource
    private OrderVoiceTextDao orderVoiceTextDao;

    @Resource
    private OrderMonitoringDao orderMonitoringDao;

    @Async
    @Transactional
    public void checkText(long orderId, String content, String uuid) {

        String label = "Normal"; //审核结果
        String suggestion = "Pass"; //后续建议
        
        //后续建议模板
        HashMap<String, String> template = new HashMap() {{
            put("0", "Pass");
            put("1", "Block");
            put("2", "Review");
        }};

        if (StrUtil.isNotBlank(content)) {
            COSCredentials cred = new BasicCOSCredentials(secretId, secretKey);
            Region region = new Region("ap-beijing");
            ClientConfig clientConfig = new ClientConfig(region);
            COSClient client = new COSClient(cred, clientConfig);
            
            // 这里是终点
            TextAuditingRequest request = new TextAuditingRequest();
            request.setBucketName(bucketPublic);
            request.getInput().setContent(Base64.encode(content));
            request.getConf().setDetectType("all");

            TextAuditingResponse response = client.createAuditingTextJobs(request);
            AuditingJobsDetail detail = response.getJobsDetail();
            String state = detail.getState();
            ArrayList keywords = new ArrayList();
            if ("Success".equals(state)) {
                label = detail.getLabel(); //检测结果
                String result = detail.getResult(); //后续建议
                suggestion = template.get(result);
                List<SectionInfo> list = detail.getSectionList();   //违规关键词

                for (SectionInfo info : list) {
                    String keywords_1 = info.getPornInfo().getKeywords();
                    String keywords_2 = info.getIllegalInfo().getKeywords();
                    String keywords_3 = info.getAbuseInfo().getKeywords();
                    if (keywords_1.length() > 0) {
                        List temp = Arrays.asList(keywords_1.split(","));
                        keywords.addAll(temp);
                    }
                    if (keywords_2.length() > 0) {
                        List temp = Arrays.asList(keywords_2.split(","));
                        keywords.addAll(temp);
                    }
                    if (keywords_3.length() > 0) {
                        List temp = Arrays.asList(keywords_3.split(","));
                        keywords.addAll(temp);
                    }
                }
            }
            Long id = orderVoiceTextDao.searchIdByUuid(uuid);
            if (id == null) {
                throw new HxdsException("没有找到代驾语音文本记录");
            }
            HashMap param = new HashMap();
            param.put("id", id);
            param.put("label", label);
            param.put("suggestion", suggestion);
            param.put("keywords", ArrayUtil.join(keywords.toArray(), ","));
            
            //更新数据表中该文本的审核结果
            int rows = orderVoiceTextDao.updateCheckResult(param);
            if (rows != 1) {
                throw new HxdsException("更新内容检查结果失败");
            }
            log.debug("更新内容验证成功");
            
            //查询该订单有多少个录音文本和需要人工审核的文本
            HashMap map = orderMonitoringDao.searchOrderRecordsAndReviews(orderId);
            
            id = MapUtil.getLong(map, "id");
            Integer records = MapUtil.getInt(map, "records");
            Integer reviews = MapUtil.getInt(map, "reviews");
            OrderMonitoringEntity entity = new OrderMonitoringEntity();
            entity.setId(id);
            entity.setOrderId(orderId);
            entity.setRecords(records + 1);
            if (suggestion.equals("Review")) {
                entity.setReviews(reviews + 1);
            }
            if (suggestion.equals("Block")) {
                entity.setSafety("danger");
            }
            
            //更新order_monitoring表中的记录
            orderMonitoringDao.updateOrderMonitoring(entity);

        }
    }
}
```



#### 5.司乘对话前端逻辑

使用小程序的同声传译插件可以实现录音，并且把录音中的语音部分转换成文本。

```js
var plugin = requirePlugin("WechatSI")
let manager = plugin.getRecordRecognitionManager()

//开始录音的回调函数
manager.onStart = function(res) {
    console.log("成功开始录音识别", res)
}

//从语音中识别出文字，会执行该回调函数
manager.onRecognize = function(res) {
    console.log("current result", res.result)
}

//录音结束的回调函数
manager.onStop = function(res) {
    console.log("record file path", res.tempFilePath)
    console.log("result", res.result)
}

//出现异常的回调函数
manager.onError = function(res) {
    console.error("error msg", res.msg)
}

//开始录音，并且识别中文语音内容
manager.start({duration:30000, lang: "zh_CN"})

// 在onload函数中，初始化录音管理器插件
onLoad: function() {
    let that = this;
    if (!that.reviewAuth) {
        ……
        let recordManager = plugin.getRecordRecognitionManager(); //初始化录音管理器
        recordManager.onRecognize = function(resp) {
          
        };
        recordManager.onStop = function(resp) {
            // console.log('record file path', resp.tempFilePath);
            // console.log('result', resp.result);
            if (that.workStatus == '开始代驾' && that.stopRecord == false) {
                that.recordManager.start({ duration: 20 * 1000, lang: 'zh_CN' });
            }

            let tempFilePath = resp.tempFilePath;
            //上传录音
            that.recordNum += 1;
            let data = {
                name: `${that.executeOrder.id}-${that.recordNum}.mp3`,
                text: resp.result
            };
            //console.log(data);
            //upload上传文件函数封装在main.js文件中，大家可以自己去查阅
            that.upload(that.url.uploadRecordFile, tempFilePath, data, function(resp) {
                console.log('录音上传成功');
            });
        };
        recordManager.onStart = function(resp) {
            console.log('成功开始录音识别');
            if (that.recordNum == 0) {
                uni.vibrateLong({
                    complete: function() {
                        uni.showToast({
                            icon: 'none',
                            title: '请提示客户系上安全带！'
                        });
                    }
                });
            }
        };
        recordManager.onError = function(resp) {
            console.error('录音识别故障', resp.msg);
        };
        that.recordManager = recordManager;
      
        that.ajax(that.url.searchDriverCurrentOrder,'POST',null, function(resp) {
            ……
            if (that.workStatus == '开始代驾') {
                that.recordManager.start({ duration: 20 * 1000, lang: 'zh_CN' });
            }
        })
        ……
    }
},
   
// 司机点击开始开始代驾按钮之后，就要开始录音，并且转换成文本内容。
startDrivingHandle: function() {
    let that = this;
    uni.showModal({
        title: '消息通知',
        content: '您已经接到客户，现在开始代驾？',
        success: function(resp) {
            if (resp.confirm) {
                that.stopRecord = false;
                let data = {
                    orderId: that.executeOrder.id,
                    customerId: that.executeOrder.customerId,
                    status: 4
                };
                that.ajax(that.url.updateOrderStatus, 'POST', data, function(resp) {
                    that.workStatus = '开始代驾';
                    uni.setStorageSync('workStatus', '开始代驾');
                    //开始录音
                    that.recordManager.start({ duration: 20 * 1000, lang: 'zh_CN' });
                });
            }
        }
    });
},
    
// 结束代驾的时候，要更新订单状态，然后停止录音。
endDrivingHandle: function() {
    let that = this;
    uni.showModal({
        title: '消息通知',
        content: '已经到达终点，现在结束代驾？',
        success: function(resp) {
            if (resp.confirm) {
                let data = {
                    orderId: that.executeOrder.id,
                    customerId: that.executeOrder.customerId,
                    status: 5
                };
                that.ajax(that.url.updateOrderStatus, 'POST', data, function(resp) {
                    that.stopRecord = true;
                    try {
                        that.recordManager.stop(); //停止录音
                        that.recordNum = 0;
                        that.stopRecord = false;
                        that.workStatus = '结束代驾';
                        uni.setStorageSync('workStatus', '结束代驾');
                        //页面发生跳转
                        uni.navigateTo({
                            url: '../../order/enter_fee/enter_fee?orderId=' + that.executeOrder.id + '&customerId=' + that.executeOrder.customerId
                        });
                    } catch (e) {
                        console.error(e);
                    }
                });
            }
        }
    });
}

```

#### 6.保存GPS定位数据

```sql
<insert id="insert" parameterType="com.example.hxds.nebula.db.pojo.OrderGpsEntity">
    UPSERT INTO hxds.order_gps("id", "order_id", "driver_id", "customer_id", "latitude", "longitude", "speed", "create_time")
    VALUES(NEXT VALUE FOR hxds.og_sequence, ${orderId}, ${driverId}, ${customerId}, '${latitude}', '${longitude}', '${speed}', NOW())
</insert>
```

```java
@Data
public class InsertOrderGpsVo extends OrderGpsEntity {
    @NotNull(message = "orderId不能为空")
    @Min(value = 1, message = "orderId不能小于1")
    @Schema(description = "订单ID")
    private Long orderId;

    @NotNull(message = "driverId不能为空")
    @Min(value = 1, message = "driverId不能小于1")
    @Schema(description = "司机ID")
    private Long driverId;

    @NotNull(message = "customerId不能为空")
    @Min(value = 1, message = "customerId不能小于1")
    @Schema(description = "客户ID")
    private Long customerId;


    @NotBlank(message = "latitude不能为空")
    @Pattern(regexp = "^(([1-8]\\d?)|([1-8]\\d))(\\.\\d{1,18})|90|0(\\.\\d{1,18})?$", message = "latitude内容不正确")
    @Schema(description = "纬度")
    private String latitude;

    @NotBlank(message = "longitude不能为空")
    @Pattern(regexp = "^(([1-9]\\d?)|(1[0-7]\\d))(\\.\\d{1,18})|180|0(\\.\\d{1,18})?$", message = "longitude内容不正确")
    @Schema(description = "经度")
    private String longitude;

    @Schema(description = "速度")
    private String speed;
}

@Data
@Schema(description = "添加订单GPS记录的表单")
public class InsertOrderGpsForm {
    @NotEmpty(message = "list不能为空")
    @Schema(description = "GPS数据")
    private ArrayList<@Valid InsertOrderGpsVo> list;

}
```

```java
@Service
public class OrderGpsServiceImpl implements OrderGpsService {
    @Resource
    private OrderGpsDao orderGpsDao;

    @Override
    @Transactional
    public int insertOrderGps(ArrayList<InsertOrderGpsVo> list) {
        int rows = 0;
        for (OrderGpsEntity entity : list) {
            rows += orderGpsDao.insert(entity);
        }
        return rows;
    }
}

@RestController
@RequestMapping("/order/gps")
@Tag(name = "OrderGpsController", description = "订单GPS记录Web接口")
public class OrderGpsController {
    @Resource
    private OrderGpsService orderGpsService;

    @PostMapping("/insertOrderGps")
    @Operation(summary = "添加订单GPS记录")
    public R insertOrderGps(@RequestBody @Valid InsertOrderGpsForm form) {
        int rows = orderGpsService.insertOrderGps(form.getList());
        return R.ok().put("rows", rows);
    }
}
```

在`App.vue`页面`onLocationChange()`函数中，补充上传定位的代码。每凑够20个定位就上传一次，减少服务器的压力。

```js
//GPS定位变化就自动提交给后端
wx.onLocationChange(function(resp) {
	// 传给HBase时，需要单独将地址写在这里
    let baseUrl = 'http://192.168.99.106:8201/hxds-driver';
    if (workStatus == '开始接单') {
        ……
    } else if (workStatus == '开始代驾') {
        let executeOrder = uni.getStorageSync('executeOrder');
        if (executeOrder != null) {
            gps.push({
                orderId: executeOrder.id,
                customerId: executeOrder.customerId,
                latitude: latitude,
                longitude: longitude,
                speed: speed
            });

            //把GPS定位保存到HBase中，每凑够20个定位就上传一次，减少服务器的压力
            if (gps.length == 20) {
                uni.request({
                    url: `${baseUrl}/order/gps/insertOrderGps`,
                    method: 'POST',
                    header: {
                        token: uni.getStorageSync('token')
                    },
                    data: {
                        list: gps
                    },
                    success: function(resp) {
                        if (resp.statusCode == 401) {
                            uni.redirectTo({
                                url: '/pages/login/login'
                            });
                        } else if (resp.statusCode == 200 && resp.data.code == 200) {
                            let data = resp.data;
                            console.log("上传GPS成功");
                        } else {
                            console.error('保存GPS定位失败', resp.data);
                        }
                        gps.length = 0;
                    },
                    fail: function(error) {
                        console.error('保存GPS定位失败', error);
                    }
                });
            }
        }
    }

    //触发自定义事件
    uni.$emit('updateLocation', location);
});

```

### 五、本章总结

#### 1.利用AI对司乘对话内容安全评级

上一章我们把司乘对话的文本只是保存到HBase里面，这一章咱们必须对司乘之间的对话内容做监控，毕竟咱们做的是商业项目，保障乘客人身安全是首要的任务。腾讯的数据万象服务可以审核文本内容，咱们要把这个功能利用起来。我们调用腾讯万象的接口，审核司乘之间的对话内容，看看是否包含色情或者暴力的成分。审核的结果咱们要更新到HBase数据表中。

#### 2.利用大数据服务保存代驾途中GPS定位

在订单执行的过程中，我们要把司机的GPS定位坐标保存到HBase里面，将来我们计算实际代驾里程的时候，把这些GPS坐标点连成线，就知道实际里程是多少了。当然了，这些GPS坐标点另有用处。我们在MIS系统的订单模块中，展开订单详情面板的时候，如果订单已经结束了，咱们要把代驾行进的线路标记出来，于是就用上了这些GPS坐标点。

## 第六章 订单支付与分账

### 一、分账规则

在代驾账单中，包含了路桥费、停车费和其他费用，这些费用需要司机手动输入。当初在创建代驾订单的时候，我们就创建了账单记录。如果司机在下面的小程序页面输入相关费用之后，我们就可以计算订单的实际费用了，比如说统计实际里程、等时的时长、系统奖励，以及分账比例等等，这些都是需要调用各种子系统来获取的。

账单生成时，需要先更新此条代驾记录的订单表和账单表。

```java
// 账单表的更新
<update id="updateBillFee" parameterType="Map">
    UPDATE tb_order_bill
    SET total         = #{total},
        mileage_fee   = #{mileageFee},
        waiting_fee   = #{waitingFee},
        toll_fee      = #{tollFee},
        parking_fee   = #{parkingFee},
        other_fee    = #{otherFee},
        return_fee    = #{returnFee},
        incentive_fee = #{incentiveFee}
    WHERE order_id = #{orderId}
</update>

// 订单表的更新
<update id="updateOrderMileageAndFee" parameterType="Map">
    UPDATE tb_order
    SET real_mileage   = #{realMileage},
        return_mileage = #{returnMileage},
        incentive_fee  = #{incentiveFee},
        real_fee       = #{total}
    WHERE id = #{orderId}
</update>
```

更新后，建立分账记录，才能确定出微信渠道、平台、司机各自得多少钱。

#### 1.规则

微信支付平台对每一笔支付是要扣取渠道使用费的。交通出行类的APP，这个渠道费的扣点是0.6%，代驾平台的是对扣除掉渠道费的金额进行分账。代驾平台和司机的分账比例是变化的，具体的对应条件，我跟你具体说明一下。

1. 如果该司机当天有差评，司机只能分账20%，平台抽成80%

2. 如果付款金额小于50元
    （a） 当天没有取消过订单，并且完成订大于等于25个，系统抽成18%

    （b）当天取消订单小于3个，并且完成订单大于10个，系统抽成19%

    （c）不满足上面两种情况，系统抽成20%

3. 如果付款金额在50~100元之间
    （a）当天没有取消过订单，并且完成订大于等于25个，系统抽成15%

    （b）当天取消订单小于3个，并且完成订单大于10个，系统抽成16%

    （c）不满足上面两种情况，系统抽成17%

4. 如果付款金额在100元以上
    （a）当天没有取消过订单，并且完成订大于等于25个，系统抽成12%

    （b）当天取消订单小于3个，并且完成订单大于10个，系统抽成14%

    （c）不满足上面两种情况，系统抽成15%

API接口规范

| 序号 | 参数         | 备注                        |
| ---- | ------------ | --------------------------- |
| 1    | systemIncome | 系统分账收入                |
| 2    | driverIncome | 司机分账收入（扣除个税）    |
| 3    | paymentRate  | 微信支付接口渠道费率        |
| 4    | paymentFee   | 微信支付接口渠道费          |
| 5    | taxRate      | 替司机代缴个税的税率（19%） |
| 6    | taxFee       | 替司机代缴个税金额          |

#### 2.分帐表

我们先来了解一下`tb_order_profitsharing`表。

| 字段名称      | 数据类型 | 非空 | 备注信息                   |
| ------------- | -------- | ---- | -------------------------- |
| id            | bigint   | Y    | 主键                       |
| order_id      | bigint   | Y    | 订单ID                     |
| rule_id       | bigint   | Y    | 规则ID                     |
| amount_fee    | decimal  | Y    | 总费用                     |
| payment_rate  | decimal  | Y    | 微信支付接口的渠道费率     |
| payment_fee   | decimal  | Y    | 微信支付接口的渠道费       |
| tax_rate      | decimal  | Y    | 为代驾司机代缴税率         |
| tax_fee       | decimal  | Y    | 税率支出                   |
| system_income | decimal  | Y    | 企业分账收入               |
| driver_income | decimal  | Y    | 司机分账收入               |
| status        | tinyint  | Y    | 分账状态，1未分账，2已分账 |

添加分账时，status = 1。

```xml
<insert id="insert" parameterType="com.example.hxds.odr.db.pojo.OrderProfitsharingEntity">
    INSERT INTO tb_order_profitsharing
    SET order_id = #{orderId},
        rule_id = #{ruleId},
        amount_fee = #{amountFee},
        payment_rate = #{paymentRate},
        payment_fee = #{paymentFee},
        tax_rate = #{taxRate},
        tax_fee = #{taxFee},
        system_income = #{systemIncome},
        driver_income = #{driverIncome},
        `status` = 1
</insert>
```

#### 3.代价结束，更新账单

```java
@Service
public class OrderBillServiceImpl implements OrderBillService {
    ……
        
    @Override
    @Transactional
    @LcnTransaction
    public int updateBillFee(Map param) {
        //更新账单数据
        int rows = orderBillDao.updateBillFee(param);
        if (rows != 1) {
            throw new HxdsException("更新账单费用详情失败");
        }

        //更新订单数据
        rows = orderDao.updateOrderMileageAndFee(param);
        if (rows != 1) {
            throw new HxdsException("更新订单费用详情失败");
        }

        //添加分账单数据
        OrderProfitsharingEntity entity = new OrderProfitsharingEntity();
        entity.setOrderId(MapUtil.getLong(param, "orderId"));
        entity.setRuleId(MapUtil.getLong(param, "ruleId"));
        entity.setAmountFee(new BigDecimal((String) param.get("total")));
        entity.setPaymentRate(new BigDecimal((String) param.get("paymentRate")));
        entity.setPaymentFee(new BigDecimal((String) param.get("paymentFee")));
        entity.setTaxRate(new BigDecimal((String) param.get("taxRate")));
        entity.setTaxFee(new BigDecimal((String) param.get("taxFee")));
        entity.setSystemIncome(new BigDecimal((String) param.get("systemIncome")));
        entity.setDriverIncome(new BigDecimal((String) param.get("driverIncome")));
        rows = profitsharingDao.insert(entity);
        if (rows != 1) {
            throw new HxdsException("添加分账记录失败");
        }
        return rows;
    }
}

@Data
@Schema(description = "更新账单的表单")
public class UpdateBillFeeForm {

    @NotBlank(message = "total不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "total内容不正确")
    @Schema(description = "总金额")
    private String total;

    @NotBlank(message = "mileageFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "mileageFee内容不正确")
    @Schema(description = "里程费")
    private String mileageFee;

    @NotBlank(message = "waitingFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "waitingFee内容不正确")
    @Schema(description = "等时费")
    private String waitingFee;

    @NotBlank(message = "tollFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "tollFee内容不正确")
    @Schema(description = "路桥费")
    private String tollFee;

    @NotBlank(message = "parkingFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "parkingFee内容不正确")
    @Schema(description = "路桥费")
    private String parkingFee;

    @NotBlank(message = "otherFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "otherFee内容不正确")
    @Schema(description = "其他费用")
    private String otherFee;

    @NotBlank(message = "returnFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "returnFee内容不正确")
    @Schema(description = "返程费用")
    private String returnFee;

    @NotBlank(message = "incentiveFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "incentiveFee内容不正确")
    @Schema(description = "系统奖励费用")
    private String incentiveFee;

    @NotNull(message = "orderId不能为空")
    @Min(value = 1, message = "orderId不能小于1")
    @Schema(description = "订单ID")
    private Long orderId;

    @NotBlank(message = "realMileage不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d+$|^0\\.\\d+$|^[1-9]\\d*$", message = "realMileage内容不正确")
    @Schema(description = "代驾公里数")
    private String realMileage;

    @NotBlank(message = "returnMileage不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d+$|^0\\.\\d+$|^[1-9]\\d*$", message = "returnMileage内容不正确")
    @Schema(description = "返程公里数")
    private String returnMileage;

    @NotNull(message = "ruleId不能为空")
    @Min(value = 1, message = "ruleId不能小于1")
    @Schema(description = "规则ID")
    private Long ruleId;

    @NotBlank(message = "paymentRate不能为空")
    @Pattern(regexp = "^0\\.\\d+$|^[1-9]\\d*$|^0$", message = "paymentRate内容不正确")
    @Schema(description = "支付手续费率")
    private String paymentRate;

    @NotBlank(message = "paymentFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "paymentFee内容不正确")
    @Schema(description = "支付手续费")
    private String paymentFee;

    @NotBlank(message = "taxRate不能为空")
    @Pattern(regexp = "^0\\.\\d+$|^[1-9]\\d*$|^0$", message = "taxRate内容不正确")
    @Schema(description = "代缴个税费率")
    private String taxRate;

    @NotBlank(message = "taxFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "taxFee内容不正确")
    @Schema(description = "代缴个税")
    private String taxFee;

    @NotBlank(message = "systemIncome不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "systemIncome内容不正确")
    @Schema(description = "代驾系统分账收入")
    private String systemIncome;

    @NotBlank(message = "driverIncome不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "driverIncome内容不正确")
    @Schema(description = "司机分账收入")
    private String driverIncome;

}

@RestController
@RequestMapping("/bill")
@Tag(name = "OrderBillController", description = "订单费用账单Web接口")
public class OrderBillController {
    @Resource
    private OrderBillService orderBillService;
    
    @PostMapping("/updateBillFee")
    @Operation(summary = "更新订单账单费用")
    public R updateBillFee(@RequestBody @Valid UpdateBillFeeForm form) {
        Map param = BeanUtil.beanToMap(form);
        int rows = orderBillService.updateBillFee(param);
        return R.ok().put("rows", rows);
    }
}
```

### 二、里程与费用计算

#### 1.实际代驾里程计算

需要把HBase中该订单的GPS坐标取出来，然后把这些GPS坐标点连成线，就能计算出实际里程了。因为要计算GPS坐标点之间的距离，所以我们需要声明一个工具类。在`com.example.hxds.nebula.util`包中创建`LocationUtil.java`类。

```java
public class LocationUtil {
    private final static double EARTH_RADIUS = 6378.137;//地球半径

    private static double rad(double d) {
        return d * Math.PI / 180.0;
    }

    /**
     * 计算两点间距离
     *
     * @return double 距离 单位公里,精确到米
     */
    public static double getDistance(double lat1, double lng1, double lat2, double lng2) {
        double radLat1 = rad(lat1);
        double radLat2 = rad(lat2);
        double a = radLat1 - radLat2;
        double b = rad(lng1) - rad(lng2);
        double s = 2 * Math.asin(Math.sqrt(Math.pow(Math.sin(a / 2), 2) +
                Math.cos(radLat1) * Math.cos(radLat2) * Math.pow(Math.sin(b / 2), 2)));
        s = s * EARTH_RADIUS;
        s = new BigDecimal(s).setScale(3, RoundingMode.HALF_UP).doubleValue();
        return s;
    }
}
```

代驾里程计算方法

```java
@Service
public class OrderGpsServiceImpl implements OrderGpsService {
    ……
        
    @Override
    public String calculateOrderMileage(long orderId) {
        ArrayList<HashMap> list = orderGpsDao.searchOrderAllGps(orderId);
        double mileage = 0;
        for (int i = 0; i < list.size(); i++) {
            if (i != list.size() - 1) {
                HashMap map_1 = list.get(i);
                HashMap map_2 = list.get(i+1);
                double latitude_1 = MapUtil.getDouble(map_1, "latitude");
                double longitude_1 = MapUtil.getDouble(map_1, "longitude");
                double latitude_2 = MapUtil.getDouble(map_2, "latitude");
                double longitude_2 = MapUtil.getDouble(map_2, "longitude");
                double distance = LocationUtil.getDistance(latitude_1, longitude_1, latitude_2, longitude_2);
                mileage += distance;
            }
        }
        return mileage + "";
    }
}
```

#### 2.代驾费计算

- 计费标准

1. 代驾默认起步里程是8公里，收费根据时间段不同也有所调整。超出8公里的部分，每公里3.5元。
    | 时间段 | 基础里程 | 收费 |
    | — | — | — |
    | 06:00 - 22:00 | 8公里 | 35元 |
    | 22:00 - 23:00 | 8公里 | 50元 |
    | 23:00 - 00:00（次日） | 8公里 | 65元 |
    | 00:00 - 06:00 | 8公里 | 85元 |
2. 默认免费等时额度为10分钟，超出部分1元/分钟，等时费最高180元封顶。
3. 如果实际代驾里程不超过8公里，不收取返程费。超过8公里，则收取返程费，返程费 = 实际代驾里程 × 1.0元

| 序号 | 参数名称           | 备注信息                               |
| ---- | ------------------ | -------------------------------------- |
| 1    | amount             | 代驾费用（不包含停车费、路桥费等）     |
| 2    | mileageFee         | 里程费                                 |
| 3    | returnFee          | 返程费                                 |
| 4    | waitingFee         | 等时费                                 |
| 5    | returnMileage      | 返程里程                               |
| 6    | baseReturnMileage  | 返程里程基数（8公里）                  |
| 7    | exceedReturnPrice  | 返程里程收费价格（1元/公里）           |
| 8    | baseMinute         | 等时基数（10分钟）                     |
| 9    | exceedMinutePrice  | 等时基数外收费价格（1元/分钟）         |
| 10   | baseMileagePrice   | 代驾基础费（35-85元）                  |
| 11   | baseMileage        | 代驾基础里程（8公里）                  |
| 12   | exceedMileagePrice | 代驾基础里程之外收费价格（3.5元/公里） |

- 司机奖励费用

代驾平台对司机的奖励规则如下，如果当天完成代驾30单以上，每笔订单再补贴5元。代驾平台的补贴是直接发放到司机钱包里面，到时候我们修改司机钱包的余额，并且还要添加一条充值记录。

| 时间段                | 完成订单 | 平台奖励 |
| --------------------- | -------- | -------- |
| 06:00 - 18:00         | 10个     | 2元/单   |
| 18:00 - 23:00         | 5个      | 3元/单   |
| 23:00 - 00:00（次日） | 5个      | 2元/单   |
| 00:00 - 06:00         | 5个      | 2元/单   |

| 序号 | 参数名称     | 备注信息 |
| ---- | ------------ | -------- |
| 1    | incentiveFee | 奖励费   |

- 计算代驾费用的表单

```java
@Data
@Schema(description = "计算代驾费用的表单")
public class CalculateOrderChargeForm {

    @Schema(description = "代驾公里数")
    private String mileage;

    @Schema(description = "代驾开始时间")
    private String time;

    @Schema(description = "等时分钟")
    private Integer minute;
}
```

- 计算奖励的表单

```java
@Data
@Schema(description = "计算系统奖励的表单")
public class CalculateIncentiveFeeForm {

    @Schema(description = "司机ID")
    private long driverId;

    @NotBlank(message = "acceptTime不能为空")
    @Pattern(regexp = "^((((1[6-9]|[2-9]\\d)\\d{2})-(0?[13578]|1[02])-(0?[1-9]|[12]\\d|3[01]))|(((1[6-9]|[2-9]\\d)\\d{2})-(0?[13456789]|1[012])-(0?[1-9]|[12]\\d|30))|(((1[6-9]|[2-9]\\d)\\d{2})-0?2-(0?[1-9]|1\\d|2[0-8]))|(((1[6-9]|[2-9]\\d)(0[48]|[2468][048]|[13579][26])|((16|[2468][048]|[3579][26])00))-0?2-29-))\\s(20|21|22|23|[0-1]\\d):[0-5]\\d:[0-5]\\d$",
            message = "acceptTime内容不正确")
    @Schema(description = "接单时间")
    private String acceptTime;
}
```

- 计算订单分账的表单

```
@Data
@Schema(description = "计算订单分账的表单")
public class CalculateProfitsharingForm {

    @Schema(description = "订单ID")
    private Long orderId;

    @Schema(description = "待分账费用")
    private String amount;
}
```

- 更新订单账单费用的表单

```java
@Data
@Schema(description = "更新订单账单费用的表单")
public class UpdateBillFeeForm {

    @Schema(description = "总金额")
    private String total;

    @Schema(description = "里程费")
    private String mileageFee;

    @Schema(description = "等时费")
    private String waitingFee;

    @NotBlank(message = "tollFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "tollFee内容不正确")
    @Schema(description = "路桥费")
    private String tollFee;

    @NotBlank(message = "parkingFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "parkingFee内容不正确")
    @Schema(description = "路桥费")
    private String parkingFee;

    @NotBlank(message = "otherFee不能为空")
    @Pattern(regexp = "^[1-9]\\d*\\.\\d{1,2}$|^0\\.\\d{1,2}$|^[1-9]\\d*$", message = "otherFee内容不正确")
    @Schema(description = "其他费用")
    private String otherFee;

    @Schema(description = "返程费用")
    private String returnFee;

    @Schema(description = "系统奖励费用")
    private String incentiveFee;

    @NotNull(message = "orderId不能为空")
    @Min(value = 1, message = "orderId不能小于1")
    @Schema(description = "订单ID")
    private Long orderId;

    @Schema(description = "司机ID")
    private Long driverId;

    @Schema(description = "代驾公里数")
    private String realMileage;

    @Schema(description = "返程公里数")
    private String returnMileage;

    @Schema(description = "规则ID")
    private Long ruleId;

    @Schema(description = "支付手续费率")
    private String paymentRate;

    @Schema(description = "支付手续费")
    private String paymentFee;

    @Schema(description = "代缴个税费率")
    private String taxRate;

    @Schema(description = "代缴个税")
    private String taxFee;

    @Schema(description = "代驾系统分账收入")
    private String systemIncome;

    @Schema(description = "司机分账收入")
    private String driverIncome;

}
```

- 整体业务逻辑

```java
@Service
public class OrderServiceImpl implements OrderService {
        
    @Override
    @Transactional
    @LcnTransaction
    public int updateOrderBill(UpdateBillFeeForm form) {
        /*
         * 1.判断司机是否关联该订单
         */
        ValidDriverOwnOrderForm form_1 = new ValidDriverOwnOrderForm();
        form_1.setOrderId(form.getOrderId());
        form_1.setDriverId(form.getDriverId());
        R r = odrServiceApi.validDriverOwnOrder(form_1);
        boolean bool = MapUtil.getBool(r, "result");
        if (!bool) {
            throw new HxdsException("司机未关联该订单");
        }
        /*
         * 2.计算订单里程数据
         */
        CalculateOrderMileageForm form_2 = new CalculateOrderMileageForm();
        form_2.setOrderId(form.getOrderId());
        r = nebulaServiceApi.calculateOrderMileage(form_2);
        String mileage = (String) r.get("result");
        mileage=NumberUtil.div(mileage,"1000",1,RoundingMode.CEILING).toString();

        /*
         * 3.查询订单消息
         */
        SearchSettlementNeedDataForm form_3 = new SearchSettlementNeedDataForm();
        form_3.setOrderId(form.getOrderId());
        r = odrServiceApi.searchSettlementNeedData(form_3);
        HashMap map = (HashMap) r.get("result");
        String acceptTime = MapUtil.getStr(map, "acceptTime");
        String startTime = MapUtil.getStr(map, "startTime");
        int waitingMinute = MapUtil.getInt(map, "waitingMinute");
        String favourFee = MapUtil.getStr(map, "favourFee");

        /*
         * 4.计算代驾费
         */
        CalculateOrderChargeForm form_4 = new CalculateOrderChargeForm();
        form_4.setMileage(mileage);
        form_4.setTime(startTime.split(" ")[1]);
        form_4.setMinute(waitingMinute);
        r = ruleServiceApi.calculateOrderCharge(form_4);
        map = (HashMap) r.get("result");
        String mileageFee = MapUtil.getStr(map, "mileageFee");
        String returnFee = MapUtil.getStr(map, "returnFee");
        String waitingFee = MapUtil.getStr(map, "waitingFee");
        String amount = MapUtil.getStr(map, "amount");
        String returnMileage = MapUtil.getStr(map, "returnMileage");

        /*
         * 5.计算系统奖励费用
         */
        CalculateIncentiveFeeForm form_5 = new CalculateIncentiveFeeForm();
        form_5.setDriverId(form.getDriverId());
        form_5.setAcceptTime(acceptTime);
        r = ruleServiceApi.calculateIncentiveFee(form_5);
        String incentiveFee = (String) r.get("result");


        form.setMileageFee(mileageFee);
        form.setReturnFee(returnFee);
        form.setWaitingFee(waitingFee);
        form.setIncentiveFee(incentiveFee);
        form.setRealMileage(mileage);
        form.setReturnMileage(returnMileage);
        //计算总费用
        String total = NumberUtil.add(amount, form.getTollFee(), form.getParkingFee(), form.getOtherFee(), favourFee).toString();
        form.setTotal(total);

        /*
         * 6.计算分账数据
         */
        CalculateProfitsharingForm form_6 = new CalculateProfitsharingForm();
        form_6.setOrderId(form.getOrderId());
        form_6.setAmount(total);
        r = ruleServiceApi.calculateProfitsharing(form_6);
        map = (HashMap) r.get("result");
        long ruleId = MapUtil.getLong(map, "ruleId");
        String systemIncome = MapUtil.getStr(map, "systemIncome");
        String driverIncome = MapUtil.getStr(map, "driverIncome");
        String paymentRate = MapUtil.getStr(map, "paymentRate");
        String paymentFee = MapUtil.getStr(map, "paymentFee");
        String taxRate = MapUtil.getStr(map, "taxRate");
        String taxFee = MapUtil.getStr(map, "taxFee");
        form.setRuleId(ruleId);
        form.setPaymentRate(paymentRate);
        form.setPaymentFee(paymentFee);
        form.setTaxRate(taxRate);
        form.setTaxFee(taxFee);
        form.setSystemIncome(systemIncome);
        form.setDriverIncome(driverIncome);

        /*
         * 7.更新代驾费账单数据
         */
        r = odrServiceApi.updateBillFee(form);
        int rows = MapUtil.getInt(r, "rows");
        return rows;
    }
}
```

- 路桥费、停车费、其他费用由司机手动添加

添加完毕后，司机点击发送订单给乘客，然后乘客需要付款。

### 三、微信支付

乘客端小程序收到账单通知消息之后，跳转到`order.vue`页面。显示账单的各项数据，例如，起点、终点、订单号、下单时间、司机信息、基础费用、额外费用、总费用等内容。

```xml
<select id="searchOrderById" parameterType="Map" resultType="HashMap">
    SELECT CAST(o.id AS CHAR) AS id,
           CAST(o.driver_id AS CHAR) AS driverId,
           CAST(o.customer_id AS CHAR) AS customerId,
           o.start_place AS startPlace,
           o.start_place_location AS startPlaceLocation,
           o.end_place AS endPlace,
           o.end_place_location AS endPlaceLocation,
           CAST(b.total AS CHAR) AS total,
           CAST(b.real_pay AS CHAR) AS realPay,
           CAST(b.mileage_fee AS CHAR) AS mileageFee,
           CAST(o.favour_fee AS CHAR) AS favourFee,
           CAST(o.incentive_fee AS CHAR) AS incentiveFee,
           CAST(b.waiting_fee AS CHAR) AS waitingFee,
           CAST(b.return_fee AS CHAR) AS returnFee,
           CAST(b.parking_fee AS CHAR) AS parkingFee,
           CAST(b.toll_fee AS CHAR) AS tollFee,
           CAST(b.other_fee AS CHAR) AS otherFee,
           CAST(b.voucher_fee AS CHAR) AS voucherFee,
           CAST(o.real_mileage AS CHAR) AS realMileage,
           o.waiting_minute AS waitingMinute,
           b.base_mileage AS baseMileage,
           CAST(b.base_mileage_price AS CHAR) AS baseMileagePrice,
           CAST(b.exceed_mileage_price AS CHAR) AS exceedMileagePrice,
           b.base_minute AS baseMinute,
           CAST(b.exceed_minute_price AS CHAR) AS exceedMinutePrice,
           b.base_return_mileage AS baseReturnMileage,
           CAST(b.exceed_return_price AS CHAR) AS exceedReturnPrice,
           CAST(o.return_mileage AS CHAR) AS returnMileage,
           o.car_plate AS carPlate,
           o.car_type AS carType,
           o.status,
           DATE_FORMAT(o.create_time, '%Y-%m-%d %H:%i:%s') AS createTime
    FROM tb_order o
    JOIN tb_order_bill b ON o.id = b.order_id
    WHERE o.id = #{orderId}
    <if test="driverId!=null">
        AND driver_id = #{driverId}
    </if>
    <if test="customerId!=null">
        AND customer_id = #{customerId}
    </if>
</select>
```

#### 1.支付账单的创建

如果乘客想要付款，需要先创建微信支付账单。但是创建微信支付账单的流程一点也不简单，比如乘客使用了代金券，咱们就要先验证代金券是否有效，然后扣减支付金额，更新账单表记录。等到创建了微信支付账单之后，我们还要把预支付ID（PrepayId）更新到订单表中。

**代金券`tb_voucher`数据表的结构：**

| **段**      | **类型** | **非空** | **备注**                                                     |
| ----------- | -------- | -------- | ------------------------------------------------------------ |
| id          | bigint   | True     | 主键                                                         |
| uuid        | varchar  | True     | UUID                                                         |
| name        | varchar  | True     | 代金券标题                                                   |
| remark      | varchar  | False    | 描述文字                                                     |
| tag         | varchar  | False    | 代金券标签，例如新人专用                                     |
| total_quota | int      | True     | 代金券数量，如果是0，则是无限量                              |
| take_count  | int      | True     | 实际领取数量                                                 |
| used_count  | int      | True     | 已经使用的数量                                               |
| discount    | decimal  | True     | 代金券面额                                                   |
| with_amount | decimal  | True     | 最少消费金额才能使用代金券                                   |
| type        | tinyint  | True     | 代金券赠送类型，如果是1则通用券，用户领取；如果是2，则是注册赠券 |
| limit_quota | smallint | True     | 用户领券限制数量，如果是0，则是不限制；默认是1，限领一张     |
| status      | tinyint  | True     | 代金券状态，如果是1则是正常可用；如果是2则是过期; 如果是3则是下架 |
| time_type   | tinyint  | False    | 有效时间限制，如果是1，则基于领取时间的有效天数days；如果是2，则start_time和end_time是优惠券有效期； |
| days        | smallint | False    | 基于领取时间的有效天数days                                   |
| start_time  | datetime | False    | 代金券开始时间                                               |
| end_time    | datetime | False    | 代金券结束时间                                               |
| create_time | datetime | True     | 创建时间                                                     |

至于哪些用户领取了代金券，我们要记录下来，

**`tb_voucher_customer`表结构**

| **字段**    | **类型** | **非空** | **备注**                                                     |
| ----------- | -------- | -------- | ------------------------------------------------------------ |
| id          | bigint   | True     | 主键                                                         |
| customer_id | bigint   | True     | 客户ID                                                       |
| voucher_id  | bigint   | True     | 代金券ID                                                     |
| status      | tinyint  | True     | 使用状态, 如果是1则未使用；如果是2则已使用；如果是3则已过期；如果是4则已经下架； |
| used_time   | datetime | False    | 使用代金券的时间                                             |
| start_time  | datetime | False    | 有效期开始时间                                               |
| end_time    | datetime | False    | 有效期截至时间                                               |
| order_id    | bigint   | False    | 订单ID                                                       |
| create_time | datetime | True     | 创建时间                                                     |

**创建微信订单的业务逻辑**

```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    @Resource
    private OdrServiceApi odrServiceApi;

    @Resource
    private VhrServiceApi vhrServiceApi;

    @Resource
    private CstServiceApi cstServiceApi;

    @Resource
    private DrServiceApi drServiceApi;

    @Resource
    private MyWXPayConfig myWXPayConfig;
    
    @Override
    @Transactional
    @LcnTransaction
    public HashMap createWxPayment(long orderId, long customerId, Long voucherId) {
        /*
         * 1.先查询订单是否为6状态，其他状态都不可以生成支付订单
         */
        ValidCanPayOrderForm form_1 = new ValidCanPayOrderForm();
        form_1.setOrderId(orderId);
        form_1.setCustomerId(customerId);
        R r = odrServiceApi.validCanPayOrder(form_1);
        HashMap map = (HashMap) r.get("result");
        String amount = MapUtil.getStr(map, "realFee");
        String uuid = MapUtil.getStr(map, "uuid");
        long driverId = MapUtil.getLong(map, "driverId");
        String discount = "0.00";
        if (voucherId != null) {
            /*
             * 2.查询代金券是否可以使用，并绑定
             */
            UseVoucherForm form_2 = new UseVoucherForm();
            form_2.setCustomerId(customerId);
            form_2.setVoucherId(voucherId);
            form_2.setOrderId(orderId);
            form_2.setAmount(amount);
            r = vhrServiceApi.useVoucher(form_2);
            discount = MapUtil.getStr(r, "result");
        }
        if (new BigDecimal(amount).compareTo(new BigDecimal(discount)) == -1) {
            throw new HxdsException("总金额不能小于优惠劵面额");
        }
        /*
         * 3.修改实付金额
         */
        amount = NumberUtil.sub(amount, discount).toString();
        UpdateBillPaymentForm form_3 = new UpdateBillPaymentForm();
        form_3.setOrderId(orderId);
        form_3.setRealPay(amount);
        form_3.setVoucherFee(discount);
        odrServiceApi.updateBillPayment(form_3);

        /*
         * 4.查询用户的OpenID字符串
         */
        SearchCustomerOpenIdForm form_4 = new SearchCustomerOpenIdForm();
        form_4.setCustomerId(customerId);
        r = cstServiceApi.searchCustomerOpenId(form_4);
        String customerOpenId = MapUtil.getStr(r, "result");

        /*
         * 5.查询司机的OpenId字符串
         */
        SearchDriverOpenIdForm form_5 = new SearchDriverOpenIdForm();
        form_5.setDriverId(driverId);
        r = drServiceApi.searchDriverOpenId(form_5);
        String driverOpenId = MapUtil.getStr(r, "result");

        /*
         * 6.TODO 创建支付订单
         */
        

    }
}
```

#### 2.微信支付资质申请

对于商家来说，想要开通微信支付，必须要去微信商户平台注册（https://pay.weixin.qq.com/index.php/core/home/login?return_url=%2F），然后把工商登记证明、企业银行账户开户证明、组织机构代码证提交上去，经过半天的审核，如果没有问题，你就开通了微信支付功能。

如果想要在网站或者小程序上面使用微信支付，还要在微信公众平台上面关联你自己的微信商户账号。前提是你的微信开发者账号必须是企业身份，个人身份的开发者账号是无法调用微信支付API的。

#### 3.微信支付接口

如果想要发起微信支付，首先要调用微信支付API接口，创建支付账单，然后把支付账单的信息传递给乘客端小程序，就能看到微信APP弹出付款界面了。

相关API:

| **字段名**   | **变量名**       | **描述**                                    |
| ------------ | ---------------- | ------------------------------------------- |
| 小程序ID     | appid            | 微信分配的小程序ID                          |
| 商户号       | mch_id           | 微信支付分配的商户号                        |
| 随机字符串   | nonce_str        | 随机字符串，长度要求在32位以内              |
| 签名         | sign             | 通过签名算法计算得出的签名值                |
| 签名类型     | sign_type        | 签名类型，默认为MD5，支持HMAC-SHA256和MD5。 |
| 商品描述     | body             | 商品简单描述，该字段请按照规范传递          |
| 附加数据     | attach           | 附加数据，属于自定义参数                    |
| 用户标识     | openid           | 付款人的openId                              |
| 商户订单号   | out_trade_no     | 商户系统内部订单号，要求32个字符内          |
| 标价金额     | total_fee        | 订单总金额，单位为分                        |
| 终端IP       | spbill_create_ip | 调用微信支付API的机器IP（可以随便写）       |
| 通知地址     | notify_url       | 异步接收微信支付结果通知的回调地址          |
| 交易类型     | trade_type       | 必须是JSAPI                                 |
| 是否需要分账 | profit_sharing   | Y，需要分账；N，不分账                      |

微信支付平台给我们返回的响应中包含了下面这些参数。

| **参数**    | **含义**             | **例子**                         |
| ----------- | -------------------- | -------------------------------- |
| return_code | 通信状态码           | SUCCESS                          |
| result_code | 业务状态码           | SUCCESS                          |
| app_id      | 微信公众账号APPID    | wx8888888888888888               |
| mch_id      | 商户ID               | 1900000109                       |
| nonce_str   | 随机字符串           | 5K8264ILTKCH16CQ2502SI8ZNMTM67VS |
| sign        | 数字签名             | C380BEC2BFD727A4B6845133519F3AD6 |
| trade_type  | 交易类型             | NATIVE                           |
| prepay_id   | 预支付交易会话标识ID | wx201410272009395522657a6905100  |

完善上节支付订单的业务逻辑

```java
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    @Resource
    private MyWXPayConfig myWXPayConfig;
    
    @Override
    @Transactional
    @LcnTransaction
    public HashMap createWxPayment(long orderId, long customerId, Long voucherId) {
        ……

        /*
         * 6.创建支付订单
         */
        try {
            WXPay wxPay = new WXPay(myWXPayConfig);
            HashMap param = new HashMap();
            param.put("nonce_str", WXPayUtil.generateNonceStr());//随机字符串
            param.put("body", "代驾费");
            param.put("out_trade_no", uuid);
            //充值金额转换成分为单位，并且让BigDecimal取整数
            //amount="1.00";
            param.put("total_fee", NumberUtil.mul(amount, "100").setScale(0, RoundingMode.FLOOR).toString());
            param.put("spbill_create_ip", "127.0.0.1");
            //TODO 这里要修改成内网穿透的公网URL
            param.put("notify_url", "http://demo.com");
            param.put("trade_type", "JSAPI");
            param.put("openid", customerOpenId);
            param.put("attach", driverOpenId);
            param.put("profit_sharing", "Y"); //支付需要分账
            
            //创建支付订单
            Map<String, String> result = wxPay.unifiedOrder(param);
            
            //预支付交易会话标识ID
            String prepayId = result.get("prepay_id");
            if (prepayId != null) {
                /*
                 * 7.更新订单记录中的prepay_id字段值
                 */
                UpdateOrderPrepayIdForm form_6 = new UpdateOrderPrepayIdForm();
                form_6.setOrderId(orderId);
                form_6.setPrepayId(prepayId);
                odrServiceApi.updateOrderPrepayId(form_6);

                //准备生成数字签名用的数据
                map.clear();
                map.put("appId", myWXPayConfig.getAppID());
                String timeStamp = new Date().getTime() + "";
                map.put("timeStamp", timeStamp);
                String nonceStr = WXPayUtil.generateNonceStr();
                map.put("nonceStr", nonceStr);
                map.put("package", "prepay_id=" + prepayId);
                map.put("signType", "MD5");
                
                //生成数据签名
                String paySign = WXPayUtil.generateSignature(map, myWXPayConfig.getKey()); //生成数字签名
                
                map.clear(); //清理HashMap，放入结果
                map.put("package", "prepay_id=" + prepayId);
                map.put("timeStamp", timeStamp);
                map.put("nonceStr", nonceStr);
                map.put("paySign", paySign);
                //uuid用于付款成功后，移动端主动请求更新充值状态
                map.put("uuid", uuid); 
                return map;
            } else {
                log.error("创建支付订单失败");
                throw new HxdsException("创建支付订单失败");
            }
        } catch (Exception e) {
            log.error("创建支付订单失败", e);
            throw new HxdsException("创建支付订单失败");
        }
    }
}
```

#### 4.小程序唤起支付窗口

用户收到付款消息后，点击付款按钮，小程序调用`uni.requestPayment()`函数的时候传入这些数据，小程序就能弹出付款窗口了。

```js
<view class="operate-container">
  <button class="btn" @tap="payHandle">立即付款</button>
</view>

payHandle: function() {
    let that = this;
    uni.showModal({
        title: '提示消息',
        content: '您确定支付该订单？',
        success: function(resp) {
            if (resp.confirm) {
                let data = {
                    orderId: that.orderId
                };
                that.ajax(that.url.createWxPayment, 'POST', data, function(resp) {
                    let result = resp.data.result;
                    let pk = result.package;
                    let timeStamp = result.timeStamp;
                    let nonceStr = result.nonceStr;
                    let paySign = result.paySign;
                    let uuid = result.uuid;
                    uni.requestPayment({
                        timeStamp: timeStamp,
                        nonceStr: nonceStr,
                        package: pk,
                        paySign: paySign,
                        signType: 'MD5',
                        success: function() {
                            console.log('付款成功');
                            //TODO 主动发起查询请求
                        },
                        fail: function(error) {
                            console.error(error);
                            uni.showToast({
                                icon: 'error',
                                title: '付款失败'
                            });
                        }
                    });
                });
            }
        }
    });
}
```

#### 5.后端接收付款结果

微信平台发送付款结果通知的Request里面是XML格式的数据，所以要从中提取出我们需要的关键数据，这部分的文档规范，大家可以查阅微信支付的官方资料（https://pay.weixin.qq.com/wiki/doc/api/native.php?chapter=9_7&index=8）。而且这个请求并不是只发送一次，如果商户系统没有接收到请求，微信平台会每隔一小段时间再发送一次付款结果。

| **参数**       | **含义**   | **类型** | **示例**                         |
| -------------- | ---------- | -------- | -------------------------------- |
| return_code    | 通信状态码 | String   | SUCCESS                          |
| result_code    | 业务状态码 | String   | SUCCESS /  FAIL                  |
| sign           | 数字签名   | String   | C380BEC2BFD727A4B6845133519F3AD6 |
| out_trade_no   | 商品订单ID | String   | 1212321211201407033568112322     |
| transaction_id | 支付订单ID | String   | 1217752501201407033233368018     |
| total_fee      | 订单金额   | int      | 150                              |
| time_end       | 支付的时间 | String   | 20141030133525                   |

```xml
<xml>
    <appid><![CDATA[wx2421b1c4370ec43b]]></appid>
    <attach><![CDATA[支付测试]]></attach>
    <bank_type><![CDATA[CFT]]></bank_type>
    <fee_type><![CDATA[CNY]]></fee_type>
    <is_subscribe><![CDATA[Y]]></is_subscribe>
    <mch_id><![CDATA[10000100]]></mch_id>
    <nonce_str><![CDATA[5d2b6c2a8db53831f7eda20af46e531c]]></nonce_str>
    <openid><![CDATA[oUpF8uMEb4qRXf22hE3X68TekukE]]></openid>
    <out_trade_no><![CDATA[1409811653]]></out_trade_no>
    <result_code><![CDATA[SUCCESS]]></result_code>
    <return_code><![CDATA[SUCCESS]]></return_code>
    <sign><![CDATA[B552ED6B279343CB493C5DD0D78AB241]]></sign>
    <time_end><![CDATA[20140903131540]]></time_end>
    <total_fee>1</total_fee>
    <coupon_fee><![CDATA[10]]></coupon_fee>
    <coupon_count><![CDATA[1]]></coupon_count>
    <coupon_type><![CDATA[CASH]]></coupon_type>
    <coupon_id><![CDATA[10000]]></coupon_id>
    <trade_type><![CDATA[JSAPI]]></trade_type>
    <transaction_id><![CDATA[1004400740201409030005092168]]></transaction_id>
</xml> 

```

返回给微信平台的响应内容也必须是XML格式的，否则微信平台就会认为你没收到付款结果通知。

```xml
<xml>
    <return_code><![CDATA[SUCCESS]]></return_code>
    <return_msg><![CDATA[OK]]></return_msg>
</xml> 
```

我们需要创建一个Web方法来接收微信平台发送过来的付款结果通知，所以我们找到`OrderController.java`类，然后声明Web方法。

```java
@RestController
@RequestMapping("/order")
@Tag(name = "OrderController", description = "订单模块Web接口")
public class OrderController {
    ……
    @RequestMapping("/recieveMessage")
    @Operation(summary = "接收代驾费消息通知")
    public void recieveMessage(HttpServletRequest request, HttpServletResponse response) throws Exception {
        request.setCharacterEncoding("utf-8");
        Reader reader = request.getReader();
        BufferedReader buffer = new BufferedReader(reader);
        String line = buffer.readLine();
        StringBuffer temp = new StringBuffer();
        while (line != null) {
            temp.append(line);
            line = buffer.readLine();
        }
        buffer.close();
        reader.close();
        String xml = temp.toString();
        if (WXPayUtil.isSignatureValid(xml, myWXPayConfig.getKey())) {
            Map<String, String> map = WXPayUtil.xmlToMap(xml);
            String resultCode = map.get("result_code");
            String returnCode = map.get("return_code");
            if ("SUCCESS".equals(resultCode) && "SUCCESS".equals(returnCode)) {
                response.setCharacterEncoding("utf-8");
                response.setContentType("application/xml");
                Writer writer = response.getWriter();
                BufferedWriter bufferedWriter = new BufferedWriter(writer);
                bufferedWriter.write("<xml><return_code><![CDATA[SUCCESS]]></return_code> <return_msg><![CDATA[OK]]></return_msg></xml>");
                bufferedWriter.close();
                writer.close();

                String uuid = map.get("out_trade_no");
                String payId = map.get("transaction_id");
                String driverOpenId = map.get("attach");
                String payTime = DateUtil.parse(map.get("time_end"), "yyyyMMddHHmmss").toString("yyyy-MM-dd HH:mm:ss");

                //TODO 修改订单状态、执行分账、发放系统奖励
            }
        } else {
            response.sendError(500, "数字签名异常");
        }
    }
}
```

乘客付款成功后，将结果改成已付款状态。       如果满足奖励条件，还要向司机发放奖励。                                                                                                                                                                                                                                                                                               

#### 6.司机奖励发放

先来了解一下司机`tb_wallet_income`表结构。

| **字段名**  | **类型** | **非空** | **备注**                  |
| ----------- | -------- | -------- | ------------------------- |
| id          | bigint   | True     | 主键                      |
| uuid        | varchar  | True     | uuid字符串                |
| driver_id   | bigint   | True     | 司机ID                    |
| amount      | decimal  | True     | 金额                      |
| type        | tinyint  | True     | 1充值，2奖励，3补贴       |
| prepay_id   | varchar  | False    | 预支付订单ID              |
| status      | tinyint  | True     | 1未支付，2已支付，3已到账 |
| remark      | varchar  | False    | 备注信息                  |
| create_time | datetime | True     | 创建时间                  |

**业务逻辑**

```java
@Service
public class OrderServiceImpl implements OrderService {
    ……
        
    @Override
    @Transactional
    @LcnTransaction
    public void handlePayment(String uuid, String payId, String driverOpenId, String payTime) {
        /* 
         * 更新订单状态之前，先查询订单的状态。
         * 因为乘客端付款成功之后，会主动发起Ajax请求，要求更新订单状态。
         * 所以后端接收到付款通知消息之后，不要着急修改订单状态，先看一下订单是否已经是7状态
         */
        HashMap map = orderDao.searchOrderIdAndStatus(uuid);
        int status = MapUtil.getInt(map, "status");
        if (status == 7) {
            return;
        }

        HashMap param = new HashMap() {{
            put("uuid", uuid);
            put("payId", payId);
            put("payTime", payTime);
        }};
        //更新订单记录的PayId、状态和付款时间
        int rows = orderDao.updateOrderPayIdAndStatus(param);
        if (rows != 1) {
            throw new HxdsException("更新支付订单ID失败");
        }
        
        //查询系统奖励
        map = orderDao.searchDriverIdAndIncentiveFee(uuid);
        String incentiveFee = MapUtil.getStr(map, "incentiveFee");
        long driverId = MapUtil.getLong(map, "driverId");
        //判断系统奖励费是否大于0
        if (new BigDecimal(incentiveFee).compareTo(new BigDecimal("0.00")) == 1) {
            TransferForm form = new TransferForm();
            form.setUuid(IdUtil.simpleUUID());
            form.setAmount(incentiveFee);
            form.setDriverId(driverId);
            form.setType((byte) 2);
            form.setRemark("系统奖励费");
            //给司机钱包转账奖励费
            drServiceApi.transfer(form);
        }
        
        //先判断是否有分账定时器   
        if (quartzUtil.checkExists(uuid, "代驾单分账任务组") || quartzUtil.checkExists(uuid, "查询代驾单分账任务组")) {
            //存在分账定时器就不需要再执行分账
            return;
        }
        //执行分账
        JobDetail jobDetail = JobBuilder.newJob(HandleProfitsharingJob.class).build();
        Map dataMap = jobDetail.getJobDataMap();
        dataMap.put("uuid", uuid);
        dataMap.put("driverOpenId", driverOpenId);
        dataMap.put("payId", payId);
        
        //2分钟之后执行分账定时器
        Date executeDate = new DateTime().offset(DateField.MINUTE, 2);
        quartzUtil.addJob(jobDetail, uuid, "代驾单分账任务组", executeDate);

        //更新订单状态为已完成状态（8）
        rows = orderDao.finishOrder(uuid);
        if (rows != 1) {
            throw new HxdsException("更新订单结束状态失败");
        }
    }
    }
}
```

#### 7.微信支付分账

在微信商户平台上面，在产品大全栏目中可以找到微信分账服务，开启这项服务即可。2. 关于分账比例

普通商户的分账比例默认上限是30%，也就是说商户自己留存70%，给员工或者其他人分账最高比例是30%，在普通业务中也还凑合，因为员工拿的提成很少有高于30%的。但是在代驾业务中，代驾司机的劳务费占了订单总额的大头，所以给司机30%的分账比例根本不够。

微信商户平台之所以给商户默认设置这么低的分账上限，主要是为了防止商户逃税或者洗钱。比如公司按照营业利润交税，但是公司老板想要少缴税，这需要通过某些手段减少企业盈利。于是把订单分账比例调高到99%，然后公司客户每笔货款都被分账到了老板指定的微信上面，揣到老板自己腰包里面，而公司的营业利润大幅降低，缴纳的税款也就变少了。微信平台为了避免有些公司通过分账来逃税，所以就给分账比例设置了30%的上线。不过也不是申请更高的分账比例，在设置分账比例上线的页面中，我们可以按照申请流程的指引，申请更高的分账比例。这就需要企业提交更多的证明材料，然后微信平台还要聘请专业的审计公司对审核的单位加以评估。即便你们公司申请下来高比例的分账，但是微信平台会把你们公司的分账记录和税务局的金税系统联网，随时监控你们公司是否伪造交易虚设分账，企图逃税。

微信支付官方提供了在线文档（https://pay.weixin.qq.com/wiki/doc/api/allocation.php?chapter=27_1&index=1），详细说明了如何创建分账请求。微信分账可以单次分账，也可以多次分账，我们只需要给司机分账，所以用单次分账即可。

| **参数名称** | **变量名称**   | **类型** | **例子**                                                     |
| ------------ | -------------- | -------- | ------------------------------------------------------------ |
| 商户号       | mch_id         | 字符串   | 1900000100                                                   |
| 小程序APPID  | appid          | 字符串   | wx8888888888888888                                           |
| 随机字符串   | nonce_str      | 字符串   | 5K8264ILTKCH16CQ2502SI8ZNMTM67VS                             |
| 数字签名     | sign           | 字符串   | C38F3AD6C380BEC2BFD727A4B684519FAD6                          |
| 微信账单ID   | transaction_id | 字符串   | 4208450740201411110007820472                                 |
| 商户订单号   | out_order_no   | 字符串   | P20150806125346                                              |
| 分账收款方   | receivers      | 字符串   | [ {"type": "PERSONAL_OPENID ", <br/>"account":"OPENID字符串", <br/>"amount":100, <br/>"description": "分到商户" }] |

提交分账请求之后，返回的响应里面也包含了很多信息，下面咱们来看一下。

| **参数名称** | **变量名称** | **例子**                               |
| ------------ | ------------ | -------------------------------------- |
| 通信状态码   | return_code  | SUCCESS / FAIL                         |
| 业务状态码   | result_code  | SUCCESS / FAIL                         |
| 错误代码     | err_code     | SYSTEMERROR                            |
| 错误代码描述 | err_code_des | 系统超时                               |
| 分账单状态   | status       | PROCESSING：处理中，FINISHED：处理完成 |
| 数字签名     | sign         | HMACSHA256算法生成的数字签名           |

假设用户付款成功，商户系统至少要2分钟之后才可以申请分账。很多同学第一次做分账，不知道这个细节，导致Web方法刚收到付款成功的通知消息，就立即发起分账，然后就收到订单暂时不可以分账的异常消息。既然我们要等待两分钟以上才可以申请分账，所以我们就应该用上Quartz定时器技术。因为执行分账前，我们要知道给司机的分账金额是多少，才能创建分账请求。这就需要我们编写SQL语句，查询分账记录的数据。另外分账成功之后，我们需要修改分账记录的状态，所以我们还需要编写UPDATE语句才行。

#### 8.定时任务进行分账

```java
@Slf4j
public class HandleProfitsharingJob extends QuartzJobBean {
    @Resource
    private OrderProfitsharingDao profitsharingDao;

    @Resource
    private MyWXPayConfig myWXPayConfig;

    @Override
    @Transactional
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        //获取传给定时器的业务数据
        Map map = context.getJobDetail().getJobDataMap();
        String uuid = MapUtil.getStr(map, "uuid");
        String driverOpenId = MapUtil.getStr(map, "driverOpenId");
        String payId = MapUtil.getStr(map, "payId");

        //查询分账记录ID、分账金额
        map = profitsharingDao.searchDriverIncome(uuid);
        if (map == null || map.size() == 0) {
            log.error("没有查询到分账记录");
            return;
        }
        String driverIncome = MapUtil.getStr(map, "driverIncome");
        long profitsharingId = MapUtil.getLong(map, "profitsharingId");

        try {
            WXPay wxPay = new WXPay(myWXPayConfig);

            //分账请求必要的参数
            HashMap param = new HashMap() {{
                put("appid", myWXPayConfig.getAppID());
                put("mch_id", myWXPayConfig.getMchID());
                put("nonce_str", WXPayUtil.generateNonceStr());
                put("out_order_no", uuid);
                put("transaction_id", payId);
            }};

            //分账收款人数组
            JSONArray receivers = new JSONArray();
            //分账收款人（司机）信息
            JSONObject json = new JSONObject();
            json.set("type", "PERSONAL_OPENID");
            json.set("account", driverOpenId);
            //分账金额从元转换成分
            int amount = Integer.parseInt(NumberUtil.mul(driverIncome, "100").setScale(0, RoundingMode.FLOOR).toString());
            json.set("amount", amount);
            //json.set("amount", 1); //设置分账金额为1分钱（测试阶段）
            json.set("description", "给司机的分账");
            receivers.add(json);

            //添加分账收款人JSON数组
            param.put("receivers", receivers.toString());

            //生成数字签名
            String sign = WXPayUtil.generateSignature(param, myWXPayConfig.getKey(), WXPayConstants.SignType.HMACSHA256);
            //添加数字签名
            param.put("sign", sign);

            String url = "/secapi/pay/profitsharing";
            //执行分账请求
            String response = wxPay.requestWithCert(url, param, 3000, 3000);
            log.debug(response);

            //验证响应的数字签名
            if (WXPayUtil.isSignatureValid(response, myWXPayConfig.getKey(), WXPayConstants.SignType.HMACSHA256)) {
                //从响应中提取数据
                Map<String, String> data = wxPay.processResponseXml(response, WXPayConstants.SignType.HMACSHA256);
                String returnCode = data.get("return_code");
                String resultCode = data.get("result_code");
                //验证通信状态码和业务状态码
                if ("SUCCESS".equals(resultCode) && "SUCCESS".equals(returnCode)) {
                    String status = data.get("status");
                    //判断分账成功
                    if ("FINISHED".equals(status)) {
                        //更新分账状态
                        int rows = profitsharingDao.updateProfitsharingStatus(profitsharingId);
                        if (rows != 1) {
                            log.error("更新分账状态失败", new HxdsException("更新分账状态失败"));
                        }
                    }
                    //判断正在分账中
                    else if ("PROCESSING".equals(status)) {
                        //TODO 创建查询分账定时器
                    }
                } else {
                    log.error("执行分账失败", new HxdsException("执行分账失败"));
                }
            } else {
                log.error("验证数字签名失败", new HxdsException("验证数字签名失败"));
            }

        } catch (Exception e) {
            log.error("执行分账失败", e);
        }
        
                if ("FINISHED".equals(status)) {
            ……
        }
        //判断正在分账中
        else if ("PROCESSING".equals(status)) {
            //如果状态是分账中，等待几分钟再查询分账结果
            JobDetail jobDetail = JobBuilder.newJob(SearchProfitsharingJob.class).build();
            Map dataMap = jobDetail.getJobDataMap();
            dataMap.put("uuid", uuid);
            dataMap.put("profitsharingId", profitsharingId);
            dataMap.put("payId", payId);

            Date executeDate = new DateTime().offset(DateField.MINUTE, 20);
            quartzUtil.addJob(jobDetail, uuid, "查询代驾单分账任务组", executeDate);
        }
    }
}
```

#### 9.定时器工具类

```java
@Component
@Slf4j
public class QuartzUtil {

    @Resource
    private Scheduler scheduler;

    /**
     * 添加定时器
     *
     * @param jobDetail    定时器任务对象
     * @param jobName      任务名字
     * @param jobGroupName 任务组名字
     * @param start        开始日期时间
     */
    public void addJob(JobDetail jobDetail, String jobName, String jobGroupName, Date start) {
        try {
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity(jobName, jobGroupName)
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                            .withMisfireHandlingInstructionFireNow())
                    .startAt(start).build();
            scheduler.scheduleJob(jobDetail, trigger);
            log.debug("成功添加" + jobName + "定时器");
        } catch (SchedulerException e) {
            log.error("定时器添加失败", e);
        }
    }

    /**
     * 查询是否存在定时器
     *
     * @param jobName      任务名字
     * @param jobGroupName 任务组名字
     * @return
     */
    public boolean checkExists(String jobName, String jobGroupName) {
        TriggerKey triggerKey = new TriggerKey(jobName, jobGroupName);
        try {
            boolean bool = scheduler.checkExists(triggerKey);
            return bool;
        } catch (Exception e) {
            log.error("定时器查询失败", e);
            return false;
        }
    }

    /**
     * 删除定时器
     *
     * @param jobName      任务名字
     * @param jobGroupName 任务组名字
     */
    public void deleteJob(String jobName, String jobGroupName) {
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroupName);
        try {
            scheduler.resumeTrigger(triggerKey);
            scheduler.unscheduleJob(triggerKey);
            scheduler.deleteJob(JobKey.jobKey(jobName, jobGroupName));
            log.debug("成功删除" + jobName + "定时器");
        } catch (SchedulerException e) {
            log.error("定时器删除失败", e);
        }

    }
}
```

#### 10.定时器查询分账结果

微信服务器消息队列中积压的分账任务很多，所以我创建分账请求之后，并没有立即成功分账，响应中status状态是分账中，所以当前的定时器并没有把分账记录修改成2状态。遇到这种情况，我们应该创建一个20分钟之后运行的定时器，用来检查分账的结果。如果分账成功，我们就把数据库中分账记录的status字段改成2状态。

微信官方提供了查询分账结果的API文档（https://pay.weixin.qq.com/wiki/doc/api/allocation.php?chapter=27_2&index=3），我们写代码之前先来看一下这部分的资料。发起查询请求的时候，需要我们提交一些参数，如下：

| **参数**   | **变量名**     | **类型** | **示例**                         |
| ---------- | -------------- | -------- | -------------------------------- |
| 商户号     | mch_id         | 字符串   | 1900000100                       |
| 微信账单号 | transaction_id | 字符串   | 4208450740201411148              |
| 商户订单号 | out_order_no   | 字符串   | P20150806125312366               |
| 随机字符串 | nonce_str      | 字符串   | 5K8264ILTKCH16CQ2502SI8Z         |
| 签名       | sign           | 字符串   | C380BEC2BFD727A4B6845133519F3AD6 |

微信平台返回的响应里面，关键的参数我列在下面的表格中了。

| **参数**     | **变量名**   | **类型** | **示例**                               |
| ------------ | ------------ | -------- | -------------------------------------- |
| 通信状态码   | return_code  | 字符串   | SUCCESS  / FAIL                        |
| 业务状态码   | result_code  | 字符串   | SUCCESS  / FAIL                        |
| 错误代码     | err_code     | 字符串   | SYSTEMERROR                            |
| 错误代码描述 | err_code_des | 字符串   | 系统超时                               |
| 数字签名     | sign         | 字符串   | C380BEC2BFD727A4B6845133519F3AD6       |
| 分账单状态   | status       | 字符串   | PROCESSING：处理中，FINISHED：处理完成 |

#### 11.分账查询任务类

```java
@Slf4j
public class SearchProfitsharingJob extends QuartzJobBean {
    @Resource
    private OrderProfitsharingDao profitsharingDao;

    @Resource
    private MyWXPayConfig myWXPayConfig;

    @Override
    @Transactional
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        Map map = context.getJobDetail().getJobDataMap();
        String uuid = MapUtil.getStr(map, "uuid");
        long profitsharingId = MapUtil.getLong(map, "profitsharingId");
        String payId = MapUtil.getStr(map, "payId");

        try {
            WXPay wxPay = new WXPay(myWXPayConfig);
            String url = "/pay/profitsharingquery";

            HashMap param = new HashMap() {{
                put("mch_id", myWXPayConfig.getMchID());
                put("transaction_id", payId);
                put("out_order_no", uuid);
                put("nonce_str", WXPayUtil.generateNonceStr());
            }};
            
            //生成数字签名
            String sign = WXPayUtil.generateSignature(param, myWXPayConfig.getKey(), WXPayConstants.SignType.HMACSHA256);
            param.put("sign", sign);
            
            //查询分账结果
            String response = wxPay.requestWithCert(url, param, 3000, 3000);
            log.debug(response);
            
            //验证响应的数字签名
            if (WXPayUtil.isSignatureValid(response, myWXPayConfig.getKey(), WXPayConstants.SignType.HMACSHA256)) {
                Map<String, String> data = wxPay.processResponseXml(response,WXPayConstants.SignType.HMACSHA256);
                String returnCode = data.get("return_code");
                String resultCode = data.get("result_code");
                if ("SUCCESS".equals(resultCode) && "SUCCESS".equals(returnCode)) {
                    String status = data.get("status");
                    if ("FINISHED".equals(status)) {
                        //把分账记录更新成2状态
                        int rows = profitsharingDao.updateProfitsharingStatus(profitsharingId);
                        if (rows != 1) {
                            log.error("更新分账状态失败", new HxdsException("更新分账状态失败"));
                        }
                    }
                } else {
                    log.error("查询分账失败", new HxdsException("查询分账失败"));
                }
            } else {
                log.error("验证数字签名失败", new HxdsException("验证数字签名失败"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

### 四、系统主动查询支付结果

接下来要处理一些意外情况。比如说微信平台的消息队列出现故障，乘客付款成功之后，微信平台没有发送付款结果通知消息给Web方法。或者说，咱们本地的网络出现了故障，没有收到付款结果通知消息。这会导致，乘客付款成功，但是代驾订单依旧是未付款状态。

为了避免收不到付款结果通知消息，而不知道付款结果，我们可以让乘客小程序在付款成功之后，自动发出Ajax请求，然后被网关子系统路由给订单子系统。订单子系统向微信支付平台发起请求，查询付款结果。如果付款成功，就更新订单状态，并且执行分账。微信官方提供了查询支付结果的API文档（https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_2），我们先简单了解一下。

创建查询请求的时候，要规定提交的数据，如下面的表格。

| **参数**   | **变量名**   | **类型** | ** 示例**                        |
| ---------- | ------------ | -------- | -------------------------------- |
| 小程序ID   | appid        | 字符串   | wxd678efh567hg6787               |
| 商户号     | mch_id       | 字符串   | 1230000109                       |
| 商户订单号 | out_trade_no | 字符串   | 20150806125346                   |
| 随机字符串 | nonce_str    | 字符串   | C380BEC2BFD727A                  |
| 数字签名   | sign         | 字符串   | 5K8264ILTKCH16CQ2502SI8ZNMTM67VS |

返回的响应中，关键的参数我提取出来放在下面表格里面了。

| 参数         | 变量名         | 示例                                                         |
| ------------ | -------------- | ------------------------------------------------------------ |
| 通信状态码   | return_code    | SUCCESS/FAIL                                                 |
| 业务状态码   | result_code    | SUCCESS/FAIL                                                 |
| 错误代码     | err_code       | SYSTEMERROR                                                  |
| 错误代码描述 | err_code       | 系统错误                                                     |
| 付款金额     | total_fee      | 100                                                          |
| 交易状态     | trade_state    | SUCCESS–支付成功 REFUND–转入退款 NOTPAY–未支付 CLOSED–已关闭 ACCEPT–已接收，等待扣款 |
| 自定义参数   | attach         |                                                              |
| 微信账单号   | transaction_id | 1009660380201506130                                          |
| 支付完成时间 | time_end       | 20141030133525                                               |

#### 1.支付结果查询与订单更新

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Resource
    private MyWXPayConfig myWXPayConfig;
    
    ……
    @Override
    @Transactional
    @LcnTransaction
    public String updateOrderAboutPayment(Map param) {
        long orderId = MapUtil.getLong(param, "orderId");
        /*
        * 查询订单状态。
        * 因为有可能Web方法先收到了付款结果通知消息，把订单状态改成了7、8状态，
        * 所以我们要先查询订单状态。
        */
        HashMap map = orderDao.searchUuidAndStatus(orderId);
        String uuid = MapUtil.getStr(map, "uuid");
        int status = MapUtil.getInt(map, "status");
        //如果订单状态已经是已付款，就退出当前方法
        if (status == 7 || status == 8) {
            return "付款成功";
        }
        
        //查询支付结果的参数
        map.clear();
        map.put("appid", myWXPayConfig.getAppID());
        map.put("mch_id", myWXPayConfig.getMchID());
        map.put("out_trade_no", uuid);
        map.put("nonce_str", WXPayUtil.generateNonceStr());
        try {
            //生成数字签名
            String sign = WXPayUtil.generateSignature(map, myWXPayConfig.getKey());
            map.put("sign", sign);
            
            WXPay wxPay = new WXPay(wxPayConfig);
            //查询支付结果
            Map<String, String> result = wxPay.orderQuery(map);
            
            String returnCode = result.get("return_code");
            String resultCode = result.get("result_code");
            if ("SUCCESS".equals(returnCode) && "SUCCESS".equals(resultCode)) {
                String tradeState = result.get("trade_state");
                if ("SUCCESS".equals(tradeState)) {
                    String driverOpenId = result.get("attach");
                    String payId = result.get("transaction_id");
                    String payTime = new DateTime(result.get("time_end"), "yyyyMMddHHmmss").toString("yyyy-MM-dd HH:mm:ss");
                    //更新订单相关付款信息和状态
                    param.put("payId", payId);
                    param.put("payTime", payTime);
                    
                    //把订单更新成7状态
                    int rows = orderDao.updateOrderAboutPayment(param);
                    if (rows != 1) {
                        throw new HxdsException("更新订单相关付款信息失败");
                    }

                    //查询系统奖励
                    map = orderDao.searchDriverIdAndIncentiveFee(uuid);
                    String incentiveFee = MapUtil.getStr(map, "incentiveFee");
                    long driverId = MapUtil.getLong(map, "driverId");
                    //判断系统奖励费是否大于0
                    if (new BigDecimal(incentiveFee).compareTo(new BigDecimal("0.00")) == 1) {
                        TransferForm form = new TransferForm();
                        form.setUuid(IdUtil.simpleUUID());
                        form.setAmount(incentiveFee);
                        form.setDriverId(driverId);
                        form.setType((byte) 2);
                        form.setRemark("系统奖励费");
                        //给司机钱包转账奖励费
                        drServiceApi.transfer(form);
                    }

                    //先判断是否有分账定时器
                    if (quartzUtil.checkExists(uuid, "代驾单分账任务组") || quartzUtil.checkExists(uuid, "查询代驾单分账任务组")) {
                        //存在分账定时器就不需要再执行分账
                        return "付款成功";
                    }
                    //执行分账
                    JobDetail jobDetail = JobBuilder.newJob(HandleProfitsharingJob.class).build();
                    Map dataMap = jobDetail.getJobDataMap();
                    dataMap.put("uuid", uuid);
                    dataMap.put("driverOpenId", driverOpenId);
                    dataMap.put("payId", payId);

                    Date executeDate = new DateTime().offset(DateField.MINUTE, 2);
                    quartzUtil.addJob(jobDetail, uuid, "代驾单分账任务组", executeDate);
                    rows = orderDao.finishOrder(uuid);
                    if(rows!=1){
                        throw new HxdsException("更新订单结束状态失败");
                    }
                    return "付款成功";
                } else {
                    return "付款异常";
                }
            } else {
                return "付款异常";
            }

        } catch (Exception e) {
            e.printStackTrace();
            throw new HxdsException("更新订单相关付款信息失败");
        }
    }
}
```

乘客支付订单后，可以主动发送Ajax请求，查询订单支付情况。

司机端程序上面设置轮询定时器，不停的查询代驾订单是否付款成功。









