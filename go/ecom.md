# Go实现电商微服务

说明：电商系统的内容不需要再赘述了，网上资料太多了，简历上估计也人手一个。本项目做的也比较简单，仅仅从用户、商品、下单、付款这几个部分着手。因此本文档，仅仅只是记录在使用Go开发过程中，主要组件的应用。

### 一、整合zap日志库

#### 1.介绍

zap是uber开源的日志包，以高性能著称。zap除了具有日志基本的功能之外，还具有很多强大的特性：

- 支持常用的日志级别，例如：Debug、Info、Warn、Error、DPanic、Panic、Fatal。

- 性能非常高，适合对性能要求比较高的场景。

- 支持结构化的日志记录。

- 支持预设日志字段。

- 支持针对特定的日志级别，输出调用堆栈。

- 能够打印基本信息，如调用文件/函数名和行号，日志时间等。

- 支持hook。

zap对性能的优化的点：

- 使用强类型，而避免使用interface{}, zap提供zap.String, zap.Int等基础类型的字段输出方法。

- 不使用反射。反射是有代价的，用户在记录的日志的时候，应该清楚每个字段填充需要传入的类型。
- 输出json格式化数据时，不使用json.Marshal和fmt.Fprintf基于interface{}反射的实现方式，而是自己实现了json Encoder， 通过明确的类型调用，直接拼接字符串，最小化性能开销。
- 使用sync.Pool缓存来复用对象，降低了GC。zap中zapcore.Entry和zap.Buffer都是用了缓存池技术，zapcorea.Entry代表一条完整的日志消息。

Zap提供了两种类型的日志记录：SugaredLogger和Logger。在性能很好但不是关键的上下文中，可以使用SugaredLogger。它比其他结构化日志包快4-10倍，并且包含结构化和printf风格的api。当性能和类型安全性非常关键时，请使用Logger。它比SugaredLogger更快，占用内存也少得多，但它只支持结构化日志。通常来说，使用Logger即可。

#### 2.安装

```bash
go get -u go.uber.org/zap
```

#### 3.使用

```go
package main
import "go.uber.org/zap"
func main() {
    //zap提供了NewExample/NewProduction/NewDevelopment构造函数创建Logger。
    logger := zap.NewExample() // 生产环境使用这个，level是info级别，输出格式为json格式
    // logger := zap.NewDevelopment() // 测试环境使用这个，level是debug级别，输出格式为console
    defer logger.Sync() // 刷新缓存
 
    const url = "http://api.ckeen.cn" // 项目地址
 
    logger.Info("fetch url:",zap.String("url", url))
 	//根据创建的Logger对象的Sugar()函数返回SugaredLogger的实例
    sugar := logger.Sugar()
 	// infow/error/panic/errorf
    // 类似一个map，但是会使用反射机制
    sugar.Infow("Failed to fetch URL.",
	    "url", url,
        "token", "xxxx",
	    "attempt", 3,
	    "backoff", time.Second,
    )
    // 类似printf
    sugar.Infof("Failed to fetch URL: %s", url)
 	// 这种方式麻烦，但是使用zap.String可以避免使用反射
    logger.Info("Failed to fetch URL.",
	    zap.String("url", url),
        zap.String("token", "xxxx"),
	    zap.Int("attempt", 3),
	    zap.Duration("backoff", time.Second),
    )
}
// 打印输出结果
// {"level":"info","ts":1655827513.5442681,"caller":"zap/main.go:22","msg":"fetch url:","url":"http://api.ckeen.cn"}
// {"level":"info","ts":1655827513.544397,"caller":"zap/main.go:25","msg":"Failed to fetch URL.","url":"http://api.ckeen.cn","token":"xxxx","attempt":3}
// {"level":"info","ts":1655827513.54445,"caller":"zap/main.go:30","msg":"Failed to fetch URL: http://api.ckeen.cn"}
// {"level":"info","ts":1655827513.5444632,"caller":"zap/main.go:32","msg":"Failed to fetch URL.","url":"http://api.ckeen.cn","token":"xxxx","attempt":3}
```

### 二、Gin整合JWT

#### 1.什么是JWT

JWT 的全称叫做 JSON WEB TOKEN，在目前前后端系统中使用较多。

JWT 是由三段构成的。分别是 HEADER，PAYLOAD，VERIFY SIGNATURE，它们生成的信息通过`.`分割。

- header 是由 一个`typ`和`alg`组成，`typ`会指明为 JWT，而`alg`是所使用的加密算法。

```json
 {	
  "alg": "HS256",	
  "typ": "JWT"	
 }
```

- payload 是 JWT  的载体，也就是我们要承载的信息。这段信息是我们可以自定义的，可以定义我们要存放什么信息，那些字段。该部分信息不宜过多，它会影响 JWT  生成的大小，还有就是请勿将敏感数据存入该部分，该端数据前端是可以解析获取 token 内信息的。官方给了七个默认字段，我们可以不全部使用，也可以加入我们需要的字段。

  | 名称      | 含义          |
  | --------- | ------------- |
  | Audience  | 表示JWT的受众 |
  | ExpiresAt | 失效时间      |
  | Id        | 签发编号      |
  | IssuedAt  | 签发时间      |
  | Issuer    | 签发人        |
  | NotBefore | 生效时间      |
  | Subject   | 主题          |

- VERIFY SIGNATURE是 JWT 的最后一段，该部分是由算法计算完成的。对刚刚的 header 进行 base64Url 编码，对 payload 进行 base64Url 编码，两端完成编码后通过`.`进行连接起来。

  ```
   base64UrlEncode(header).base64UrlEncode(payload)
  ```
  
- 完成上述步骤后，就要通过我们 header 里指定的加密算法对上部分进行加密，同时我们还要插入我们的一个密钥，来确保我的 JWT 签发是安全的。

#### 2.JWT登录原理

简单的说就是当用户登录的时候，服务器校验登录名称和密码是否正确，正确的话，会生成 JWT 返回给客户端。客户端获取到 JWT  后要进行保存，之后的每次请求都会讲 JWT 携带在头部，每次服务器都会获取头部的 JWT  是否正确，如果正确则正确执行该请求，否者验证失败，重新登录。

#### 3.使用jwt-go颁发Token

```go
func CreateJwt(ctx *gin.Context) {
	// 获取用户
	user := &model.User{}
	result := &model.Result{
		Code:    200,
		Message: "登录成功",
		Data:    nil,
	}
	if e := ctx.BindJSON(&user); e != nil {
		result.Message = "数据绑定失败"
		result.Code = http.StatusUnauthorized
		ctx.JSON(http.StatusUnauthorized, gin.H{
			"result": result,
		})
	}
	u := user.QueryByUsername()
    // 验证通过
	if u.Password == user.Password {
        // 设置过期时间，这里是一天
		expiresTime := time.Now().Unix() + int64(config.OneDayOfHours)
		claims := jwt.StandardClaims{
			Audience:  user.Username,     // 受众
			ExpiresAt: expiresTime,       // 失效时间
			Id:        string(user.ID),   // 编号
			IssuedAt:  time.Now().Unix(), // 签发时间
			Issuer:    "gin hello",       // 签发人
			NotBefore: time.Now().Unix(), // 生效时间
			Subject:   "login",           // 主题
		}
        // 设置的密钥，可以从配置文件读取
		var jwtSecret = []byte(config.Secret)
        // 通过 HS256 算法生成 tokenClaims ,这就是我们的 HEADER 部分和 PAYLOAD。
		tokenClaims := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
		if token, err := tokenClaims.SignedString(jwtSecret); err == nil {
			result.Message = "登录成功"
            //将我们的 token 和 Bearer 拼接在一起，同时中间用空格隔开
			result.Data = "Bearer " + token
			result.Code = http.StatusOK
			ctx.JSON(result.Code, gin.H{
				"result": result,
			})
		} else {
			result.Message = "登录失败"
			result.Code = http.StatusOK
			ctx.JSON(result.Code, gin.H{
				"result": result,
			})
		}
	} else {
		result.Message = "登录失败"
		result.Code = http.StatusOK
		ctx.JSON(result.Code, gin.H{
			"result": result,
		})
	}
}
```

通过.http请求测试，结果如下：

```json
{
  "result": {
    "code": 200,
    "message": "登录成功",
    "data": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiIxMjMiLCJleHAiOjE1NjQ3OTY0MzksImp0aSI6Ilx1MDAwMCIsImlhdCI6MTU2NDc5NjQxOSwiaXNzIjoiZ2luIGhlbGxvIiwibmJmIjoxNTY0Nzk2NDE5LCJzdWIiOiJsb2dpbiJ9.CpacmfBSMgmK2TgrT-KwNB60bsvwgyryGQ0pWZr8laU"
  }
}
```

#### 4.校验Token

首先在请求头获取 token ,然后对先把 token 进行解析，将 Bearer 和 JWT 拆分出来，将 JWT 进行校验。我们只需要对我们需要校验的路由进行添加中间件校验即可。

**首先先编写我们的解析 token 方法，`parseToken()`**

```go
func parseToken(token string) (*jwt.StandardClaims, error) {
	jwtToken, err := jwt.ParseWithClaims(token, &jwt.StandardClaims{}, func(token *jwt.Token) (i interface{}, e error) {
		return []byte(config.Secret), nil
	})
	if err == nil && jwtToken != nil {
		if claim, ok := jwtToken.Claims.(*jwt.StandardClaims); ok && jwtToken.Valid {
			return claim, nil
		}
	}
	return nil, err
}
```

**再编写校验函数**

```go
func Auth() gin.HandlerFunc {
	return func(context *gin.Context) {
		result := model.Result{
			Code:    http.StatusUnauthorized,
			Message: "无法认证，重新登录",
			Data:    nil,
		}
		auth := context.Request.Header.Get("Authorization")
		if len(auth) == 0 {
			context.Abort()
			context.JSON(http.StatusUnauthorized, gin.H{
				"result": result,
			})
		}
		auth = strings.Fields(auth)[1]
		// 校验token
		_, err := parseToken(auth)
		if err != nil {
			context.Abort()
			result.Message = "token 过期" + err.Error()
			context.JSON(http.StatusUnauthorized, gin.H{
				"result": result,
			})
		} else {
			println("token 正确")
		}
		context.Next()
	}
}
```

