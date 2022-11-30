# CentOS系统环境搭建

## 一、环境说明

大数据软件的搭建，全部做成分布式集群模式，使用虚拟机CentOS(最小安装)搭建，并且每个虚拟机直接使用root用户，不设置其他用户权限。

CentOS版本为7.5，共5台，每台的网络地址和内存网盘使用情况如下表配置：

| 名称      | 网关          | ip地址          | 内存 | 硬盘 | 职责 |
| --------- | ------------- | --------------- | ---- | ---- | ---- |
| hadoop001 | 192.168.114.2 | 192.168.114.101 | 4G   | 50G  | 主机 |
| hadoop002 | 192.168.114.2 | 192.168.114.102 | 4G   | 50G  | 从机 |
| hadoop003 | 192.168.114.2 | 192.168.114.103 | 4G   | 50G  | 从机 |

## 二、虚拟机网络配置

成功安装虚拟机并且打开后，首先需要配置网络，这里详细介绍以下配置网络的过程。这里只举一个例子，其他两个机器，操作与下面命令一致。

1. 修改机器名

```bash
# 查看机器名
cat /etc/hostname
# 修改每台机器名称为设定的名称，使用下面命令，删除文件内容，填写设定的机器名
vim /etc/hostname
```

2. 修改地址与机器名映射

```bash
vim /etc/hosts
//添加以下内容
192.168.114.101	hadoop001
192.168.114.102	hadoop002	
192.168.114.103	hadoop003	
```

3. 设置网络信息

```bash
vi  /etc/sysconfig/network-scripts/ifcfg-ens33 
//添加以下内容
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEEROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
#删除UUID,防止克隆时出现两台机器的唯一标识是一样的
DEVICE=ens33
ONBOOT=yes
#ip
IPADDR=192.168.114.101
#网关
GATEWAY=192.168.114.2
#子网掩码
NETMASK=255.255.255.0
#使用主的DNS
DNS1=114.114.114.114
#备用的DNS
DNS2=8.8.8.8

// 保存后，重启网卡
systemctl restart network.service
```

4. 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

5. 安装vim并设置

```bash
yum install -y vim*

// 配置vim
vim /etc/vimrc
set nu          // 设置显示行号
set showmode    // 设置在命令行界面最下面显示当前模式等
set ruler       // 在右下角显示光标所在的行数等信息
set autoindent  // 设置每次单击Enter键后，光标移动到下一行时与上一行的起始字符对齐
set tabstop     // tab4个空格
set cursorline  // 当前行高亮	
syntax on       // 即设置语法检测，当编辑C或者Shell脚本时，关键字会用特殊颜色显示
```

6. 安装openssh和git

```bash
yum install -y openssh-server
yum install -y git
// 创建公钥密钥
ssh-keygen -t rsa
// 一路回车之后，会创建一个.ssh文件夹，里面有三个文件，分别是公钥、密钥等
// 设置免密登录
cd .ssh
cat id_rsa.pub >> authorized_keys
// 在hadoop001上，操作下面
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop001
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop002
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop003
```

7. 安装JDK

```bash
# 解压jdk，并将其配置到环境变量中
# Java env
export JAVA_HOME=/root/app/jdk1.8.0_211
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin:$PATH
```

8.安装hadoop集群

```bash
# 解压hadoop安装包，并将其配置到环境变量中
# hadoop env
export HADOOP_HOME=/root/app/hadoop
export PATH=$HADOOP_HOME/bin:$PATH
export PATH=$HADOOP_HOME/sbin:$PATH

# vi core-site.xml添加如下配置
<configuration> 
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop001:8020</value>
    </property>
</configuration>

# vi hdfs-site.xml
<configuration> 
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/root/app/hadoop/data/dfs/name</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/root/app/hadoop/data/dfs/data</value>
    </property>
</configuration>


# vi mapred-site.xml
<configuration>
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
</configuration>

# vi yarn-site.xml
<configuration>
    <property>
    	<name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
	</property>

    <property>
    	<name>yarn.nodemanager.local-dirs</name>
    	<value>/root/app/hadoop/data/nm-local-dir</value>
    </property>

</configuration>
   
# vi slaves
hadoop001
hadoop002
hadoop003
```



## 三、软件环境变量配置

