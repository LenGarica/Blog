# Go实现Redis

本项目来源于这个 https://github.com/HDT3213/godis 项目。作者发布了详细的博客，详见 https://www.cnblogs.com/Finley/category/1598973.html。

本项目为学习项目，学习目的：加强Go语言基础，深入Redis相关细节。

## 一、实现TCP服务器

### 1.规定TCP服务器接口

首先，定义TCP服务接口，主要是为了规范TCP服务器的作用，用来建立连接和关闭连接，具体业务逻辑需要在具体的方法中实现。

```go
package tcp

import (
	"context"
	"net"
)

// Handler 接口定义tcp连接和关闭业务
type Handler interface {
	Handler(ctx context.Context, conn net.Conn)
	Close() error
}
```

### 2.TCP Server底层逻辑

TCP Server底层主要包含连接、处理、关闭步骤。

首先，定义一个结构体Config，用来描述TCP监听地址（后期可以添加一些监听过期时间等参数），在go语言中，习惯使用address:port表示地址，因此，这里只需要定义一个string类型的address即可。

其次，编写服务函数ListenAndServeWithSignal，这个函数能够监听地址，并为客户端提供相应的服务，在监听到操作系统返回的关闭信号，就可以关闭服务。

```go
import (
	"context"
	"github.com/hdt3213/godis/lib/logger"
	"goRedis/interface/tcp"
	"net"
	"os"
	"os/signal"
	"sync"
	"syscall"
)

// Config 启动TCP server的配置
type Config struct {
	// 监听的地址
	Address string
}

// ListenAndServeWithSignal 监听、服务、并且监听中断信号并通过 closeChan 通知服务器关闭
func ListenAndServeWithSignal(cfg *Config, handler tcp.Handler) error {

	// 使用一个空结构体，表示仅仅发送一个信号，表示关闭服务
	closeChan := make(chan struct{})
	// 传输操作系统的信号
	sigChan := make(chan os.Signal)
	// 该方法将监听后面SIGHUP，SIGQUIT，SIGTERM，SIGINT这几种情况，然后传递个sigChan
	signal.Notify(sigChan, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)

	// 该协程会从sigChan中除去判断
	go func() {
		signal := <-sigChan
		switch signal {
		case syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			closeChan <- struct{}{}
		}
	}()

	// 监听新连接
	listener, err := net.Listen("tcp", cfg.Address)
	if err != nil {
		return err
	}
	logger.Info("start listen")
	ListenAndServe(listener, handler, closeChan)
	return nil
}

// ListenAndServe 监听并提供服务，并在收到 closeChan 发来的关闭通知后关闭
func ListenAndServe(listener net.Listener, handler tcp.Handler, closeChan <-chan struct{}) {
	// 用户关闭情况，或者操作系统杀死进程时，非正常情况下的关闭连接和处理
	go func() {
		// 从closeChan中取出一个变量，该变量为空结构体，这个结构体起到信号的作用，如果没有读到变量，则该协程会阻塞在这里
		<-closeChan
		logger.Info("os send shutting down signal")
		logger.Info("server is shutting down")
		_ = listener.Close()
		_ = handler.Close()
	}()

	// 服务完毕，关闭连接和处理
	defer func() {
		_ = listener.Close()
		_ = handler.Close()
	}()
	// 拿到上下文
	ctx := context.Background()
	// 等待所有链接退出
	var waitDone sync.WaitGroup
	// 循环监听
	for true {
		// 查看新连接是否建立成功
		conn, err := listener.Accept()
		if err != nil {
			logger.Info("failed link")
			// 如果新连接接受出现问题，直接跳出即可
			break
		}
		logger.Info("accepted link")
		// 每次服务一个连接，等待队列就加1
		waitDone.Add(1)
		// 一个连接，开启一个协程去处理
		go func() {
			defer func() {
				// 服务完成就减1
				waitDone.Done()
			}()
			handler.Handler(ctx, conn)
		}()
	}
	// 服务完毕一个连接，等待队列就减1
	waitDone.Wait()
}
```