**在路由上的设置**

```
router.GET("/", middleware.Auth(), func(context *gin.Context) {
  context.JSON(http.StatusOK, time.Now().Unix())
})
```

**HTTP请求头：**

```http
GET http://localhost:8080
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiIxMjMiLCJleHAiOjE1NjQ3OTQzNjIsImp0aSI6Ilx1MDAwMCIsImlhdCI6MTU2NDc5NDM0MiwiaXNzIjoiZ2luIGhlbGxvIiwibmJmIjoxNTY0Nzk0MzQyLCJzdWIiOiJsb2dpbiJ9.uQxGMsftyVFtYIGwQVm1QB2djw-uMfDbw81E5LMjliU
```

### 三、解决跨域问题

#### 1.跨域问题

浏览器对于javascript的同源策略的限制,例如[http://a.cn](https://link.zhihu.com/?target=http%3A//a.cn)下面的js不能调用[http://b.cn](https://link.zhihu.com/?target=http%3A//b.cn)中的js,对象或数据(因为[http://a.cn](https://link.zhihu.com/?target=http%3A//a.cn)和[http://b.cn](https://link.zhihu.com/?target=http%3A//b.cn)是不同域),所以跨域就出现了.

**同域：**简单的解释就是域名相同,端口相同,协议相同

#### 2.出现跨域的情况

非简单请求且跨域的情况下,浏览器会发送OPTIONS预检请求。预检请求首先需要向另外一个域名的资源发送一个HTTP OPTIONS请求头,其目的就是为了判断实际发送的请求是安全的。

**简单请求：**

简单请求需满足以下两个条件

1. 请求方法是以下三种方法之一：

- HEAD
- GET
- POST

2. HTTP的头信息不超出以下几种字段：

- Accept
- Accept Language
- Last-Event-ID
- Content-Type:只限于(application/x-www-form-urlencoded,multipart/form-data,text/plain)

**复杂请求：**

非简单请求即是复杂请求，常见的复杂请求有:

1. 请求方法为PUT或DELETE

2. Content-Type字段类型为 application/json

3. 添加核外的http header

跨域的情况下，非简单请求会先发起一次空body的OPTIONS请求，称为"预检"请求，用于向服务器请求权限信息等预检请求被成功响应后，才会发起真正的http请求。浏览器的预检请求结果可以通过Access-Control-Max-Age进行缓存。

#### 3.代码实现

```go
package middlewares

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Cors() gin.HandlerFunc {
	return func(c *gin.Context) {
		method := c.Request.Method

		c.Header("Access-Control-Allow-Origin", "*")
		c.Header("Access-Control-Allow-Headers", "Content-Type,AccessToken,X-CSRF-Token, Authorization, Token, x-token")
		c.Header("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PATCH, PUT")
		c.Header("Access-Control-Expose-Headers", "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers, Content-Type")
		c.Header("Access-Control-Allow-Credentials", "true")

		if method == "OPTIONS" {
			c.AbortWithStatus(http.StatusNoContent)
		}
	}
}
```

### 四、服务注册与发现

#### 1.什么是服务注册和发现

在电商网站上，我们通常碰到某些节假日的促销活动，这种促销活动不是每天都有的，而是具有特别的时效性，而且针对不同地区，平台会给出不同的促销活动。

假如促销活动的产品已经在线上运行。在某个节假日，运营想搞一场促销活动，假设为了为这场促销活动提供服务，我们可能要新开启三个微服务实例来支撑这场促销活动。而与此同时，作为苦逼程序员的你就只有手动去APl gateway 中添加新增的这三个微服务实例的ip 与port，一个真正在线的微服务系统可能有成百上千微服务，难道也要一个一个去手动添加吗?有没有让系统自动去实现这些操作的方法呢?答案当然是有的。
当我们新添加一个微服务实例的时候，微服务就会将自己的 ip 与port 发送到注册中心，在注册中心里面记录起来。当APl gateway需要访问某些微服务的时候，就会去注册中心取到相应的 ip与port。从而实现白动化操作。

#### 2.技术选型

| 名称      | 优点                                                         | 缺点                                                         | 接口     | 一致性算法 |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------- | ---------- |
| zookeeper | 1.提供wathcer机制能够实时获取服务提供者的状态。2.dubbo等框架支持 | 1.没有健康检查2.需要在服务中集成sdk，复杂度高3.不支持多数据中心 | sdk      | Paxos      |
| consul    | 1.简单易用，不需要集成sdk 2.自带健康检查 3.支持多数据中心 4.提供web管理界面 | 1.不能实时获取服务信息的变化通知                             | http/dns | Raft       |
| etcd      | 1.简单易用，不需要集成sdk2.可配置性强                        | 1.没有健康检查 2.需要配合第三方工具一起完成服务发现 3.不支持多数据中心 | http     | Raft       |

#### 3.Paxos共识算法

Paxos 算法就是一种基于消息传递模型的共识算法，解决的问题是一个分布式系统如何就某个值（决议）达成一致。Paxos 算法适用的几种情况：一台机器中多个进程/线程达成数据一致；分布式文件系统或者分布式数据库中多客户端并发读写数据；分布式存储中多个副本响应读写请求的一致性。

Paxos算法运行在允许宕机故障的异步系统中，不要求可靠的消息传递，可容忍消息丢失、延迟、乱序以及重复。它利用大多数 (Majority) 机制保证了2F+1的容错能力，即2F+1个节点的系统最多允许F个节点同时出现故障。

一个或多个提议进程 (Proposer) 可以发起提案 (Proposal)，Paxos算法使所有提案中的某一个提案，在所有进程中达成一致。系统中的多数派同时认可该提案，即达成了一致。最多只针对一个确定的提案达成一致。

Paxos将系统中的角色分为提议者 (Proposer)，决策者 (Acceptor)，和最终决策学习者 (Learner):

- **Proposer**: 提出提案 (Proposal)。Proposal信息包括提案编号 (Proposal ID) 和提议的值 (Value)。
- **Acceptor**：参与决策，回应Proposers的提案。收到Proposal后可以接受提案，若Proposal获得多数Acceptors的接受，则称该Proposal被批准。
- **Learner**：不参与决策，从Proposers/Acceptors学习最新达成一致的提案（Value）。

在多副本状态机中，每个副本同时具有Proposer、Acceptor、Learner三种角色。

![paxos一致性算法角色](../picture/paxos.png)

Paxos算法通过一个决议分为两个阶段（Learn阶段之前决议已经形成）：

1. 第一阶段：Prepare阶段。Proposer向Acceptors发出Prepare请求，Acceptors针对收到的Prepare请求进行Promise承诺。
2. 第二阶段：Accept阶段。Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求，Acceptors针对收到的Propose请求进行Accept处理。
3. 第三阶段：Learn阶段。Proposer在收到多数Acceptors的Accept之后，标志着本次Accept成功，决议形成，将形成的决议发送给所有Learners。

Paxos算法流程中的每条消息描述如下：

- Prepare: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。

- Promise: Acceptors收到Prepare请求后，做出“两个承诺，一个应答”。

两个承诺：

1. 不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Prepare请求。

2. 不再接受Proposal ID小于（注意：这里是< ）当前请求的Propose请求。

一个应答：

不违背以前作出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

- Propose: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
- Accept: Acceptor收到Propose请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。
- Learn: Proposer收到多数Acceptors的Accept后，决议形成，将形成的决议发送给所有Learners。


#### 4.Raft一致性算法

raft一致性算法基于Paxos算法，但是更容易理解和实现。

raft同样利用大多数 (Majority) 机制保证了2F+1的容错能力，即2F+1个节点的系统最多允许F个节点同时出现故障。

raft协议的工作原理进行了高度的概括：raft会在node中先选举出leader，leader完全负责replicated  log的管理。leader负责接受所有客户端更新请求，然后复制到follower节点，并在“安全”的时候执行这些请求。如果leader故障，followers会重新选举出新的leader。

##### （1）leader election

在raft协议中， 任何一个node都处于一下三种状态之一：

- leader
- follower
- candidate

所有节点启动时都是follower状态；在一段时间内如果没有收到来自leader的心跳，从follower切换到candidate，发起选举；如果收到majority的造成票（含自己的一票）则切换到leader状态；如果发现其他节点比自己先更新，则主动切换到follower。

总之，系统中最多只有一个leader，如果在一段时间里发现没有leader，则大家通过选举-投票选出leader。leader会不停的给follower发心跳消息，表明自己的存活状态。如果leader故障，那么follower会转换成candidate，重新选出leader。

##### （2）term

哪个节点做leader是大家投票选举出来的，每个leader工作一段时间，然后选出新的leader继续负责。这根民主社会的选举很像，每一届新的履职期称之为一届任期，在raft协议中，也是这样的，对应的术语叫***term***。

term（任期）以选举（election）开始，然后就是一段或长或短的稳定工作期（normal  Operation），且任期是递增的。

##### （3）选举过程详解

如果follower在*election timeout*内没有收到来自leader的心跳，（也许此时还没有选出leader，大家都在等；也许leader挂了；也许只是leader与该follower之间网络故障），则会主动发起选举。步骤如下：

- 增加节点本地的 *current term* ，切换到candidate状态
- 投自己一票
- 并行给其他节点发送 *RequestVote RPCs*
- 等待其他节点的回复

在这个过程中，根据来自其他节点的消息，可能出现三种结果

1. 收到majority的投票（含自己的一票），则赢得选举，成为leader
2. 被告知别人已当选，那么自行切换到follower
3. 一段时间内没有收到majority投票，则保持candidate状态，重新发出选举

第一种情况，赢得了选举之后，新的leader会立刻给所有节点发消息，广而告之，避免其余节点触发新的选举。在这里，先回到投票者的视角，投票者如何决定是否给一个选举请求投票呢，有以下约束：

- 在任一任期内，单个节点最多只能投一票
- 候选人知道的信息不能比自己的少（这一部分，后面介绍log replication和safety的时候会详细介绍）
- first-come-first-served 先来先得

第二种情况，比如有三个节点A B C。A  B同时发起选举，而A的选举消息先到达C，C给A投了一票，当B的消息到达C时，已经不能满足上面提到的第一个约束，即C不会给B投票，而A和B显然都不会给对方投票。A胜出之后，会给B,C发心跳消息，节点B发现节点A的term不低于自己的term，知道有已经有Leader了，于是转换成follower。

第三种情况，没有任何节点获得majority投票。假设总共有四个节点，Node A,B,C,D。其中，Node C、Node D同时成为了candidate，进入了term 4，但Node  A投了NodeD一票，NodeB投了Node C一票，这就出现了平票 split  vote的情况。这个时候大家都在等啊等，直到超时后重新发起选举。如果出现平票的情况，那么就延长了系统不可用的时间（没有leader是不能处理客户端写请求的），因此raft引入了randomized election timeouts来尽量避免平票情况。同时，leader-based  共识算法中，节点的数目都是奇数个，尽量保证majority的出现。

##### （4）log replication

当有了leader，系统应该进入对外工作期了。客户端的一切请求来发送到leader，leader来调度这些并发请求的顺序，并且保证leader与followers状态的一致性。raft中的做法是，将这些请求以及执行顺序告知followers。leader和followers以相同的顺序来执行这些请求，保证状态一致。

在raft中，leader将客户端请求（command）封装到一个个log entry，将这些log  entries复制（replicate）到所有follower节点，然后大家按相同顺序应用（apply）log  entry中的command，则状态肯定是一致的。

当系统（leader）收到一个来自客户端的写请求，到返回给客户端，整个过程从leader的视角来看会经历以下步骤：

- leader 将client的请求命令作为一条新的日志项写入日志。

- leader 发送AppendEntries RPC 给follower 备份 该日志项。

- follower收到leader的AppendEntries RPC，将该日志项记录到日志并反馈ack。

- leader 收到 半数以上的follower 的ack，即认为消息发送成功

- leader 将 该日志项 提交状态机(state machine)处理

- leader 将执行结果返回给 client

- leader 发送AppendEntries RPC 通知 follower 提交状态机

- follower 收到AppendEntries RPC，follower判断该日志项是否已执行，若未执行则执行commitIndex以及之前的日志项。

logs由顺序编号的log entry组成 ，每个log entry除了包含command，还包含产生该log entry时的leader  term。各个节点的日志并不完全一致，raft算法为了保证高可用，并不是强一致性，而是最终一致性，leader会不断尝试给follower发log entries，直到所有节点的log entries都相同。

在上面的流程中，leader只需要日志被复制到大多数节点即可向客户端返回，一旦向客户端返回成功消息，那么系统就必须保证log（其实是log所包含的command）在任何异常的情况下都不会发生回滚。这里有两个词：commit（committed），apply(applied)，前者是指日志被复制到了大多数节点后日志的状态；而后者则是节点将日志应用到状态机，真正影响到节点状态。

##### （5）safety

在上面提到只要日志被复制到majority节点，就能保证不会被回滚，即使在各种异常情况下，这根leader election提到的选举约束有关。在这一部分，主要讨论raft协议在各种各样的异常情况下如何工作的。

衡量一个分布式算法，有许多属性，如

- safety：nothing bad happens,
- liveness： something good eventually happens.

在任何系统模型下，都需要满足safety属性，即在任何情况下，系统都不能出现不可逆的错误，也不能向客户端返回错误的内容。比如，raft保证被复制到大多数节点的日志不会被回滚，那么就是safety属性。而raft最终会让所有节点状态一致，这属于liveness属性。

##### （6）Election safty

选举安全性，即任一任期内最多一个leader被选出。这一点非常重要，在一个复制集中任何时刻只能有一个leader。系统中同时有多余一个leader，被称之为分裂（ split），这是非常严重的问题，会导致数据的覆盖丢失。在raft中，两点保证了这个属性：

- 一个节点某一任期内最多只能投一票；
- 只有获得majority投票的节点才会成为leader。

因此，**某一任期内一定只有一个leader**。

#### 5.Consul的安装和配置

```bash
docker run -d -p 8500:8500 -p 8300:8300 -p 8301:8301 -p8302:8302 -p 8600:8600/udpconsul consul agent -dev -client=0.0.0.0

docker container update --restart=always 容器名字
```

浏览器访问 http://127.0.0.1:8500

#### 6.go集成Consul

```go
import (
	"fmt"
	"github.com/hashicorp/consul/api"
)
// 服务注册
func Register(address string, port int, name string, tags []string, id string) error {
	cfg := api.DefaultConfig()
	cfg.Address = "192.168.1.103:8500"

	client, err := api.NewClient(cfg)
	if err != nil {
		panic(err)
	}
	//生成对应的检查对象
	check := &api.AgentServiceCheck{
		HTTP:                           "http://192.168.1.102:8021/health",
		Timeout:                        "5s",
		Interval:                       "5s",
		DeregisterCriticalServiceAfter: "10s",
	}

	//生成注册对象
	registration := new(api.AgentServiceRegistration)
	registration.Name = name
	registration.ID = id
	registration.Port = port
	registration.Tags = tags
	registration.Address = address
	registration.Check = check

	err = client.Agent().ServiceRegister(registration)
	client.Agent().ServiceDeregister()
	if err != nil {
		panic(err)
	}
	return nil
}

func AllServices() {
	cfg := api.DefaultConfig()
	cfg.Address = "192.168.1.103:8500"

	client, err := api.NewClient(cfg)
	if err != nil {
		panic(err)
	}

	data, err := client.Agent().Services()
	if err != nil {
		panic(err)
	}
	for key, _ := range data {
		fmt.Println(key)
	}
}
func FilterSerivice() {
	cfg := api.DefaultConfig()
	cfg.Address = "192.168.1.103:8500"

	client, err := api.NewClient(cfg)
	if err != nil {
		panic(err)
	}

	data, err := client.Agent().ServicesWithFilter(`Service == "user-web"`)
	if err != nil {
		panic(err)
	}
	for key, _ := range data {
		fmt.Println(key)
	}
}

func main() {
	//_ = Register("192.168.1.102", 8021, "user-web", []string{"mxshop", "bobby"}, "user-web")
	//AllServices()
	//FilterSerivice()
	fmt.Println(fmt.Sprintf(`Service == "%s"`, "user-srv"))
}
```

#### 7.将用户GRPC服务集成到Consul中

首先，添加config文件

```
package config

type MysqlConfig struct{
	Host string `mapstructure:"host" json:"host"`
	Port int    `mapstructure:"port" json:"port"`
	Name string `mapstructure:"db" json:"db"`
	User string `mapstructure:"user" json:"user"`
	Password string `mapstructure:"password" json:"password"`
}

type ConsulConfig struct{
	Host string `mapstructure:"host" json:"host"`
	Port int    `mapstructure:"port" json:"port"`
}

type ServerConfig struct{
	Name string `mapstructure:"name" json:"name"`
	MysqlInfo MysqlConfig `mapstructure:"mysql" json:"mysql"`
	ConsulInfo ConsulConfig `mapstructure:"consul" json:"consul"`
}

type NacosConfig struct {
	Host      string `mapstructure:"host"`
	Port      uint64    `mapstructure:"port"`
	Namespace string `mapstructure:"namespace"`
	User      string `mapstructure:"user"`
	Password  string `mapstructure:"password"`
	DataId    string `mapstructure:"dataid"`
	Group     string `mapstructure:"group"`
}
```

对上面配置进行初始化。

首先，viper和zap从nacos中读取yml或者json格式的配置，进行初始化

```go
import (
	"encoding/json"
	"fmt"
	"github.com/nacos-group/nacos-sdk-go/clients"
	"github.com/nacos-group/nacos-sdk-go/common/constant"
	"github.com/nacos-group/nacos-sdk-go/vo"
	"github.com/spf13/viper"
	"go.uber.org/zap"
	"mxshop_srvs/user_srv/global"
)
func GetEnvInfo(env string) bool {
	viper.AutomaticEnv()
	return viper.GetBool(env)
}

func InitConfig(){
	//从配置文件中读取出对应的配置
	debug := GetEnvInfo("MXSHOP_DEBUG")
	configFilePrefix := "config"
	configFileName := fmt.Sprintf("user_srv/%s-pro.yaml", configFilePrefix)
	if debug {
		configFileName = fmt.Sprintf("user_srv/%s-debug.yaml", configFilePrefix)
	}

	v := viper.New()
	//文件的路径如何设置
	v.SetConfigFile(configFileName)
	if err := v.ReadInConfig(); err != nil {
		panic(err)
	}
	//这个对象如何在其他文件中使用 - 全局变量
	if err := v.Unmarshal(&global.NacosConfig); err != nil {
		panic(err)
	}
	zap.S().Infof("配置信息: %v", global.NacosConfig)

	//从nacos中读取配置信息
	sc := []constant.ServerConfig{
		{
			IpAddr: global.NacosConfig.Host,
			Port: global.NacosConfig.Port,
		},
	}

	cc := constant.ClientConfig {
		NamespaceId:         global.NacosConfig.Namespace, // 如果需要支持多namespace，我们可以场景多个client,它们有不同的NamespaceId
		TimeoutMs:           5000,
		NotLoadCacheAtStart: true,
		LogDir:              "tmp/nacos/log",
		CacheDir:            "tmp/nacos/cache",
		RotateTime:          "1h",
		MaxAge:              3,
		LogLevel:            "debug",
	}

	configClient, err := clients.CreateConfigClient(map[string]interface{}{
		"serverConfigs": sc,
		"clientConfig":  cc,
	})
	if err != nil {
		panic(err)
	}

	content, err := configClient.GetConfig(vo.ConfigParam{
		DataId: global.NacosConfig.DataId,
		Group:  global.NacosConfig.Group})

	if err != nil {
		panic(err)
	}
	//fmt.Println(content) //字符串 - yaml
	//想要将一个json字符串转换成struct，需要去设置这个struct的tag
	err = json.Unmarshal([]byte(content), &global.ServerConfig)
	if err != nil{
		zap.S().Fatalf("读取nacos配置失败： %s", err.Error())
	}
	fmt.Println(&global.ServerConfig)
}
```

其次，将mysql注册

```go
import (
	"log"
	"mxshop_srvs/user_srv/global"
	"os"
	"time"
	"fmt"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
	"gorm.io/gorm/schema"
	"gorm.io/driver/mysql"
)

func InitDB(){
	c := global.ServerConfig.MysqlInfo
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		c.User, c.Password, c.Host, c.Port, c.Name)
	newLogger := logger.New(
		log.New(os.Stdout, "\r\n", log.LstdFlags), // io writer
		logger.Config{
			SlowThreshold: time.Second,   // 慢 SQL 阈值
			LogLevel:      logger.Silent, // Log level
			Colorful:      true,         // 禁用彩色打印
		},
	)

	// 全局模式
	var err error
	global.DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{
		NamingStrategy: schema.NamingStrategy{
			SingularTable: true,
		},
		Logger: newLogger,
	})
	if err != nil {
		panic(err)
	}
}
```

main函数的编写，主要进行各种配置初始化，服务注册和服务健康检查

```go
package main

import (
	"flag"
	"fmt"
	"net"
	"os"
	"os/signal"
	"syscall"

	"go.uber.org/zap"
	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	"google.golang.org/grpc/health/grpc_health_v1"
	"github.com/satori/go.uuid"

	"mxshop_srvs/user_srv/handler"
	"mxshop_srvs/user_srv/initialize"
	"mxshop_srvs/user_srv/proto"
	"mxshop_srvs/user_srv/global"
	"mxshop_srvs/user_srv/utils"
	"github.com/hashicorp/consul/api"
)

func main() {
	IP := flag.String("ip", "0.0.0.0", "ip地址")
	Port := flag.Int("port", 0, "端口号")

	//初始化
	initialize.InitLogger()
	initialize.InitConfig()
	initialize.InitDB()
	zap.S().Info(global.ServerConfig)

	flag.Parse()
	zap.S().Info("ip: ", *IP)
	if *Port == 0{
		*Port, _ = utils.GetFreePort()
	}

	zap.S().Info("port: ", *Port)

	server := grpc.NewServer()
	proto.RegisterUserServer(server, &handler.UserServer{})
	lis, err := net.Listen("tcp", fmt.Sprintf("%s:%d", *IP, *Port))
	if err != nil {
		panic("failed to listen:" + err.Error())
	}
	//注册服务健康检查
	grpc_health_v1.RegisterHealthServer(server, health.NewServer())

	//服务注册
	cfg := api.DefaultConfig()
	cfg.Address = fmt.Sprintf("%s:%d", global.ServerConfig.ConsulInfo.Host,
		global.ServerConfig.ConsulInfo.Port)

	client, err := api.NewClient(cfg)
	if err != nil {
		panic(err)
	}
	//生成对应的检查对象
	check := &api.AgentServiceCheck{
		GRPC:                           fmt.Sprintf("192.168.0.103:%d", *Port),
		Timeout:                        "5s",
		Interval:                       "5s",
		DeregisterCriticalServiceAfter: "15s",
	}

	//生成注册对象
	registration := new(api.AgentServiceRegistration)
	registration.Name = global.ServerConfig.Name
	serviceID := fmt.Sprintf("%s", uuid.NewV4())
	registration.ID = serviceID
	registration.Port = *Port
	registration.Tags = []string{"imooc", "bobby", "user", "srv"}
	registration.Address = "192.168.0.103"
	registration.Check = check
	//1. 如何启动两个服务
	//2. 即使我能够通过终端启动两个服务，但是注册到consul中的时候也会被覆盖
	err = client.Agent().ServiceRegister(registration)
	if err != nil {
		panic(err)
	}

	go func() {
		err = server.Serve(lis)
		if err != nil {
			panic("failed to start grpc:" + err.Error())
		}
	}()

	//接收终止信号
	quit := make(chan os.Signal)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	if err = client.Agent().ServiceDeregister(serviceID); err != nil{
		zap.S().Info("注销失败")
	}
	zap.S().Info("注销成功")
}
```

#### 8.将web层服务集成到consul中

从注册中心获取用户服务的信息，即获取用户服务的ip和端口

获取信息的代码如下：

```go
func InitSrvConn2()  {
	//从注册中心获取到用户服务的信息
	cfg := api.DefaultConfig()
	consulInfo := global.ServerConfig.ConsulInfo
	cfg.Address = fmt.Sprintf("%s:%d", consulInfo.Host, consulInfo.Port)

	userSrvHost := ""
	userSrvPort := 0
	client, err := api.NewClient(cfg)
	if err != nil {
		panic(err)
	}

	data, err := client.Agent().ServicesWithFilter(fmt.Sprintf("Service == \"%s\"", global.ServerConfig.UserSrvInfo.Name))
	//data, err := client.Agent().ServicesWithFilter(fmt.Sprintf(`Service == "%s"`, global.ServerConfig.UserSrvInfo.Name))
	if err != nil {
		panic(err)
	}
	for _, value := range data{
		userSrvHost = value.Address
		userSrvPort = value.Port
		break
	}
	if userSrvHost == ""{
		zap.S().Fatal("[InitSrvConn] 连接 【用户服务失败】")
		return
	}

	//拨号连接用户grpc服务器 跨域的问题 - 后端解决 也可以前端来解决
	userConn, err := grpc.Dial(fmt.Sprintf("%s:%d", userSrvHost, userSrvPort), grpc.WithInsecure())
	if err != nil {
		zap.S().Errorw("[GetUserList] 连接 【用户服务失败】",
			"msg", err.Error(),
		)
	}
	//1. 后续的用户服务下线了 2. 改端口了 3. 改ip了 负载均衡来做

	//2. 已经事先创立好了连接，这样后续就不用进行再次tcp的三次握手
	//3. 一个连接多个groutine共用，性能 - 连接池
	userSrvClient := proto.NewUserClient(userConn)
	global.UserSrvClient = userSrvClient
}
```

主要步骤：

```go
引入 github.com/hashicorp/consul/api

consulInfo = global.ServerConfig.COnsulInfo
// 从注册中心获取用户服务信息
cfg :=api.DefaultConfig()
// 获取地址
cfg.Address = fmt.Sprintf("%s:%d", consulInfo.Host, consulInfo.Port);
// 装载配置
client, err := api.NewClient(cfg)
data, err := client.Agent().ServicesWithFilter(fmt.Sprintf("Service == \"%s\"", global.ServerConfig.UserSrvInfo))
// 拨号连接
userConn, err := grpc.Dial(fmt.Sprintf("%s:%d" , userSrvHost,global.ServerConfig.UserSrvInfo.Port), grpc.withInsecure())
```

完整代码

```go
cfg := api.DefaultConfig()
consulInfo := global.ServerConfig.ConsulInfo
cfg.Address = fmt.Sprintf("%s:%d", consulInfo.Host, consulInfo.Port)

userSrvHost := ""
userSrvPort := 0
client, err := api.NewClient(cfg)
if err != nil {
	panic(err)
}

data, err := client.Agent( ).ServiceswithFilter(fmt.Sprintf("Service == \"%s\"", global.ServerConfig.UserSrvInfo))
if err != nil {
	panic(err)
}

for _, value := range data{
	userSrvHost = value.Address
	userSrvPort = value.Port
	break
}

//拨号连接用户grpc服务器跨域的问题–后端解决也可以前端来解决
userConn,err := grpc.Dial(fmt.Sprintf("%s:%d" , userSrvHost,global.ServerConfig.UserSrvInfo.Port), grpc.withInsecure())
if err != nil {
    zap.s( ).Errorw( msg: " [GetUserList] 连接【用户服务失败】"，"msg" , err.Error( ),
    )
}

```

### 五、gin集成grpc的负载均衡

引入 

```go
_ "github.com/mbobakov/grpc-consul-resolver" // It's important
```

重新修改初始化连接代码：

```go
func InitSrvConn(){
	consulInfo := global.ServerConfig.ConsulInfo
	userConn, err := grpc.Dial(
		fmt.Sprintf("consul://%s:%d/%s?wait=14s", consulInfo.Host, consulInfo.Port, global.ServerConfig.UserSrvInfo.Name),
		grpc.WithInsecure(),
		grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy": "round_robin"}`),
	)
	if err != nil {
		zap.S().Fatal("[InitSrvConn] 连接 【用户服务失败】")
	}

	userSrvClient := proto.NewUserClient(userConn)
	global.UserSrvClient = userSrvClient
}
```

### 六、分布式配置中心

#### 1.为什么需要分布式配置中心

我们现在有一个项目，使用gin进行开发的，配置文件的话我们知道是一个叫做config.yaml的文件。我们也知道这个配置文件会在项目启动的时候被加载到内存中进行使用的。

考虑两种情况:

a.添加配置项

- 你现在的用户服务有10个部署实例，那么添加配置项你得去十个地方修改配置文件还得重新启动等。
- 即使go的viper能完成修改配置文件自动生效，那么你得考虑其他语言是否也能做到这点，其他的服务是否也一定会使用viper?
  imooc

b.修改配置项

- 大量的服务可能会使用同一个配置，比如我要更好jwt的secrect，这么多实例怎么办?

c.开发、测试、生产环境如何隔离:

- 前面虽然已经介绍了viper，但是依然一样的问题，这么多服务如何统一这种考虑因素?

#### 2.配置中心技术选型

目前最主流的分布式配置中心主要是有spring cloud config、 apollo和nacos，spring cloud属于java的spring体系，我们就考虑apollo和nacos。apollo与nacos都为目前比较流行且维护活跃的2个配置中心。apollo是携程开源，nacos是阿里开源

- apollo大而全，功能完善。nacos小而全，可以对比成django和flask的区别
- 部署nacos更加简单。
- nacos不止支持配置中心还支持服务注册和发现。
- 都支持各种语言，不过apollo是第三方支持的，nacos是官方支持各种语言

#### 3.Nacos的安装

```
docker run --name nacos-standalone -e NODE=standalone -e WM_XIS=512m -e JWMA_XN=512m -e VM_XN=256m -p 8848:8848 -d nacos/nacos-server : latest
```

#### 4.gin集成Nacos

将Nacos中的配置映射成go的struct，因为go本身支持将json的字符串反射成struct。将nacos的yml配置文件转换为json格式。

```json
{
	"name" : "user-web1",
	"port": 8021,
	"user_srv" :{
		"host": "192.168.1.102",
		"port": 50051,
		"name" : "user-srv"
	}，
	"jwt": {
	"key": "5$!UEmvB#nRBCIwab#Sy!zofKEOGLRtE"}，
	"sms" : {
		"key" :"LTAI4FzGkwzJyrKfCex9kpP1",
		"secrect": "r5aWSxybxkuT4ROcwpRqqusXtcwxt5",
		"expire": 300
	}
	"redis":{
		"host": "192.168.1.103",
		"port": 6379
	},
	"consul" : {
		"host":"192.168.1.103"，
		"port": 8500
	}
}
```

定义对应的config

```go
type ConsulConfig struct {
	Host string `mapstructure:"host" json:"host"`
	Port int    `mapstructure:"port" json:"port"`
}