```bash
# node js
export NODE_HOME=/home/tony/app/node-v14.16.1-linux-x64/bin
export PATH=$NODE_HOME:$PATH

# maven env
export MAVEN_HOME=/home/tony/app/apache-maven-3.6.3-bin/apache-maven-3.6.3
export PATH=$MAVEN_HOME/bin:$PATH

# scala env
export SCALA_HOME=/home/tony/app/scala-2.11.12
export PATH=$SCALA_HOME/bin:$PATH

# hive env
export HIVE_HOME=/home/tony/app/hive-1.1.0-cdh5.15.1
export PATH=$HIVE_HOME/bin:$PATH

# hbase env
export HBASE_HOME=/home/willhope/app/hbase-1.2.0-cdh5.15.1
export PATH=$HBASE_HOME/bin:$PATH

# spark environment
export SPARK_HOME=/home/tony/app/spark-2.4.4-bin-2.6.0-cdh5.15.1
export PATH=$SPARK_HOME/bin:$PATH


# sqoop environment
export SQOOP_HOME=/home/tony/app/sqoop-1.4.7.bin__hadoop-2.6.0
export PATH=$SQOOP_HOME/bin:$PATH

# zookeeper environment
export ZOOKEEPER_HOME=/home/tony/app/zookeeper-3.4.5-cdh5.15.1
export PATH=$ZOOKEEPER_HOME/bin:$PATH

# kafka environment
export KAFKA_HOME=/home/tony/app/kafka_2.11-0.9.0.0
export PATH=$KAFKA_HOME/bin:$PATH

# flume environment
export FLUME_HOME=/home/tony/app/flume
export PATH=$FLUME_HOME/bin:$PATH

# flink env
export FLINK_HOME=/home/tony/app/flink-1.7.0
export PATH=$FLINK_HOME/bin:$PATH

# kibana env
export LIBANA_HOME=/home/tony/kibana-6.8.12-linux-x86_64
export PATH=$KIBANA_HOME/bin:$PATH

# elastic
export ELASTIC_HOME=/home/tony/app/elasticsearch-6.8.12
export PATH=$ELASTIC_HOME/bin:$PATH

# go env
export GO_HOME=/home/tony/app/go
export PATH=$GO_HOME/bin:$PATH
```

## 三、Hive部署

在 http://archive.cloudera.com/cdh5/cdh/5/找到hive-1.1.0-cdh5.15.1.tar.gz 这个包，将其下载下来，解压到app目录下。将hive添加到系统环境中，方便使用，但是这里最好重新启动一下机器。在使用hive前，必须先将hadoop平台的所有东西启动起来。

进入hive目录进行配置，修改配置conf目录下的hive-env.sh、hive-site.xml，再拷贝MySQL驱动包到$HIVE_HOME/lib，但前提是要准备安装一个MySQL数据库，sudo apt-get install去安装一个MySQL数据库 https://www.cnblogs.com/julyme/p/5969626.html

```xml

    <!--本部分写在hive-site.xml，注意更换你的mysql配置-->
    <?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

    <configuration>
    <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://127.0.0.1:3306/hadoop_hive?createDatabaseIfNotExist=true</value>
    </property>

    <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>root</value>
    </property>

    <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>123456</value>
    </property>
    </configuration>

```

这部分写在hive-env.sh中

```bash

HADOOP_HOME=/home/willhope/app/hadoop-2.6.0-cdh5.15.1（注意更换你的地址）

```

## 四、Hive配置遇到的坑

遇到一个坑，以前学的时候用的是别人提供的镜像没有这种问题，现在学的时候用的是deepin，在配置好各种内容，启动hive，在hive中查询时，总是会提出Error: Syntax error: Encountered “” at line 1, column 64。

搜到网上各种教程，说是hive默认的是derby，要进行初始化。然而跟着网上的教程做，发现依然无法解决上面的错。

最终，解决方法是，删除原来的hive，然后重新配置好hive，在启动hive之前，进行初始化，进入到bin目录，执行 ./schematool -dbType mysql -initSchema -verbose，schemaTool completed则表明成功，并且会完成在mysql中数据库的创建（也就是hive-site.xml中配置的数据库），此时数据库中的表都是空的，没有内容。然后在bin下执行hive，执行create database test_db后，表中就有内容了，以及其他查询操作，即可成功。（在hive执行sql语句时，会发现一个ssl警告，可以忽略，也可以在hive-site.xml，配置数据库名字那一行createDatabaseIfNotexist=true后面添加上;ssl=true）

## 五、Sqoop安装

#### 1.Sqoop 简介