最后，实现Handler方法，即服务的详细操作，这里模拟一个Echo服务器，即客户端发送什么，服务端就回发什么。该方法目前用预测。

```go
package tcp

import (
	"bufio"
	"context"
	"github.com/hdt3213/godis/lib/logger"
	"goRedis/lib/sync/atomic"
	"goRedis/lib/sync/wait"
	"io"
	"net"
	"sync"
	"time"
)

// 该文件主要用来回复，即用户输入什么，服务端应答什么

// EchoClient 记录客户端信息
type EchoClient struct {
	Conn net.Conn
	// 添加了相互等待（超时等待）的waitGroup
	Waiting wait.Wait
}

// EchoHandler 用来记录有多少连接
type EchoHandler struct {
	activeConn sync.Map
	// 这里可能存在多线程并发问题，因此使用封装的原子bool类型
	closing atomic.Boolean
}

// Close 关闭单个客户端，给一个超时时间，这个超时时间过了之后，就关闭
func (e *EchoClient) Close() error {
	e.Waiting.WaitWithTimeout(10 * time.Second)
	_ = e.Conn.Close()
	return nil
}

// Handler EchoHandler 实现该方法
func (handler *EchoHandler) Handler(ctx context.Context, conn net.Conn) {
	if handler.closing.Get() {
		_ = conn.Close()
	}
	// 新来一个链接，将这个链接包装成client结构体，代表一个客户端
	client := &EchoClient{
		Conn: conn,
	}
	// 记录该客户端，这里模拟set，将传入的value写成空结构体即可
	handler.activeConn.Store(client, struct{}{})
	// 使用缓存区读取连接发送的数据
	reader := bufio.NewReader(conn)
	// 不断读取客户端发过来的数据
	for {
		// 使用 \n 进行读取
		msg, err := reader.ReadString('\n')
		if err != nil {
			if err == io.EOF {
				logger.Info("Connecting close")
				handler.activeConn.Delete(client)
			} else {
				logger.Warn(err)
			}
			return
		}
		// 将消息进行回发
		// 等待队列加1
		client.Waiting.Add(1)
		// 将读取的msg转换为切片
		b := []byte(msg)
		_, _ = conn.Write(b)
		client.Waiting.Done()
	}
}

// Close 关闭服务器
func (handler *EchoHandler) Close() error {
	logger.Info("handler shutting ")
	// 发送true，关闭服务端
	handler.closing.Set(true)
	// 将记录的客户端都关闭
	// sync下的map要使用Range()进行遍历
	handler.activeConn.Range(func(key, value any) bool {
		client := key.(*EchoClient)
		// 关闭连接
		_ = client.Conn.Close()
		// 然后返回true，将上面操作施加到map中的每个元素
		return true
	})
	return nil
}
```

main函数的编写

```go
package main

import (
	"fmt"
	"github.com/hdt3213/godis/lib/logger"
	"goRedis/config"
	"goRedis/tcp"
	"os"
)

// 设置读取的配置文件地址
const configFile string = "redis.conf"

// 设置默认的配置
var defaultProperties = &config.ServerProperties{
	Bind: "0.0.0.0",
	Port: 6379,
}

// 判断文件是否存在的函数
func fileExists(filename string) bool {
	info, err := os.Stat(filename)
	return err == nil && !info.IsDir()
}

func main() {
	// 设置日志格式
	logger.Setup(&logger.Settings{
		Path:       "logs",
		Name:       "godis",
		Ext:        "log",
		TimeFormat: "2022-10-01",
	})

	// 输入的文件不存在，就使用默认的配置
	if fileExists(configFile) {
		config.SetupConfig(configFile)
	} else {
		config.Properties = defaultProperties
	}

	err := tcp.ListenAndServeWithSignal(
		&tcp.Config{
			Address: fmt.Sprintf("%s:%d", config.Properties.Bind, config.Properties.Port),
		},
		tcp.MakeHandler())

	if err != nil {
		logger.Error(err)
	}

}
```