type RedisConfig struct {
	Host   string `mapstructure:"host" json:"host"`
	Port   int    `mapstructure:"port" json:"port"`
	Expire int    `mapstructure:"expire" json:"expire"`
}

type ServerConfig struct {
	Name        string        `mapstructure:"name" json:"name"`
	Host        string         `mapstructure:"host" json:"host"`
	Tags        []string       `mapstructure:"tags" json:"tags"`
	Port        int           `mapstructure:"port" json:"port"`
	UserSrvInfo UserSrvConfig `mapstructure:"user_srv" json:"user_srv"`
	JWTInfo     JWTConfig     `mapstructure:"jwt" json:"jwt"`
	AliSmsInfo  AliSmsConfig  `mapstructure:"sms" json:"sms"`
	RedisInfo   RedisConfig   `mapstructure:"redis" json:"redis"`
	ConsulInfo  ConsulConfig  `mapstructure:"consul" json:"consul"`
}

type NacosConfig struct {
	Host      string `mapstructure:"host"`
	Port      uint64    `mapstructure:"port"`
	Namespace string `mapstructure:"namespace"`
	User      string `mapstructure:"user"`
	Password  string `mapstructure:"password"`
	DataId    string `mapstructure:"dataid"`
	Group     string `mapstructure:"group"`
}
```

从nacos上获取，初始化

```go