Sqoop 是一个常用的数据迁移工具，主要用于在不同存储系统之间实现数据的导入与导出：

+ 导入数据：从 MySQL，Oracle 等关系型数据库中导入数据到 HDFS、Hive、HBase 等分布式文件存储系统中；

+ 导出数据：从 分布式文件系统中导出数据到关系数据库中。

其原理是将执行命令转化成 MapReduce 作业来实现数据的迁移，如下图：

 <img  src="../picture/sqoop-tool.png"/>

#### 2.安装

版本选择：目前 Sqoop 有 Sqoop 1 和 Sqoop 2 两个版本，但是截至到目前，官方并不推荐使用 Sqoop 2，因为其与 Sqoop 1 并不兼容，且功能还没有完善，所以这里优先推荐使用 Sqoop 1。

<div align="center"> <img  src="/pictures/sqoop-version-selected.png"/> </div>



##### 2.1 下载并解压

下载所需版本的 Sqoop ，这里我下载的是 `CDH` 版本的 Sqoop 。下载地址为：http://archive.cloudera.com/cdh5/cdh/5/

```shell
# 下载后进行解压
tar -zxvf  sqoop-1.4.6-cdh5.15.1.tar.gz
```

##### 2.2 配置环境变量

```shell
# vim /etc/profile
```

添加环境变量：

```shell
export SQOOP_HOME=/usr/app/sqoop-1.4.6-cdh5.15.1
export PATH=$SQOOP_HOME/bin:$PATH
```

使得配置的环境变量立即生效：

```shell
# source /etc/profile
```

##### 2.3 修改配置

进入安装目录下的 `conf/` 目录，拷贝 Sqoop 的环境配置模板 `sqoop-env.sh.template`

```shell
# cp sqoop-env-template.sh sqoop-env.sh
```

修改 `sqoop-env.sh`，内容如下 (以下配置中 `HADOOP_COMMON_HOME` 和 `HADOOP_MAPRED_HOME` 是必选的，其他的是可选的)：

```shell
# Set Hadoop-specific environment variables here.
#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/home/willhope/app/hadoop-2.6.0-cdh5.15.1

#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/home/willhope/app/hadoop-2.6.0-cdh5.15.1

#set the path to where bin/hbase is available
export HBASE_HOME=/home/willhope/app/hbase-1.2.0-cdh5.15.1 

#Set the path to where bin/hive is available
export HIVE_HOME=/home/willhope/app/hive-1.1.0-cdh5.15.1

#Set the path for where zookeper config dir is
export ZOOCFGDIR=/home/willhope/app/zookeeper-3.4.13/conf

```

##### 2.4 拷贝数据库驱动

将 MySQL 驱动包拷贝到 Sqoop 安装目录的 `lib` 目录下, 驱动包的下载地址为 https://dev.mysql.com/downloads/connector/j/  。

<div align="center"> <img  src="../pictures/sqoop-mysql-jar.png"/> </div>



##### 2.5 验证

由于已经将 sqoop 的 `bin` 目录配置到环境变量，直接使用以下命令验证是否配置成功：

```shell
# sqoop version
```

出现对应的版本信息则代表配置成功：

<div align="center"> <img  src="/pictures/sqoop-version.png"/> </div>

这里出现的两个 `Warning` 警告是因为我们本身就没有用到 `HCatalog` 和 `Accumulo`，忽略即可。Sqoop 在启动时会去检查环境变量中是否有配置这些软件，如果想去除这些警告，可以修改 `bin/configure-sqoop`，注释掉不必要的检查。

```shell
# Check: If we can't find our dependencies, give up here.
if [ ! -d "${HADOOP_COMMON_HOME}" ]; then
  echo "Error: $HADOOP_COMMON_HOME does not exist!"
  echo 'Please set $HADOOP_COMMON_HOME to the root of your Hadoop installation.'
  exit 1
fi
if [ ! -d "${HADOOP_MAPRED_HOME}" ]; then
  echo "Error: $HADOOP_MAPRED_HOME does not exist!"
  echo 'Please set $HADOOP_MAPRED_HOME to the root of your Hadoop MapReduce installation.'
  exit 1
fi

## Moved to be a runtime check in sqoop.
if [ ! -d "${HBASE_HOME}" ]; then
  echo "Warning: $HBASE_HOME does not exist! HBase imports will fail."
  echo 'Please set $HBASE_HOME to the root of your HBase installation.'
fi

## Moved to be a runtime check in sqoop.
if [ ! -d "${HCAT_HOME}" ]; then
  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
fi

if [ ! -d "${ACCUMULO_HOME}" ]; then
  echo "Warning: $ACCUMULO_HOME does not exist! Accumulo imports will fail."
  echo 'Please set $ACCUMULO_HOME to the root of your Accumulo installation.'
fi
if [ ! -d "${ZOOKEEPER_HOME}" ]; then
  echo "Warning: $ZOOKEEPER_HOME does not exist! Accumulo imports will fail."
  echo 'Please set $ZOOKEEPER_HOME to the root of your Zookeeper installation.'
fi
```