## 二、RESP（Redis协议解析器）

客户端和服务端之间通信协议。

### 1.常见协议

RESP有5中通信格式：正常回复、错误回复、整数、多行字符串、数组。

1. **正常回复**

以“+”开头，以“\r\n”结尾的字符串形式

```
+Ok\r\n
```

2. **错误回复**

以“-”开头，以“\r\n”结尾的字符串形式

```
-Error message\r\n
```

3. **整数**

以“：”开头，以“\r\n”结尾的字符串形式

```
:123456\r\n
```

4. **多行字符串**

以“$”开头，后面实际发送字节数，以“\r\n”结尾的字符串形式

```
$9\r\nimooc.com\r\n
```

5. **数组**

以“*”开头，后面跟成员个数

SET key value

```
*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
```

### 2.接口的定义

首先，定义一个resp的连接接口，主要包含写方法、查询某个数据库，选择要使用的某个数据库

```go
// Connection resp协议层的连接定义
type Connection interface {
   // Write 写方法
   Write([]byte) error
   // GetDBIndex 查询某个数据库，redis默认有16个DB
   GetDBIndex() int
   // SelectDB 选择要使用某个数据库
   SelectDB(int)
}
```

其次，定义Reply接口，主要用于将服务端返回的数据转换为字节。

```go
// Reply 各种服务端给客户端的回复
type Reply interface {
	// ToBytes 服务端返回的数据转换为字节
	ToBytes() []byte
}
```

#### （1）固定正常回复

接下，定义一些常量正常回复字符串

```go
package reply

type PongReply struct {
}

var pongBytes = []byte("+PONG\r\n")

// ToBytes 收到ping的指令，返回pang
func (p PongReply) ToBytes() []byte {
	return pongBytes
}

// MakePongReply 变成习惯，用于提供外界一个结构体
func MakePongReply() *PongReply {
	return &PongReply{}
}

type OkReply struct {
}

var okBytes = []byte("+OK\r\n")

func (o OkReply) ToBytes() []byte {
	return okBytes
}

var theOkReply = new(OkReply)

// MakeOkReply 向外界暴露
func MakeOkReply() *OkReply {
	return theOkReply
}

type NullBulkReply struct {
}

var nullBulkReplyBytes = []byte("$-1\r\n")

func (n NullBulkReply) ToBytes() []byte {
	return nullBulkReplyBytes
}

// MakeNullBulkReply 向外界暴露
func MakeNullBulkReply() *NullBulkReply {
	return &NullBulkReply{}
}

var emptyMultiBulkReplyBytes = []byte("*0\r\n")

type EmptyMultiBulkReply struct {
}

func (e EmptyMultiBulkReply) ToBytes() []byte {
	return emptyMultiBulkReplyBytes
}

// MakeEmptyMultiBulkReply 向外界暴露
func MakeEmptyMultiBulkReply() *EmptyMultiBulkReply {
	return &EmptyMultiBulkReply{}
}

type NoReply struct {
}

var noBytes = []byte("")

func (n NoReply) ToBytes() []byte {
	return noBytes
}
```

#### （2）动态正确、错误和自定义回复

接下来，定义一些发送正确的回复、错误的回复以及自定义的回复