import (
	"encoding/json"
	"fmt"

	"github.com/nacos-group/nacos-sdk-go/clients"
	"github.com/nacos-group/nacos-sdk-go/common/constant"
	"github.com/nacos-group/nacos-sdk-go/vo"
	"github.com/spf13/viper"
	"go.uber.org/zap"

	"mxshop-api/user-web/global"
)

func GetEnvInfo(env string) bool {
	viper.AutomaticEnv()
	return viper.GetBool(env)
	//刚才设置的环境变量 想要生效 我们必须得重启goland
}

func InitConfig(){
	debug := GetEnvInfo("MXSHOP_DEBUG")
	configFilePrefix := "config"
	configFileName := fmt.Sprintf("user-web/%s-pro.yaml", configFilePrefix)
	if debug {
		configFileName = fmt.Sprintf("user-web/%s-debug.yaml", configFilePrefix)
	}

	v := viper.New()
	//文件的路径如何设置
	v.SetConfigFile(configFileName)
	if err := v.ReadInConfig(); err != nil {
		panic(err)
	}
	//这个对象如何在其他文件中使用 - 全局变量
	if err := v.Unmarshal(global.NacosConfig); err != nil {
		panic(err)
	}
	zap.S().Infof("配置信息: &v", global.NacosConfig)

	//从nacos中读取配置信息
	sc := []constant.ServerConfig{
		{
			IpAddr: global.NacosConfig.Host,
			Port: global.NacosConfig.Port,
		},
	}

	cc := constant.ClientConfig {
		NamespaceId:         global.NacosConfig.Namespace, // 如果需要支持多namespace，我们可以场景多个client,它们有不同的NamespaceId
		TimeoutMs:           5000,
		NotLoadCacheAtStart: true,
		LogDir:              "tmp/nacos/log",
		CacheDir:            "tmp/nacos/cache",
		RotateTime:          "1h",
		MaxAge:              3,
		LogLevel:            "debug",
	}

	configClient, err := clients.CreateConfigClient(map[string]interface{}{
		"serverConfigs": sc,
		"clientConfig":  cc,
	})
	if err != nil {
		panic(err)
	}

	content, err := configClient.GetConfig(vo.ConfigParam{
		DataId: global.NacosConfig.DataId,
		Group:  global.NacosConfig.Group})

	if err != nil {
		panic(err)
	}
	//fmt.Println(content) //字符串 - yaml
	//想要将一个json字符串转换成struct，需要去设置这个struct的tag
	err = json.Unmarshal([]byte(content), &global.ServerConfig)
	if err != nil{
		zap.S().Fatalf("读取nacos配置失败： %s", err.Error())
	}
	fmt.Println(&global.ServerConfig)
}
```

### 七、文件服务

商品微服务需要使用到本文存储服务，因此本项目使用阿里云OSS，首先需要在阿里云上开通oss服务。

#### 1.阿里云OSS对象存储作用

- 提供标准、低频、归档多种类型，通过生命周期自动化实现数据类型转换与过期管理，覆盖从热到冷多种存储场景。
- 提供RESTFul API、控制台、多种开发语言的SDK、命令行工具、图形化工具等多种使用方式，您可以随时随地通过网络管理您的数据。
- 提供多种数据处理能力，如图片处理、视频截帧、文档预览、图片场景识别、人脸识别、SQL就地查询等，并无缝对接Hadoop生态、以及阿里云函数计算、EMR、DataLakeAnalytics、BatchCompute、MaxCompute、DBS等产品，满足企业数据分析与管理的需求。
- 支持按量付费模式，按实际使用量付费，无需提前一次性投入。同时，自动弹性扩展，不限用户存储的容量和文件数目，满足EB级海量存储需求。
- 支持服务端加密、客户端加密、防盗链、IP黑白名单、细粒度权限管控、日志审计、WORM特性，并获得多项合规认证，包括SEC、FINRA等，满足企业数据安全与合规要求。

#### 2.阿里云OSS资源术语

阿里云支持多种主流的SDK。

| 中文      | 英文      | 说明                                                         |
| --------- | --------- | ------------------------------------------------------------ |
| 存储空间  | Bucket    | 存储空间是您用于存储对象（Object）的容器，所有的对象都必须隶属于某个存储空间。 |
| 对象/文件 | Object    | 对象是 OSS 存储数据的基本单元，也被称为OSS的文件。对象由元信息（Object Meta）、用户数据（Data）和文件名（Key）组成。对象由存储空间内部唯一的Key来标识。 |
| 地域      | Region    | 地域表示 OSS 的数据中心所在物理位置。您可以根据费用、请求来源等综合选择数据存储的地域。详情请查看[OSS已经开通的Region](https://help.aliyun.com/document_detail/31837.htm#concept-zt4-cvy-5db)。 |
| 访问域名  | Endpoint  | Endpoint 表示OSS对外服务的访问域名。OSS以HTTP RESTful API的形式对外提供服务，当访问不同地域的时候，需要不同的域名。通过内网和外网访问同一个地域所需要的域名也是不同的。具体的内容请参见[各个Region对应的Endpoint](https://help.aliyun.com/document_detail/31837.htm#concept-zt4-cvy-5db)。 |
| 访问密钥  | AccessKey | AccessKey，简称 AK，指的是访问身份验证中用到的AccessKeyId 和AccessKeySecret。OSS通过使用AccessKeyId  和AccessKeySecret对称加密的方法来验证某个请求的发送者身份。AccessKeyId用于标识用户，AccessKeySecret是用户用于加密签名字符串和OSS用来验证签名字符串的密钥，其中AccessKeySecret                                 必须保密。 |

#### 3.整合OSS

安装OSS

```go
go get github.com/aliyun/aliyun-oss-go-sdk/oss
```

引入

```go
import (
  "github.com/aliyun/aliyun-oss-go-sdk/oss"
)
```

##### （1）创建Client

Client是OSS的Go客户端，用于管理存储空间和文件等OSS资源。 新建Client时，需要指定Endpoint（Endpoint表示OSS对外服务的访问域名）。

| Region        | Region ID       | 外网Endpoint                 | 内网Endpoint                          |
| ------------- | --------------- | ---------------------------- | ------------------------------------- |
| 华东1（杭州） | oss-cn-hangzhou | oss-cn-hangzhou.aliyuncs.com | oss-cn-hangzhou-internal.aliyuncs.com |

```go
// oss.Timeout(10, 120)表示设置HTTP连接超时时间为10秒（默认值为30秒），HTTP读写超时时间为120秒（默认值为60秒）。0表示永不超时（不推荐使用）
client, err := oss.New("<yourEndpoint>", "<yourAccessKeyId>", "<yourAccessKeySecret>", oss.Timeout(10, 120))
if err != nil {
    fmt.Println("Error:", err)
    os.Exit(-1)
}
```

##### （2）创建与获取bucket

```
// 1. 当没有手动在阿里云oss上创建bucket时，则可以调用CreateBucket(bucketName)函数来创建
// yourBucketName填写Bucket名称。
bucketName := "yourBucketName"
// 创建存储空间。
err = client.CreateBucket(bucketName)
if err != nil {
    handleError(err)
}

