# Go实现电商微服务

说明：电商系统的内容不需要再赘述了，网上资料太多了，简历上估计也人手一个。本项目做的也比较简单，仅仅从用户、商品、下单、付款这几个部分着手。因此本文档，仅仅只是记录在使用Go开发过程中，整合的各个组件。

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