```go
package reply

import (
	"bytes"
	"goRedis/interface/resp"
	"strconv"
)

// 该文件使服务端向客户端发送正确的回复、错误的回复以及自定义的回复

var (
	nullReplyBytes = []byte("$-1")
	CRLF           = "\r\n"
)

type BulkReply struct {
	Arg []byte
}

func (b *BulkReply) ToBytes() []byte {
	if len(b.Arg) == 0 {
		return nullReplyBytes
	}
	// 转换操作，例如将 moody 转换为 $5\r\nmoody\r\n
	return []byte("$" + strconv.Itoa(len(b.Arg)) + CRLF + string(b.Arg) + CRLF)
}

func MakeBulkReply(arg []byte) *BulkReply {
	return &BulkReply{Arg: arg}
}

type MultiBulkReply struct {
	Args [][]byte
}

func (m *MultiBulkReply) ToBytes() []byte {
	argLen := len(m.Args)
	var buf bytes.Buffer
	buf.WriteString("*" + strconv.Itoa(argLen) + CRLF)
	for _, arg := range m.Args {
		if arg == nil {
			buf.WriteString(string(nullReplyBytes) + CRLF)
		} else {
			buf.WriteString("$" + strconv.Itoa(len(arg)) + CRLF + string(arg) + CRLF)
		}
	}
	return buf.Bytes()
}

func MakeMultiBulkReply(arg [][]byte) *MultiBulkReply {
	return &MultiBulkReply{Args: arg}
}

type StatusReply struct {
	Status string
}

func (s *StatusReply) ToBytes() []byte {
	return []byte("+" + s.Status + CRLF)
}

func MakeStatusReply(status string) *StatusReply {
	return &StatusReply{
		Status: status,
	}
}

type IntReply struct {
	Code int64
}

func (i IntReply) ToBytes() []byte {
	return []byte(":" + strconv.FormatInt(i.Code, 10) + CRLF)
}

func MakeIntReply(code int64) *IntReply {
	return &IntReply{
		Code: code,
	}
}

type ErrorReply interface {
	// Error 返回错误字符串
	Error() string
	ToBytes() []byte
}

type StandardErrReply struct {
	Status string
}

func (s *StandardErrReply) Error() string {
	return s.Status
}

func (s *StandardErrReply) ToBytes() []byte {
	return []byte("-" + s.Status + CRLF)
}

func MakeErrReply(status string) *StandardErrReply {
	return &StandardErrReply{
		Status: status,
	}
}

func IsErrReply(reply resp.Reply) bool {
	return reply.ToBytes()[0] == '-'
}

```

#### （3）异常回复

定义一些异常回复

```go
package reply

type UnknownErrReply struct {
}

var unknownErrBytes = []byte("-Err unknown\r\n")

func (u UnknownErrReply) Error() string {
	return "-Err unknown"
}

func (u UnknownErrReply) ToBytes() []byte {
	return unknownErrBytes
}

// ArgNumErrReply 参数错误
type ArgNumErrReply struct {
	Cmd string
}

func (a *ArgNumErrReply) Error() string {
	return "ERR wrong number of arguments for '" + a.Cmd + "'command"
}

func (a *ArgNumErrReply) ToBytes() []byte {
	return []byte("-ERR wrong number of arguments for '" + a.Cmd + "'command\r\n")
}

func MakeArgNumErrReply(cmd string) *ArgNumErrReply {
	return &ArgNumErrReply{
		Cmd: cmd,
	}
}

// SyntaxErrReply 参数错误
type SyntaxErrReply struct {
}

var syntaxErrBytes = []byte("-Err syntax error\r\n")
var theSyntaxErrReply = &SyntaxErrReply{}

func (a *SyntaxErrReply) Error() string {
	return "ERR syntax error"
}

func (a *SyntaxErrReply) ToBytes() []byte {
	return syntaxErrBytes
}

func MakeSyntaxErrReply(cmd string) *SyntaxErrReply {
	return theSyntaxErrReply
}

// WrongTypeErrReply 参数错误
type WrongTypeErrReply struct {
}

var wrongTypeErrBytes = []byte("-WrongType Operation against a key holding the wrong kind of value\r\n")

func (a *WrongTypeErrReply) Error() string {
	return "WrongType Operation against a key holding the wrong kind of value"
}

func (a *WrongTypeErrReply) ToBytes() []byte {
	return wrongTypeErrBytes
}

// ProtocolErrReply 接口协议错误
type ProtocolErrReply struct {
	Msg string
}

func (a *ProtocolErrReply) Error() string {
	return "ERR Protocol error:'" + a.Msg
}

func (a *ProtocolErrReply) ToBytes() []byte {
	return []byte("-ERR Protocol error:'" + a.Msg + "'\r\n")
}
```