// 2. 当已经存在bucket时，可以直接获取bucket进行操作
// 获取存储空间。
bucket, err := client.Bucket(bucketName)
if err != nil {
    handleError(err)
}
```

##### （3）上传文件

```go
 // yourEndpoint填写Bucket对应的Endpoint，以华东1（杭州）为例，填写为https://oss-cn-hangzhou.aliyuncs.com。其它Region请按实际情况填写。
    endpoint := "yourEndpoint"
    // 阿里云账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM用户进行API访问或日常运维，请登录RAM控制台创建RAM用户。
    accessKeyId := "yourAccessKeyId"
    accessKeySecret := "yourAccessKeySecret"
    // yourBucketName填写存储空间名称。
    bucketName := "yourBucketName"
    // yourObjectName填写Object完整路径，完整路径不包含Bucket名称。例如，first.jpg
    objectName := "yourObjectName"
    // yourLocalFileName填写本地文件的完整路径。
    localFileName := "yourLocalFileName"
    // 创建OSSClient实例。
    client, err := oss.New(endpoint, accessKeyId, accessKeySecret)
    if err != nil {
        handleError(err)
    }
    // 获取存储空间。
    bucket, err := client.Bucket(bucketName)
    if err != nil {
        handleError(err)
    }
    // 上传文件。
    err = bucket.PutObjectFromFile(objectName, localFileName)
    if err != nil {
        handleError(err)
    }