##### 2.6 错误解决方法

1. 执行sqoop时候，会报错rror: Could not find or load main class org.apache.sqoop.Sqoop，这是由于没有将sqoop-1.4.7.jar包添加到sqoop中，因此，我们需要先去下载这个jar包，然后将其放到sqoop根目录下，然后在bin目录下的sqoop脚本中将最后一句话更改。
```bash
exec ${HADOOP_COMMON_HOME}/bin/hadoop jar $SQOOP_HOME/sqoop-1.4.7.jar org.apache.sqoop.Sqoop "$@"
```

2. lang3错误，我们需要自己下载lang3的jar包，将其放入lib目录下。

下载地址：http://commons.apache.org/proper/commons-lang/download_lang.cgi

下载后，要记得解压缩，然后将jar包放入lib下

3. java.lang.ClassNotFoundException: org.apache.hadoop.hive.conf.HiveConf

```bash
cp hive-common-1.1.0-cdh5.15.1.jar ~/app/sqoop-1.4.7/lib/

```
4. Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.hive.shims.ShimLo
```bash
cp $HIVE_HOME/lib/hive-shims-*.jar ~/app/sqoop-1.4.7/lib/
```

## 六、Spark安装

### 1.Spark源码的编译

在spark.apache.org中下载spark，这里我们选择2.4.4的source code版本。下载后解压到software文件夹中。[Spark学习的官网](http://spark.apache.org/docs/latest/)。

注意：如果使用maven编译的话，这里有一个非常大的坑，官网要求基于 maven 的构建是 Apache Spark 的引用构建。 使用 Maven 构建 Spark 需要 Maven 3.5.4和 java8。 注意，从 Spark 2.2.0开始，Java 7的支持就被移除了。因此，一定要下载好对应的maven版本，否则会出现编译错误的情况。此外，因为本项目都是在CDH5.15.1平台上，因此在spark源码下的pom.xml文件中，要加上下面。但其实也可以使用Spark自带的maven进行编译，这样会省一些事情，本项目用的是自带的进行编译。

````xml
  <repository>
        <id>cloudera</id>
        <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
  </repository>
````

在spark源码目录下的dev目录中更改 make-distribution.sh

```bash
export MAVEN_OPTS="${MAVEN_OPTS:--Xmx2g -XX:ReservedCodeCacheSize=512m -XX:ReservedCodeCacheSize=512m}"
```

此时看一下自己的scala版本，在终端中输入scala -version进行查看，spark2.4版本只支持2.11和2.12版本的scala。然后在spark目录下执行./dev/change-scala-version.sh 2.11

在Spark源码目录下，执行下面的语句，这个过程会非常缓慢。估计得30分钟到1个小时，刚开始编译时，可能会发现终端一直卡着不动，这是正在检索所需要的环境，要耐心等待。

```m
./build/mvn -Pyarn -Phadoop-2.6 -Phive -Phive-thriftserver -Dhadoop.version=2.6.0-cdh5.15.1 -DskipTests clean package

推荐使用这个
./dev/make-distribution.sh --name 2.6.0-cdh5.15.1 --tgz  -Pyarn -Phadoop-2.6 -Phive -Phive-thriftserver -Dhadoop.version=2.6.0-cdh5.15.1
执行完后，会出现一个spark-2.4.4-bin-2.6.0-cdh5.15.1.tgz包。
```

### 2.Spark环境的搭建

将编译好的spark-2.4.4-bin-2.6.0-cdh5.15.1.tgz包解压到app中。进入到解压的目录下，我们在spark的bin目录下，看到很多.cmd的文件，这些文件是在windows上运行的，因此我们可以删除，使用rm -rf *.cmd

将spark配入到系统环境变量中，使用pwd查看路径，然后在终端中打开/etc/profile，添加如下的代码：

```xml
  export SPARK_HOME=/home/willhope/app/spark-2.4.4-bin-2.6.0-cdh5.15.1
  export PATH=$SPARK_HOME/bin:$PATH

  然后保存后，执行source etc/profile
```

修改sbin目录下的spark-config.sh,添加jdk的环境变量。

在spark目录下的bin下，执行spark-shell --master local[2]

可以在 http://192.168.0.100:4040 监控spark。**此地址每个电脑都不一样。在执行上面的指令后，我们会看到Spark context Web UI available at http://192.168.1.4:4041 这句话**，这个地址是多少，我们就使用多少。


在conf目录下，设置 cp spark-env.sh.template spark-env.sh，然后配置下面的语句

```bash
SPARK_MASTER_HOST=willhope-pc
SPARK_WORKER_CORES=2
SPARK_WORKER_MEMORY=2g
SPARK_WORKER_INSTANCES=1  # 这里以后可以任意设置
```

Spark Standalone模式的架构和Hadoop HDFS/YARN很类似的 1 master + n worker

启动spark，进入sbin目录，然后输入./start-all.sh，进入 http://192.168.0.100:8080/ (**前面的地址每个电脑都不一样，但是端口号通常是一样的**)可以查看信息，然后在bin目录下，执行spark-shell --master spark://willhope-PC:7077  (后面这个spark://willhope-PC:7077在你的 http://192.168.0.100:8080/ 页面的顶部位置可见)，启动时间有些长。


### 3.使用Spark完成wordcount统计

在bin目录下，执行spark-shell --master spark://willhope-PC:7077 ，会出现一个spark图像；也可以使用spark-shell --master local[2],推荐使用后者，这样可以使机器负载低一些。

```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/

Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_211)
Type in expressions to have them evaluated.
Type :help for more information.

scala>

```

然后进行scala输入，可以将下面三行代码直接粘贴进去。注意，自己定义一个文件，用来操作wordcount统计

```s
# 仅仅只需要三行，就可以完成之前java写MR那些代码，但其实我们也可以使用Java8提供的函数式编程来简化代码

//获取文件
val file = spark.sparkContext.textFile("file:///home/willhope/data/hello.txt")
val wordCounts = file.flatMap(line => line.split("\t")).map((word => (word, 1))).reduceByKey(_ + _)
wordCounts.collect

运行结果：

scala> val file = spark.sparkContext.textFile("file:///home/willhope/data/hello.txt")
file: org.apache.spark.rdd.RDD[String] = file:///home/willhope/data/hello.txt MapPartitionsRDD[1] at textFile at <console>:23

scala> val wordCounts = file.flatMap(line => line.split("\t")).map((word => (word, 1))).reduceByKey(_ + _)
wordCounts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[4] at reduceByKey at <console>:25

scala> wordCounts.collect
res0: Array[(String, Int)] = Array((word,3), (hello,5), (world,3))  

```

## 七、Flink单机部署

### 1.前置条件：

JDK、Maven3、Flink源码进行编译，不是使用直接下载二进制包

下载flink源码包：wget [https://github.com/apache/flink/archive/release-1.7.0.tar.gz](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fapache%2Fflink%2Farchive%2Frelease-1.7.0.tar.gz)

### 2.编译

将下载的源码包进行解压，进入到该解压包，执行下面的语句进行编译：

mvn clean install -DskipTests -Pvendor-repos -Dfast -Dhadoop.version=2.6.0-cdh5.15.1

第一次编译是需要花费很长时间的，因为需要去中央仓库下载flink源码中所有的依赖包

如果遇到某个包下载不下来而出现错误，可以根据错误提示手动下载包，并进行安装，之后重新编译。此处举一个例子，我在编译时候，这个maprfs-5.2.1-mapr.jar包总数出现问题，那就去[https://mvnrepository.com/](https://gitee.com/link?target=https%3A%2F%2Fmvnrepository.com%2F) 找这个包，下载后，进行手动安装。

mvn install:install-file -DgroupId=com.mapr.hadoop  -DartifactId=maprfs -Dversion=5.2.1-mapr -Dpackaging=jar   -Dfile=/home/willhope/Downloads/maprfs-5.2.1-mapr.jar

mvn install:install-file -DgroupId=org.apache.flink  -DartifactId=flink-mapr-fs -Dversion=1.7.0 -Dpackaging=jar  -Dfile=/home/willhope/Downloads/flink-mapr-fs-1.7.0.jar

安装完毕后，继续编译mvn clean install -DskipTests -Pvendor-repos -Dfast -Dhadoop.version=2.6.0-cdh5.15.1 -rf :flink-mapr-fs

测试：使用flink执行wordcount

```
./bin/flink run examples/streaming/SocketWindowWordCount.jar --port 9000
```

将flink添加到环境中并source，$FLINK_HOME

## 八、Elastic的部署

在官网下载源文件 [https://www.elastic.co/cn/downloads/elasticsearch](https://gitee.com/link?target=https%3A%2F%2Fwww.elastic.co%2Fcn%2Fdownloads%2Felasticsearch)

解压源文件，在bin目录下执行./elasticsearch，然后在浏览器中输入，[http://localhost:9200/，查看是否出现elasticsearch相关信息。](https://gitee.com/link?target=http%3A%2F%2Flocalhost%3A9200%2F%EF%BC%8C%E6%9F%A5%E7%9C%8B%E6%98%AF%E5%90%A6%E5%87%BA%E7%8E%B0elasticsearch%E7%9B%B8%E5%85%B3%E4%BF%A1%E6%81%AF%E3%80%82)

如果想要后台运行，则在bin目录下，执行 ./elasticsearch -d

## 九、Kibana部署

与Elastic一样在官网下载，解压源文件，在bin目录下，执行 ./kibana ，然后在浏览器中输入 [http://localhost:5601/app/home#](https://gitee.com/link?target=http%3A%2F%2Flocalhost%3A5601%2Fapp%2Fhome%23) 即可访问kibana

## 十、Flume安装

1. 在终端下载： wget：[http://archive.cloudera.com/cdh5/cdh/5/flume-ng-1.6.0-cdh5.15.1.tar.gz](https://gitee.com/link?target=http%3A%2F%2Farchive.cloudera.com%2Fcdh5%2Fcdh%2F5%2Fflume-ng-1.6.0-cdh5.15.1.tar.gz)

[http://archive.cloudera.com/cdh5/cdh/5/zookeeper-3.4.5-cdh5.15.1.tar.gz](https://gitee.com/link?target=http%3A%2F%2Farchive.cloudera.com%2Fcdh5%2Fcdh%2F5%2Fzookeeper-3.4.5-cdh5.15.1.tar.gz)

1. 解压缩： tar -zxvf flume-ng-1.6.0-cdh5.15.1.tar.gz /home/willhope/app
2. 添加环境变量： sudo vi /etc/profile

```
# flume environment

export FLUME_HOME=/home/willhope/app/apache-flume-1.6.0-cdh5.15.1-bin

export PATH=$FLUME_HOME/bin:$PATH
```

1. 生效 source /etc/profile
2. 到flume下面的conf目录设置

```
cp flume-env.sh.template flume-env.sh
vi flume-env.sh
export JAVA_HOME=/..........  在终端输入echo JAVA_HOME可以查看
```

1. 查看安装成功

在flume的bin目录下输入：flume-ng version

## 十一、zookeeper的安装

下载zookeeper-3.4.5-cdh5.15.1.tar.gz，解压缩到app目录下，然后配到系统环境变量，再source一下

```
#zookeeper environment
export ZK_HOME=/home/willhope/app/zookeeper-3.4.5-cdh5.15.1
export PATH=$ZK_HOME/bin:$PATH
```

将conf目录下的zoo_sample.cfg复制一份为zoo.cfg，然后修改conf目录下的zoo.cfg，设置dataDir=/home/willhope/app/tmp/zookeeper

进入到bin目录，输入 ./zkServer.sh start 启动zookeeper

jps查看成功与否：出现QuorumPeerMain 则表明成功

连接客户端： ./zkCli.sh

## 十二、Kafka安装

官网下载 kafka_2.11-2.4.0.tgz，然后解压到 app目录中，配置到环境变量中，再source一下

```
#kafka environment
export KF_HOME=/home/willhope/app/kafka_2.11-2.4.0
export PATH=$KF_HOME/bin:$PATH
```

- 单节点单brocker的部署以及使用

Kafka目录下的config下的server.properties

```
broker.id=0c
listeners=PLAINTEXT://:9092
host.name=willhope-pc
log.dirs=/home/willhope/app/tmp/kafka-logs
zookeeper.connect=willhope-pc:2181
```

启动Kafka， 在bin目录下执行： kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties ，jps 之后会出现kafka进程