```

##### （4）下载文件

```go
  // yourObjectName填写Object完整路径，完整路径中不能包含Bucket名称
    objectName := "yourObjectName"
    // yourDownloadedFileName填写本地文件的完整路径。
    downloadedFileName := "yourDownloadedFileName"
    // 创建OSSClient实例。
    client, err := oss.New(endpoint, accessKeyId, accessKeySecret)
    if err != nil {
        handleError(err)
    }
    // 获取存储空间。
    bucket, err := client.Bucket(bucketName)
    if err != nil {
        handleError(err)
    }
    // 下载文件。
    err = bucket.GetObjectToFile(objectName, downloadedFileName)
    if err != nil {
        handleError(err)
    }
```

##### （5）列举文件

```go
 // yourEndpoint填写Bucket对应的Endpoint，以华东1（杭州）为例，填写为https://oss-cn-hangzhou.aliyuncs.com。其它Region请按实际情况填写。
    endpoint := "yourEndpoint"
    // 阿里云账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM用户进行API访问或日常运维，请登录RAM控制台创建RAM用户。
    accessKeyId := "yourAccessKeyId"
    accessKeySecret := "yourAccessKeySecret"
    // 创建OSSClient实例。
    client, err := oss.New("yourEndpoint", "yourAccessKeyId", "yourAccessKeySecret")
    if err != nil {
        HandleError(err)
    }
    // 获取存储空间。
    bucketName := "yourBucketName"
    bucket, err := client.Bucket(bucketName)
    if err != nil {
        handleError(err)
    }
    // 列举文件。
    marker := ""
    for {
        //oss.Marker(marker)这句话标明本次列举文件的起点
        lsRes, err := bucket.ListObjects(oss.Marker(marker))
        // 设置列举文件的最大个数，并列举文件。
        // lsRes, err := bucket.ListObjects(oss.MaxKeys(200))

        if err != nil {
            HandleError(err)
        }
        // 打印列举文件，默认情况下一次返回100条记录。 
        for _, object := range lsRes.Objects {
            fmt.Println("Bucket: ", object.Key)
        }
        if lsRes.IsTruncated {
            marker = lsRes.NextMarker
        } else {
            break
        }
    }
```

##### （6）删除文件

```go
 // yourEndpoint填写Bucket对应的Endpoint，以华东1（杭州）为例，填写为https://oss-cn-hangzhou.aliyuncs.com。其它Region请按实际情况填写。
    endpoint := "yourEndpoint"
    // 阿里云账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM用户进行API访问或日常运维，请登录RAM控制台创建RAM用户。
    accessKeyId := "yourAccessKeyId"
    accessKeySecret := "yourAccessKeySecret"
    // yourBucketName填写存储空间名称。
    bucketName := "yourBucketName"
    // yourObjectName填写Object完整路径，完整路径中不能包含Bucket名称。
    objectName := "yourObjectName"
    // 创建OSSClient实例。
    client, err := oss.New(endpoint, accessKeyId, accessKeySecret)
    if err != nil {
        handleError(err)
    }
    // 获取存储空间。
    bucket, err := client.Bucket(bucketName)
    if err != nil {
        handleError(err)
    }
    // 删除文件。
    err = bucket.DeleteObject(objectName)
    if err != nil {
        handleError(err)
    }
```

##### （7）直传OSS

当出现大文件上传时，可以直接让浏览器端直传文件到OSS上。可以先让浏览器发起请求，gin发送给浏览器一个签名，浏览器带着这个签名向OSS发起文件直传。客户端进行表单直传到OSS时，会从浏览器向OSS发送带有Origin的请求消息。OSS对带有Origin头的请求消息会进行跨域规则(CORS)的验证。因此需要为Bucket设置跨域规则以支持Post方法。

1.登录OSS管理控制台。

2.单击Bucket列表，之后单击目标Bucket名称。

3.单击权限管理>跨域设置，在跨域设置区域单击设置。

4.单击创建规则，配置如下图所示。

### 八、库存服务

电商系统中最重要的一点就是库存服务，库存服务与商品服务、订单服务、支付服务都有着极强的相关性。为了防止恶意下单，需要有一个超时机制和库存归还的步骤。

<img src="../picture/inventory.png">

#### 1.库存扣减

库存扣减的时候，需要根据下单的商品id查询是否有库存，如果没有，则直接返回没有库存信息，如果有，则判断库存是否充足，库存充足再扣减。

**本地事务问题**

例如，一个订单有三件商品，1号商品扣减10份,  2号商品扣减5份, 3号商品扣减20份。必须保证三件商品都扣减成功，要么都失败。因此需要手动开启事务。

```go
//开始事务
tx := db.Begin()
//在事务中家行一些 db 操作(从这里开始，您应该使用‘tx’而不是'db')
tx.Create(...)
// ...
//遇到错误时回滚事务
tx.Rollback()
//否则，提交事务
tx.Commit()
```

```go
//扣减库存，本地事务[1 : 10,2:5，3:20]
//数据库基本的一个应用场景︰数据库事务
tx := global.DB.Begin()
for _, goodInfo := range req.GoodsInfo {
	var inv model.Inventory
	if result := global.DB.First(&inv，goodInfo.GoodsId); result.RowsAffected == 0 {
        tx.Rollback()//回滚之前的操作
		return nil,status.Errorf(codes.InvalidArgument,format: "没有库存信息")
	}
	//判断库存是否充足
	if inv. Stocks < goodInfo.Num {
        tx.Rollback()//回滚之前的操作
		return nil,status.Errorf(codes.ResourceExhausted,format: "库存不足")
	}
	//扣减
	inv.Stocks -= goodInfo.Num
    tx.Save(&inv)
}
tx.Commit()//需要自己手动提交操作
return &emptypb.Empty{}, nil

```

**并发问题**

例如，两个线程或者两个用户对同一个商品进行下单的时候，可能会造成超售超卖的问题，并发问题无法正确扣减。这里需要使用到锁，因为本项目是微服务式的，这里使用分布式锁。

```go
// var m sync.Mutex
func (*InventoryServer) Sell(ctx context.Context, req *proto.SellInfo) (*emptypb.Empty, error) {
	//扣减库存， 本地事务 [1:10,  2:5, 3: 20]
	//数据库基本的一个应用场景：数据库事务
	//并发情况之下 可能会出现超卖 1
	client := goredislib.NewClient(&goredislib.Options{
		Addr: "192.168.0.104:6379",
	})
	pool := goredis.NewPool(client) // or, pool := redigo.NewPool(...)
	rs := redsync.New(pool)
	// 开启事务
	tx := global.DB.Begin()
	//m.Lock() //获取锁 这把锁有问题吗？  假设有10w的并发， 这里并不是请求的同一件商品  这个锁就没有问题了吗？

	//这个时候应该先查询表，然后确定这个订单是否已经扣减过库存了，已经扣减过了就别扣减了
	//并发时候会有漏洞， 同一个时刻发送了重复了多次， 使用锁，分布式锁
	sellDetail := model.StockSellDetail{
		OrderSn: req.OrderSn,
		Status:  1,
	}
	var details []model.GoodsDetail
	for _, goodInfo := range req.GoodsInfo {
		details = append(details, model.GoodsDetail{
			Goods: goodInfo.GoodsId,
			Num: goodInfo.Num,
		})

		var inv model.Inventory
		//if result := tx.Clauses(clause.Locking{Strength: "UPDATE"}).Where(&model.Inventory{Goods:goodInfo.GoodsId}).First(&inv); result.RowsAffected == 0 {
		//	tx.Rollback() //回滚之前的操作
		//	return nil, status.Errorf(codes.InvalidArgument, "没有库存信息")
		//}

		//for {
        // 获取锁
		mutex := rs.NewMutex(fmt.Sprintf("goods_%d", goodInfo.GoodsId))
		if err := mutex.Lock(); err != nil {
			return nil, status.Errorf(codes.Internal, "获取redis分布式锁异常")
		}

		if result := global.DB.Where(&model.Inventory{Goods:goodInfo.GoodsId}).First(&inv); result.RowsAffected == 0 {
			tx.Rollback() //回滚之前的操作
			return nil, status.Errorf(codes.InvalidArgument, "没有库存信息")
		}
		//判断库存是否充足
		if inv.Stocks < goodInfo.Num {
			tx.Rollback() //回滚之前的操作
			return nil, status.Errorf(codes.ResourceExhausted, "库存不足")
		}
		//扣减， 会出现数据不一致的问题 - 锁，分布式锁
		inv.Stocks -= goodInfo.Num
		tx.Save(&inv)

		if ok, err := mutex.Unlock(); !ok || err != nil {
			return nil, status.Errorf(codes.Internal, "释放redis分布式锁异常")
		}
			//update inventory set stocks = stocks-1, version=version+1 where goods=goods and version=version
			//这种写法有瑕疵，为什么？
			//零值 对于int类型来说 默认值是0 这种会被gorm给忽略掉
			//if result := tx.Model(&model.Inventory{}).Select("Stocks", "Version").Where("goods = ? and version= ?", goodInfo.GoodsId, inv.Version).Updates(model.Inventory{Stocks: inv.Stocks, Version: inv.Version+1}); result.RowsAffected == 0 {
			//	zap.S().Info("库存扣减失败")
			//}else{
			//	break
			//}
		//}
		//tx.Save(&inv)
	}
	sellDetail.Detail = details
	//写selldetail表
	if result := tx.Create(&sellDetail); result.RowsAffected == 0 {
		tx.Rollback()
		return nil, status.Errorf(codes.Internal, "保存库存扣减历史失败")
	}
	tx.Commit() // 需要自己手动提交操作
	//m.Unlock() //释放锁
	return &emptypb.Empty{}, nil
}
```

#### 2.库存归还

有多种情况： 1：订单超时归还 2. 订单创建失败，归还之前扣减的库存 3. 手动归还

```go
func (*InventoryServer) Reback(ctx context.Context, req *proto.SellInfo) (*emptypb.Empty, error) {
	//库存归还： 1：订单超时归还 2. 订单创建失败，归还之前扣减的库存 3. 手动归还
	tx := global.DB.Begin()
    m.Lock()
	for _, goodInfo := range req.GoodsInfo {
		var inv model.Inventory
		if result := global.DB.Where(&model.Inventory{Goods:goodInfo.GoodsId}).First(&inv); result.RowsAffected == 0 {
			tx.Rollback() //回滚之前的操作
			return nil, status.Errorf(codes.InvalidArgument, "没有库存信息")
		}

		//扣减， 会出现数据不一致的问题 - 锁，分布式锁
		inv.Stocks += goodInfo.Num
		tx.Save(&inv)
	}
	tx.Commit() // 需要自己手动提交操作
    m.Unlock() //释放锁
	return &emptypb.Empty{}, nil
}
```

#### 3.分布式锁

为了提高服务并发，可能会在多个服务器上配置服务。上述的库存扣减和归还都使用了Mutex锁，这个锁只能管住在自己单体服务中的协程，但是无法做到去给别的单体服务协程加锁。

实际上，分布式锁就是给所有的服务提供一个统一的锁。

##### (1)常见的分布式锁实现方案

1. 基于mysql的悲观锁、乐观锁（不要与mysql中提供的锁机制表锁、行锁、排他锁、共享锁混为一谈）

2. 基于redis的分布式锁
3. 基于zookeeper的分布式锁

##### (2)基于mysql实现

**悲观锁**：处理数据时持悲观态度，认为会发生并发冲突，获取和修改数据时，别人会修改数据。所以在整个数据处理过程中，需要将数据锁定。悲观锁的实现，需要依靠数据库mysql的排他锁机制来实现。

查询条件后面使用for update，如果有明确的查询条件，那么将会锁住明确的数据，如果查询的条件属性没有索引，会将整个表锁住。

**gorm的实现：**

```go
tx.Clauses(clause.Locking{Strength: "UPDATE"}).Where(&model.Inventory{Goods:goodInfo.GoodsId}).First(&inv);
```

**乐观锁：**利用数据库本身的行锁，使用了version号来解决，提高了悲观锁的性能。不会让数据库加锁，也不会出现数据不一致。

**gorm的实现：**

```go
tx.Model(&model.Inventory{}).Select("Stocks", "Version").Where("goods = ? and version= ?", goodInfo.GoodsId, inv.Version).Updates(model.Inventory{Stocks: inv.Stocks, Version: inv.Version+1})
```

##### (3)基于redis实现分布式锁

使用工具redsync，访问地址：https://github.com/go-redsync/redsync

### 九、支付宝支付

支付宝开放平台注册自己的账号，并创建一个应用，选择网页移动应用，创建时需要添加一个网址url。

开发时，可以使用沙箱环境用于测试，支付宝提供了测试账户（包括卖家和买家账号）。系统默认会生成一个APPID和支付宝网关、RSA密钥。

支付步骤：浏览器下订单，平台返回支付宝url，跳转到支付宝支付，支付成功后，支付宝返回信息

这里使用第三方alipay工具，引入工具：

```go
import "github.com/smartwalle/alipay"
```

获取client

```go
appID:="应用id"
privateKey := "应用私钥"
aliPublicKey := "支付宝返回的公钥"// 这个可以使用支付宝开发工具，生成一个
var client, err = alipay.New(appId, privateKey,false)
if err != nil {
  panic(err)
}
```

加载公钥，设置订单各种信息

```go
err = client.LoadAliPayPublicKey(aliPublicKey)
if err != nil {
    panic(err)
}
// 手机支付，使用TradeWapPay，网页支付使用TradePagePay{}
var p = alipay.TradeWapPay{}
// 通知URL，即支付宝完成操作后，会给指定的url发送支付结果通知
p.NotifyURL = "http : //xxx"
// 设置支付完成后的跳转页面
p.ReturnURL = "http : //xx×"
p.Subject ="标题"
p.outTradeNo ="传递一个唯一单号"
// 价格
p.TotalAmount = "10.00"
// 手机支付，使用QUICK_WAP_WAY，网页支付使用FAST_INSTANT_TRADE_PAY
p.ProductCode = "QUICK_WAP_WAY"

// 订单支付url
url, err := client.TradePagePay(p)
```

#### 1.支付宝回调通知

```go
func Notify(ctx *gin.Context) {
	//支付宝回调通知
	client, err := alipay.New(global.ServerConfig.AliPayInfo.AppID, global.ServerConfig.AliPayInfo.PrivateKey, false)
	if err != nil {
		zap.S().Errorw("实例化支付宝失败")
		ctx.JSON(http.StatusInternalServerError, gin.H{
			"msg":err.Error(),
		})
		return
	}
	err = client.LoadAliPayPublicKey((global.ServerConfig.AliPayInfo.AliPublicKey))
	if err != nil {
		zap.S().Errorw("加载支付宝的公钥失败")
		ctx.JSON(http.StatusInternalServerError, gin.H{
			"msg":err.Error(),
		})
		return
	}

	noti, err := client.GetTradeNotification(ctx.Request)
	if err != nil {
		ctx.JSON(http.StatusInternalServerError, gin.H{})
		return
	}

	_, err = global.OrderSrvClient.UpdateOrderStatus(context.Background(), &proto.OrderStatus{
		OrderSn: noti.OutTradeNo,
		Status:  string(noti.TradeStatus),
	})
	if err != nil {
		ctx.JSON(http.StatusInternalServerError, gin.H{})
		return
	}
	ctx.String(http.StatusOK, "success")
}
```

### 十、elasticsearch

#### 1.为什么需要elasticsearch

mysql搜索面临的问题：a.数据量大时，搜索性能低下   b.没有相关性排名   c.无法全文搜索   d.搜索不准确-没有分词

Elasticsearch是一个分布式可扩展的实时搜索和分析引擎，一个建立在全文搜索引擎Apache Lucene(TM)基础上的搜索引擎。当然Elasticsearch并不仅仅是Lucene那么简单，它不仅包括了全文搜索功能，还可以进行以下工作:

- 分布式实时文件存储，并将每一个字段都编入索引，使其可以被搜索
- 实时分析的分布式搜索引擎
- 可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据

#### 2.什么是全文搜索

我们生活中的数据总体分为两种:结构化数据和非结构化数据。

结构化数据：指具有固定格式或有限长度的数据，如数据库，元数据等。

非结构化数据：指不定长或无固定格式的数据，如邮件，word文档等。非结构化数据又一种叫法叫全文数据。

按照数据的分类，搜索也分为两种:

对结构化数据的搜索：如对数据库的搜索，用SQL语句。再如对元数据的搜索，如利用windows搜索对文件名，类型，修改时间进行搜索等。

对非结构化数据的搜索：如利用windows的搜索也可以搜索文件，Linux下的grep命令，再如用Google和百度可以搜索大量内容数据。

#### 3.适用场景

- 维基百科
- The Guardian、新闻
- 电商网站、检索商品
- 日志数据分析、logstash采集日志、ES进行复杂的数据分析(ELK)
- 商品价格监控网站、用户设定价格阈值
- BI系统、商业智能、ES执行数据分析和挖掘

#### 4.ES特点

- 可以作为一个大型的分布式集群(数百台服务器)技术，处理PB级数据，服务大公司，可以运行在单机上，服务小公司。
- ES不是什么新技术，主要是将全文检索、数据分析以及分布式技术合并在一起，才形成了独一无二的ES.lucene (全文检索)、商用的数据分析软件、分布式数据库(mycat)
- 对用户而言，是开箱即用，非常简单，作为中小型的应用，直接3分钟部署ES，就可以作为生产环境的系统使用，数据量不大，操作不是很复杂。
- 数据库的功能面对很多领域是不够用的(事务，还有各种联机事务的操作):特殊的功能，比如全文检索、同义词处理、相关度排名、复杂数据分析、海量数据近实时处理;ES作为传统数据库的一个补充，提供了数据库所不能提供的很多功能。

#### 5.ES的docker安装

```bash
# 新建es的config配置文件
mkdir -p es_data/elasticsearch/config es_data/elasticsearch/data es_data/elasticsearch/plugins
# 给目录设置权限
chmod 777 -R es_data
# 写入配置到elasticsearch.yml
echo "http.host: 0.0.0.0" >> es_data/elasticsearch/config/elasticsearch.yml
# 安装es
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms256m -Xmx512m" \
-v /home/tony/es_data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /home/tony/es_data/elasticsearch/data:/usr/share/elasticsearch/data \
-v /home/tony/es_data/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.10.1
```

#### 6.Kibana的docker安装

```bash
docker run -d --name kibana -e ELASTICSEARCH_HOSTS="http://192.168.31.35:9200" -p 5601:5601 kibana:7.10.1
```

#### 7.ES中的概念

| mysql    | es                                           |
| -------- | -------------------------------------------- |
| database | index（7.x版本可以理解为table）              |
| table    | type（7.x开始type为固定值_doc，指定数据类型) |
| row      | document                                     |
| column   | field                                        |
| schema   | mapping                                      |
| sql      | DSL                                          |

**索引**：ES将他的数据存储到一个或多个索引中，索引就像数据库，可以向索引写入文档或者从索引中读取文档。

**文档**：文档是ES中的主要实体，所有es最终会归结到文档的搜索上，从客户端上看，文档就像一个json对象，文档由字段构成，每个字段包含字段名以及一个或多个字段值。文档之间可能有各自不同的字段集合，文档没有固定的模式或者强制结构。

```bash
# 查看索引
GET _cat/indices
# 获取具体数据
GET /account/_source/1

# Put添加数据
PUT /account/_doc/1
{
	"name": "wendy",
	"age": 18,
	"company": {
		"name": "cqust",
		"address": "cq"
	}
}

# 返回
{
  "_index" : "account",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

# Post添加
POST user/_doc/1
{
	"name" : "bobby",
	"company" : "imooc"
}

# 如果想要创建的索引已经有了，让其报错
POST user/_create/1
{
	"name" : "bobby",
	"company" : "imooc"
}
```

#### 8.ES中的查询方式

1. URI带有查询条件(轻量查询)：查询能力有限，不是所有的查询都可以使用此方式
2. 请求体中带有查询条件(复杂查询)
   查询条件以JSON格式表现，作为查询请求的请求体，适合复杂的查询

```bash
# 普通查询
GET user/_search?q=bobby
# request查询
GET account/_search
{
	"query": {
		"match_all": {}
	}
}
```

#### 9.Go语言集成ES

ES查询基于HTTP，因此请求非常方便。引入官方/第三方插件。第三方的使用比较方便，因此这里引入第三方库，地址为：https://github.com/olivere/elastic

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"reflect"
	"time"

	"github.com/olivere/elastic"
)

// Tweet is a structure used for serializing/deserializing data in Elasticsearch.
type Tweet struct {
	User     string                `json:"user"`
	Message  string                `json:"message"`
	Retweets int                   `json:"retweets"`
	Image    string                `json:"image,omitempty"`
	Created  time.Time             `json:"created,omitempty"`
	Tags     []string              `json:"tags,omitempty"`
	Location string                `json:"location,omitempty"`
	Suggest  *elastic.SuggestField `json:"suggest_field,omitempty"`
}

const mapping = `
{
	"settings":{
		"number_of_shards": 1,
		"number_of_replicas": 0
	},
	"mappings":{
		"tweet":{
			"properties":{
				"user":{
					"type":"keyword"
				},
				"message":{
					"type":"text",
					"store": true,
					"fielddata": true
				},
				"image":{
					"type":"keyword"
				},
				"created":{
					"type":"date"
				},
				"tags":{
					"type":"keyword"
				},
				"location":{
					"type":"geo_point"
				},
				"suggest_field":{
					"type":"completion"
				}
			}
		}
	}
}`

func main() {
	// Starting with elastic.v5, you must pass a context to execute each service
	ctx := context.Background()

	// Obtain a client and connect to the default Elasticsearch installation
	// on 127.0.0.1:9200. Of course you can configure your client to connect
	// to other hosts and configure it in various other ways.
    // 创建一个链接
    host:="http://192.168.31.35:9200"
    client, err := elastic.NewClient(elastic.SetURL(host),elastic.SetSniff(false))
	if err != nil {
		// Handle error
		panic(err)
	}

	// Ping the Elasticsearch server to get e.g. the version number
	info, code, err := client.Ping("http://127.0.0.1:9200").Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Elasticsearch returned with code %d and version %s\n", code, info.Version.Number)

	// Getting the ES version number is quite common, so there's a shortcut
	esversion, err := client.ElasticsearchVersion("http://127.0.0.1:9200")
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Elasticsearch version %s\n", esversion)

	// Use the IndexExists service to check if a specified index exists.
	exists, err := client.IndexExists("twitter").Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	if !exists {
		// Create a new index.
		createIndex, err := client.CreateIndex("twitter").BodyString(mapping).Do(ctx)
		if err != nil {
			// Handle error
			panic(err)
		}
		if !createIndex.Acknowledged {
			// Not acknowledged
		}
	}

	// Index a tweet (using JSON serialization)
	tweet1 := Tweet{User: "olivere", Message: "Take Five", Retweets: 0}
	put1, err := client.Index().
		Index("twitter").
		Type("tweet").
		Id("1").
		BodyJson(tweet1).
		Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Indexed tweet %s to index %s, type %s\n", put1.Id, put1.Index, put1.Type)

	// Index a second tweet (by string)
	tweet2 := `{"user" : "olivere", "message" : "It's a Raggy Waltz"}`
	put2, err := client.Index().
		Index("twitter").
		Type("tweet").
		Id("2").
		BodyString(tweet2).
		Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Indexed tweet %s to index %s, type %s\n", put2.Id, put2.Index, put2.Type)

	// Get tweet with specified ID
	get1, err := client.Get().
		Index("twitter").
		Type("tweet").
		Id("1").
		Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	if get1.Found {
		fmt.Printf("Got document %s in version %d from index %s, type %s\n", get1.Id, get1.Version, get1.Index, get1.Type)
	}

	// Flush to make sure the documents got written.
	_, err = client.Flush().Index("twitter").Do(ctx)
	if err != nil {
		panic(err)
	}

	// Search with a term query
	termQuery := elastic.NewTermQuery("user", "olivere")
	searchResult, err := client.Search().
		Index("twitter").   // search in index "twitter"
		Query(termQuery).   // specify the query
		Sort("user", true). // sort by "user" field, ascending
		From(0).Size(10).   // take documents 0-9
		Pretty(true).       // pretty print request and response JSON
		Do(ctx)             // execute
	if err != nil {
		// Handle error
		panic(err)
	}

	// searchResult is of type SearchResult and returns hits, suggestions,
	// and all kinds of other information from Elasticsearch.
	fmt.Printf("Query took %d milliseconds\n", searchResult.TookInMillis)

	// Each is a convenience function that iterates over hits in a search result.
	// It makes sure you don't need to check for nil values in the response.
	// However, it ignores errors in serialization. If you want full control
	// over iterating the hits, see below.
	var ttyp Tweet
	for _, item := range searchResult.Each(reflect.TypeOf(ttyp)) {
		if t, ok := item.(Tweet); ok {
			fmt.Printf("Tweet by %s: %s\n", t.User, t.Message)
		}
	}
	// TotalHits is another convenience function that works even when something goes wrong.
	fmt.Printf("Found a total of %d tweets\n", searchResult.TotalHits())

	// Here's how you iterate through results with full control over each step.
	if searchResult.Hits.TotalHits > 0 {
		fmt.Printf("Found a total of %d tweets\n", searchResult.Hits.TotalHits)

		// Iterate through results
		for _, hit := range searchResult.Hits.Hits {
			// hit.Index contains the name of the index

			// Deserialize hit.Source into a Tweet (could also be just a map[string]interface{}).
			var t Tweet
			err := json.Unmarshal(*hit.Source, &t)
			if err != nil {
				// Deserialization failed
			}

			// Work with tweet
			fmt.Printf("Tweet by %s: %s\n", t.User, t.Message)
		}
	} else {
		// No hits
		fmt.Print("Found no tweets\n")
	}

	// Update a tweet by the update API of Elasticsearch.
	// We just increment the number of retweets.
	update, err := client.Update().Index("twitter").Type("tweet").Id("1").
		Script(elastic.NewScriptInline("ctx._source.retweets += params.num").Lang("painless").Param("num", 1)).
		Upsert(map[string]interface{}{"retweets": 0}).
		Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("New version of tweet %q is now %d\n", update.Id, update.Version)

	// ...

	// Delete an index.
	deleteIndex, err := client.DeleteIndex("twitter").Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	if !deleteIndex.Acknowledged {
		// Not acknowledged
	}
}
```

#### 10.Match查询

```go
// 方法一：
// 第一个是查询字段，第二个是查询字段的值
q := elastic.NewMatchQuery("address","street")
// 执行
src, err := q.Source()
if err != nil {
	panic(err)
}
data, err := json.Marshal(src)
got := string(data)
fmt.Println(got)

// 方法二：
q := elastic.NewMatchQuery("address","street")
result, err := client.Search().Index("user").Query(q).Do(context.Background())
if err != nil {
    panic(err)
}
// 搜索结果数量
total := result.Hits.TatolHits.Value
for index, value := range result.Hits.Hits {
    if jsonData, err := value.Source.MarshalJSON(); err != nil {
        fmt.Println(string(json))
    }else{
        panic(err)
    }
}
```

#### 11.保存数据到es中

```go
tweet1 := Tweet{User: "olivere", Message: "Take Five", Retweets: 0}
	put1, err := client.Index().
		Index("twitter").
		Type("tweet").
		Id("1").
		BodyJson(tweet1).
		Do(ctx)
	if err != nil {
		// Handle error
		panic(err)
	}
	fmt.Printf("Indexed tweet %s to index %s, type %s\n", put1.Id, put1.Index, put1.Type)
```

#### 12.添加索引

```go
// 定义表结构
// Tweet is a structure used for serializing/deserializing data in Elasticsearch.
type Tweet struct {
	User     string                `json:"user"`
	Message  string                `json:"message"`
	Retweets int                   `json:"retweets"`
	Image    string                `json:"image,omitempty"`
	Created  time.Time             `json:"created,omitempty"`
	Tags     []string              `json:"tags,omitempty"`
	Location string                `json:"location,omitempty"`
	Suggest  *elastic.SuggestField `json:"suggest_field,omitempty"`
}

const mapping = `
{
	"settings":{
		// 分片数量
		"number_of_shards": 1,
		// 副本数量
		"number_of_replicas": 0
	},
	"mappings":{
		"tweet":{
			"properties":{
				"user":{
					"type":"text"，
					"analyzer": "ik_max_word"
				},
				"message":{
					"type":"text",
					"store": true,
					"fielddata": true
				},
				"image":{
					"type":"keyword"
				},
				"created":{
					"type":"date"
				},
				"tags":{
					"type":"keyword"
				},
				"location":{
					"type":"geo_point"
				},
				"suggest_field":{
					"type":"completion"
				}
			}
		}
	}
}`

if !exists {
    // Create a new index.
    createIndex, err := client.CreateIndex("twitter").BodyString(mapping).Do(ctx)
    if err != nil {
        // Handle error
        panic(err)
    }
    if !createIndex.Acknowledged {
        // Not acknowledged
    }
}
```

#### 13.将mysql商品数据同步到ES中

```go
var goods []model.Goods
db.Find(&goods)
for _, g = range goods {
    esModel := model.EsGoods{
        ID: g.ID,
        CategoryID: g.CategoryID,
        BrandsID: g.BrandsID,
        OnSale: g.Onsale,
        ShipFree: g.ShipFree,
        IsNew: g.IsNew,
        IsHot: g.IsHot,
        Name: g.Name,
        ClickNum: g.ClickNum,
        SoldNum: g.SoldNum,
        FavNum: g.FavNum,
        MarketPrice: g.MarketPrice,
        GoodsBrief: g.GoodsBrief,
        ShopPrice: g.ShopPrice,
    }
    _,err := global.EsClient.Index().Index(esModel.GetIndexName()).BodyJson(esModel).Id(strconv.Itoa(int(g,ID)).Do(context.Background())
     if err != nil {panic(err)}
     
}
```

