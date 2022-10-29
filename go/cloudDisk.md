# Go实现分布式存储

## 一、本项目主要内容

- 基于Golang实现分布式文件上传服务
- 结合开源存储Ceph以及阿里OSS支持断点续传以及秒传功能
- 微服务以及容器化部署

## 二、文件上传的接口

| 接口描述             | 接口URL            |
| -------------------- | ------------------ |
| 文件上传接口         | POST /file/upload  |
| 文件查询接口         | GET /file/query    |
| 文件下载接口         | GET /file/download |
| 文件删除接口         | POST /file/delete  |
| 文件修改(重命名)接口 | POST /file/update  |

## 三、文件元信息的存储

### 1.元信息结构体

```go
package meta

import "sort"

// FileMeta 文件元信息结构
type FileMeta struct {
	// 文件hash值，可用作key
	FileSha1 string
	// 文件名称
	FileName string
	// 文件大小
	FileSize int64
	// 文件位置
	Location string
	// 上传时间
	UploadAt string
}

// 定义一个map，string - FileMeta 类型
var fileMetas map[string]FileMeta

func init() {
	fileMetas = make(map[string]FileMeta)
}

// UpdateFileMeta 更新fileMeta元信息
func UpdateFileMeta(fmeta FileMeta) {
	fileMetas[fmeta.FileSha1] = fmeta
}

// GetFileMeta 通过文件hash获取文件元信息
func GetFileMeta(fileSha1 string) FileMeta {
	return fileMetas[fileSha1]
}

// GetLastFileMetas 批量获取文件元信息列表
func GetLastFileMetas(count int) []FileMeta {
	fMetaArray := make([]FileMeta, len(fileMetas))
	for _, meta := range fMetaArray {
		fMetaArray = append(fMetaArray, meta)
	}

	sort.Sort(ByUploadTime(fMetaArray))
	return fMetaArray[0:count]
}

func ByUploadTime(array []FileMeta) sort.Interface {
	// TODO... 这里按照时间进行排序
	return nil
}

// RemoveFileMeta 删除文件元数据
func RemoveFileMeta(fileSha1 string) {
	// TODO... 这里可能存在线程安全的问题
	delete(fileMetas, fileSha1)
}
```

### 2.文件上传的步骤

- GET请求获取上传页面
- 选取本地文件，form形式上传文件
- 云端接受文件流，写入本地存储
- 云端更新文件云信息集合

```go
package handler

import (
	"file-store/meta"
	"file-store/util"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

// UploadHandler 文件上传
func UploadHandler(w http.ResponseWriter, r *http.Request) {

	// GET 方法获取上传页面
	if r.Method == "GET" {
		// 返回上传html页面
		data, err := os.ReadFile("../static/view/home.html")
		if err != nil {
			io.WriteString(w, "internal server error")
			return
		}
		io.WriteString(w, string(data))

	} else if r.Method == "POST" {
		// 接受文件流以及存储到本地目录，使用form形式上传文件
		file, header, err := r.FormFile("file")
		if err != nil {
			fmt.Printf("Failed to get data, err:%s\n", err.Error())
			return
		}
		defer file.Close()

		// 文件元数据的创建
		fileMeta := meta.FileMeta{
			FileName: header.Filename,
			Location: "/tmp/" + header.Filename,
			// Format中的时间格式是Go语言中特有的写法
			UploadAt: time.Now().Format("2006-01-02 15:04:05"),
		}

		// 传入文件存放地址，创建一个本地文件
		newFile, err := os.Create(fileMeta.Location)
		if err != nil {
			fmt.Printf("Failed to create file, err:%s\n", err.Error())
			return
		}
		defer newFile.Close()

		// 计算文件大小
		fileMeta.FileSize, err = io.Copy(newFile, file)
		if err != nil {
			fmt.Printf("Failed to save data into file, err:%s\n", err.Error())
			return
		}

		// 更新文件元信息
		newFile.Seek(0, 0)
		fileMeta.FileSha1 = util.FileSha1(newFile)
		meta.UpdateFileMeta(fileMeta)

		http.Redirect(w, r, "/file/upload/success", http.StatusFound)

	}

}

// UploadSuccessHandler 上传已完成
func UploadSuccessHandler(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "Upload finished!")
}
```

### 3.文件元信息查询逻辑

根据文件Hash值获取文件元信息

```go
// GetFileMetaHandler 获取文件元信息
func GetFileMetaHandler(w http.ResponseWriter, r *http.Request) {
	// 参数解析
    r.ParseForm()
	fileHash := r.Form["fileHash"][0]
	fileMeta := meta.GetFileMeta(fileHash)
	data, err := json.Marshal(fileMeta)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	w.Write(data)

}
```

### 4.文件下载接口

```go
// DownloadHandler 文件下载接口
func DownloadHandler(w http.ResponseWriter, r *http.Request) {
	// 参数解析
	r.ParseForm()
	// 获取文件hash
	fsha1 := r.Form.Get("filehash")
	// 获取文件meta
	fm := meta.GetFileMeta(fsha1)
	// 根据文件地址打开文件
	f, err := os.Open(fm.Location)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	defer f.Close()
	// 读取文件
	data, err := ioutil.ReadAll(f)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	//content-type 指示响应内容的格式
	//content-disposition 指示如何处理响应内容
	w.Header().Set("Content-Type", "application/octect-stream")
	w.Header().Set("Content-Disposition", "attachment;filename=\""+fm.FileName+"\"")
	w.Write(data)
}
```

### 5.文件元信息更新

```go

// FileMetaUpdataHandler 更新元信息接口(重命名)
func FileMetaUpdataHandler(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	// 获取文件操作数operation，例如，0表示重命名，1表示其他的更新操作
	opType := r.Form.Get("op")
	fileSha1 := r.Form.Get("filehash")

	// 获取新的文件名称
	newFileName := r.Form.Get("filename")

	if opType != "0" {
		w.WriteHeader(http.StatusForbidden)
		return
	}
	if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}
	// 根据文件hash获取文件元信息
	curFileMeta := meta.GetFileMeta(fileSha1)
	// 更改文件元信息的名称
	curFileMeta.FileName = newFileName
	// 更新
	meta.UpdateFileMeta(curFileMeta)

	w.WriteHeader(http.StatusOK)
	data, err := json.Marshal(curFileMeta)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusOK)
	w.Write(data)
}
```

### 6.文件删除

```go

// FileDeleteHandler 删除文件以及元数据
func FileDeleteHandler(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	fileSha1 := r.Form.Get("filehash")
	fileMeta := meta.GetFileMeta(fileSha1)
	os.Remove(fileMeta.Location)
	meta.RemoveFileMeta(fileSha1)
	w.WriteHeader(http.StatusOK)
}
```

### 7.主函数编写

```go
package main

import (
	"file-store/handler"
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/file/upload", handler.UploadHandler)
	http.HandleFunc("/file/upload/success", handler.UploadSuccessHandler)
	http.HandleFunc("/file/meta", handler.GetFileMetaHandler)
	http.HandleFunc("/file/query", handler.FileQueryHandler)
	http.HandleFunc("/file/download", handler.DownloadHandler)
	http.HandleFunc("/file/update", handler.FileMetaUpdataHandler)
	http.HandleFunc("/file/delete", handler.FileDeleteHandler)
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		fmt.Println("Failed to start server, err:%s", err.Error())
	}
}
```

## 四、MySQL存储

### 1.主从设置

首先，使用docker创建两个mysql数据库，端口分别为3306、3307，我们将3306设置为主节点、3307设置为从节点。

```sql
# 1. 主节点查看master等信息，记录下binlog等信息
show master status

# 2. 从节点，设置master信息
change master to master_host='主机ip',master_user='主节点用户名',master_password='主节点密码',master_log_file='步骤1中主节点binlog文件名称',master_log_pos=0;

# 3. 从节点，启动slave模式
start slave;

# 4. 从节点，查看是否配置成功
show slave status\G;
```

### 2.文件表的设计

```sql
--创建文件存储服务数据库，编码utf8mb4
CREATE DATABASE filestore_server DEFAULT CHARACTER SET utf8mb4

--创建文件表
DROP TABLE IF EXISTS `tbl_file`;
CREATE TABLE `tbl_file`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `file_hash` char(40) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '文件hash，sha1',
  `file_name` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '文件名',
  `file_size` bigint(20) NOT NULL COMMENT '文件字节数',
  `file_addr` varchar(1024) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '文件存储位置',
  `create_at` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建日期',
  `update_at` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '修改日期',
  `status` int(20) UNSIGNED NOT NULL DEFAULT 0 COMMENT '可用 禁用 删除',
  `ext1` int(11) UNSIGNED NOT NULL DEFAULT 0 COMMENT '备用字段',
  `ext2` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL COMMENT '备用字段2',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uk_file_hash`(`file_hash`) USING BTREE,
  INDEX `idx_status`(`status`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;


--创建用户表
DROP TABLE IF EXISTS `tbl_user`;
CREATE TABLE `tbl_user`  (
  `id` int(11) NOT NULL,
  `user_name` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '用户名',
  `user_pwd` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '加密后的用户密码',
  `email` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '用户邮箱',
  `phone` char(11) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '手机号',
  `email_validated` tinyint(1) UNSIGNED NOT NULL COMMENT '电子邮箱是否经过验证',
  `phone_validated` tinyint(1) UNSIGNED NOT NULL COMMENT '手机号是否经过验证',
  `signup_at` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册日期',
  `last_active` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '最后活跃日期',
  `profile` text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '用户属性',
  `status` int(11) UNSIGNED NOT NULL DEFAULT 0 COMMENT '状态（启用、禁用、锁定、删除）',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uk_phone`(`phone`) USING BTREE,
  INDEX `idx_status`(`status`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

--创建用户token表
DROP TABLE IF EXISTS `tbl_user_token`;
CREATE TABLE `tbl_user_token`  (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_name` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '用户名',
  `user_token` char(40) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '用户token',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uk_user_name`(`user_name`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;


-- 创建用户文件表
CREATE TABLE `tbl_user_file` (
  `id` int(11) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `user_name` varchar(64) NOT NULL COMMENT '用户名',,
  `file_sha1` varchar(64) NOT NULL DEFAULT '' COMMENT '文件hash，去文件表中查询文件元信息',
  `file_size` bigint(20) DEFAULT '0' COMMENT '文件大小',
  `file_name` varchar(256) NOT NULL DEFAULT '' COMMENT '文件名',
  `upload_at` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '上传时间',
  `last_update` datetime DEFAULT CURRENT_TIMESTAMP
          ON UPDATE CURRENT_TIMESTAMP COMMENT '最后修改时间',
  `status` int(11) NOT NULL DEFAULT '0' COMMENT '文件状态(0正常1已删除2禁用)',
  UNIQUE KEY `idx_user_file` (`user_name`, `file_sha1`),
  KEY `idx_status` (`status`),
  KEY `idx_user_id` (`user_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

--创建用户文件表
DROP TABLE IF EXISTS `tbl_user_file`;
CREATE TABLE `tbl_user_file`  (
  `id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_name` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '用户名',
  `file_hash` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '文件hash',
  `file_size` bigint(20) UNSIGNED NOT NULL COMMENT '文件大小',
  `file_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '文件名',
  `upload_at` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '上传时间',
  `last_update` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '最后修改时间',
  `status` int(11) UNSIGNED NOT NULL DEFAULT 0 COMMENT '文件状态(0正常1已删除2禁用)',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uk_user_file`(`user_name`, `file_hash`) USING BTREE,
  INDEX `idx_status`(`status`) USING BTREE,
  INDEX `idx_user_name`(`user_name`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;
```

### 3.分库分表

- 水平分表

  假设分成256张文件表，按文件Sha1值后两位来切分，则以：tbl_${file_sha1}[:-2]的规则到对应表进行存取。

  缺点，要更改分成的文件表数，需要重新进行hash

### 4.连接MySQL

```go
package mysql

import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
	"log"
)

var db *sql.DB
var err error

func init() {
	//注意，不要在这里用短变量声明，否则赋值的根本不是外部的db变量
	db, err = sql.Open("mysql", "root:123456@tcp(127..0.0.1:3306)/fileserver?charset=utf8mb4")
	if err != nil {
		log.Fatalf("初始化数据库连接失败：%v", err.Error())
	}
	db.SetMaxOpenConns(2000)
	err = db.Ping()
	if err != nil {
		log.Fatalf("连接数据库失败：%v", err.Error())
	}
	log.Println("连接数据库成功")
}

// DBConn 获取连接对象
func DBConn() *sql.DB {
	return db
}
```

### 5.文件表操作

```go
package db

import (
	"database/sql"
	"file-store/db/entities"
	"file-store/db/mysql"
	"log"
)

//处理文件相关数据库操作

// AddFileInfo 增加文件信息到文件表
func AddFileInfo(f *entities.AddFileEntity) (result bool, err error) {
	stmtIn, err := mysql.DBConn().Prepare("INSERT IGNORE INTO tbl_file(file_hash,file_size,file_name,file_addr) VALUES(?,?,?,?)")
	if err != nil {
		log.Println("AddFileInfo预编译失败：" + err.Error())
		return
	}
	defer stmtIn.Close()
	// 执行sql语句，并将实体的属性与所写的sql语句一一对应起来
	r, err := stmtIn.Exec(f.FileHash, f.FileSize, f.FileName, f.FileAddr)
	if err != nil {
		log.Println("AddFileInfo执行失败：" + err.Error())
		return
	}

	// 判断是否添加成功
	if rows, err := r.RowsAffected(); err == nil {
		if rows <= 0 {
			//由于sql采用了ignore，因此若err为nil且rows<=0，则一定是filehash重复导致的插入失败
			//tbl_file表，filehash列具有唯一索引
			log.Printf("filehash:%v 已被上传过", f.FileHash)
		}
		return true, nil
	}
	return false, err
}

// GetFileMeta 根据filehash获取文件信息
func GetFileMeta(filehash string) (fileEntity *entities.GetFileEntity, err error) {
	stmtOut, err := mysql.DBConn().Prepare("SELECT file_hash,file_name,file_size,file_addr,create_at FROM tbl_file WHERE file_hash=? and status=1")
	if err != nil {
		return
	}
	defer stmtOut.Close()
	fileEntity = &entities.GetFileEntity{}
	err = stmtOut.QueryRow(filehash).Scan(&fileEntity.FileHash, &fileEntity.FileName, &fileEntity.FileSize, &fileEntity.FileAddr, &fileEntity.UpdateAt)
	if err == nil {
		return fileEntity, nil
	}

	//如果未匹配到数据，则不是错误
	if err == sql.ErrNoRows {
		return nil, nil
	}
	return nil, err
}

// UpdateFileLocation 更新本地文件
func UpdateFileLocation(filehash string, location string) error {
	stmtIn, err := mysql.DBConn().Prepare("UPDATE tbl_file SET file_addr=? WHERE file_hash=?")
	if err != nil {
		log.Println("UpdateFileLocation预编译失败：" + err.Error())
		return err
	}
	defer stmtIn.Close()
	r, err := stmtIn.Exec(location, filehash)
	if err != nil {
		log.Println("UpdateFileLocation执行失败：" + err.Error())
		return err
	}

	_, err = r.RowsAffected()
	if err != nil {
		return err
	}
	return nil
}
```

## 五、秒传实现

### 1.文件校验值计算

校验算法类型：CRC、MD5、SHA1

对比表

| 类型          | 校验值长度(字节) | 校验值类别 | 安全级别(校验值碰撞概率) | 计算效率 | 应用场景           |
| ------------- | ---------------- | ---------- | ------------------------ | -------- | ------------------ |
| CRC(16/32/64) | 2/4/8            | CRC校验码  | 弱                       | 高       | 用于传输数据校验   |
| MD5           | 16               | Hash值     | 中                       | 中       | 文件校验和数据签名 |
| SHA1          | 20               | Hash值     | 高                       | 低       | 文件校验和数据签名 |

### 2.秒传原理

场景：

- 用户上传大文件时，如果有用户已经完成上传过大文件，那么该用户可使用秒传大文件
- 离线下载，将资源先保存到自己的网盘中，然后使用网盘进行下载，以达到高速下载的目的

实现的关键点：

- 要记录每个文件的Hash值
- 用户与文件的关联，同一文件 

上传文件流程与秒传：

1. 用户发起文件上传
2. 服务器比对文件Hash值，如果没有查到相同Hash值，则保存文件到存储，使用Hash计算微服务对文件进行Hash计算，将文件元信息和Hash值写入到唯一文件表，再将用户和文件元信息写入到用户文件表中

3. 查到有相同的Hash值，则表示触发秒传，直接将该文件元信息和用户进行关联，保存到用户文件表中，注意，这里有可能存在文件名不同，但是文件内容相同，因此保存到用户文件表中，存在hash值相同，但是文件名不同的情况。

### 3.用户上传文件进行绑定

Dao层代码：

```go
package db

import (
	mydb "file-store/db/mysql"
	"fmt"
	"time"
)

// UserFile : 用户文件表结构体
type UserFile struct {
	UserName    string
	FileHash    string
	FileName    string
	FileSize    int64
	UploadAt    string
	LastUpdated string
}

// OnUserFileUploadFinished : 更新用户文件表
func OnUserFileUploadFinished(username, filehash, filename string, filesize int64) bool {
	stmt, err := mydb.DBConn().Prepare(
		"insert ignore into tbl_user_file (`user_name`,`file_sha1`,`file_name`," +
			"`file_size`,`upload_at`) values (?,?,?,?,?)")
	if err != nil {
		return false
	}
	defer stmt.Close()

	_, err = stmt.Exec(username, filehash, filename, filesize, time.Now())
	if err != nil {
		return false
	}
	return true
}

// QueryUserFileMetas : 批量获取用户文件信息
func QueryUserFileMetas(username string, limit int) ([]UserFile, error) {
	stmt, err := mydb.DBConn().Prepare(
		"select file_sha1,file_name,file_size,upload_at," +
			"last_update from tbl_user_file where user_name=? limit ?")
	if err != nil {
		return nil, err
	}
	defer stmt.Close()

	rows, err := stmt.Query(username, limit)
	if err != nil {
		return nil, err
	}

	var userFiles []UserFile
	for rows.Next() {
		ufile := UserFile{}
		err = rows.Scan(&ufile.FileHash, &ufile.FileName, &ufile.FileSize,
			&ufile.UploadAt, &ufile.LastUpdated)
		if err != nil {
			fmt.Println(err.Error())
			break
		}
		userFiles = append(userFiles, ufile)
	}
	return userFiles, nil
}
```

业务逻辑：

```go
//将用户和文件元信息保存到用户文件表中
r.ParseForm()
username := r.Form.Get("username")
suc := dblayer.OnUserFileUploadFinished(username, fileMeta.FileHash, fileMeta.FileName, fileMeta.FileSize)
if suc {
    http.Redirect(w, r, "/static/view/home.html", http.StatusFound)
} else {
    w.Write([]byte("Upload Failed!"))
}
```

### 4.秒传接口

步骤：

- 解析请求参数
- 从文件表中查询相同hash的文件记录
- 查不到记录则返回秒传失败
- 上传过则将文件信息写入用户文件表，返回成功

```go
// TryFastUploadHandler 尝试秒传接口
func TryFastUploadHandler(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	//1.解析请求参数
	username := r.Form.Get("username")
	filehash := r.Form.Get("filehash")
	filename := r.Form.Get("filename")
	filesize, _ := strconv.Atoi(r.Form.Get("filesize"))

	//2.从文件表中查询相同hash的文件记录
	fileMeta, err := meta.GetFileMetaDB(filehash)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//3.查不到记录则返回妙传失败
	if fileMeta == nil {
		resp := util.RespMsg{
			Code: -1,
			Msg:  "秒传失败，请访问普通上传接口",
		}
		w.Write(resp.JSONBytes())
		return
	}

	//4.上传过则将文件信息写入用户文件表，返回成功
	suc := dblayer.OnUserFileUploadFinished(username, filehash, filename, int64(filesize))
	if suc {
		resp := util.RespMsg{
			Code: 0,
			Msg:  "秒传成功",
		}
		w.Write(resp.JSONBytes())
		return
	} else {
		resp := util.RespMsg{
			Code: -2,
			Msg:  "秒传失败，请稍后重试",
		}
		w.Write(resp.JSONBytes())
		return
	}
}
```

### 5.场景问题

假如，当前有多个用户，同时上传同一个文件，如何解决?

思路：允许不同用户同时上传同一个文件，先完成上传的先入库，后上传的只更新用户文件表，并删除以上传的文件

## 六、分块上传和断点续传

### 1.分块上传

客户端上传文件时，将文件切分多块，独立传输，上传完成后进行合并。小文件不建议分块上传。可以并行上传分块，并且可以无序上传。分块上传能够极大提高上传效率。减少传输失败后重试的流量以及时间。

### 2.断点续传

传输暂停或异常中断后，可基于原来进度重传（百度网盘中）

### 3.上传流程

用户发起上传，服务端切分文件，并行上传文件分块，通知上传完成，后续合并文件。

用户可以取消上传，和查询上传状态。

### 4.分块上传通用接口

```go
//分块上传接口
// 初始化分块信息
http.HandleFunc("/file/mpupload/init",handler.AccessAuth(handler.InitialMultipartUploadHandler))
// 上传分块
http.HandleFunc("/file/mpupload/uppart",handler.AccessAuth(handler.UploadPartHandler))
// 通知分块上传完成
http.HandleFunc("/file/mpupload/complete",handler.AccessAuth(handler.CompleteUploadHandler))
// 取消上传分块
http.HandleFunc("/file/mpupload/cancel",handler.AccessAuth(handler.CancelUploadPartHandler))
// 查看分块上传的整体状态
http.HandleFunc("/file/mpupload/status",handler.AccessAuth(handler.MultipartUploadStatusHandler))
```

### 5.分块结构体

```go
// MultipartUploadInfo : 初始化信息
type MultipartUploadInfo struct {
	FileHash string
	FileSize int
	// 上传唯一标志，每次上传都有不同的id
	UploadID string
	// 分块大小，最后一块小于或等于规定的大小
	ChunkSize int
	// 分块数量
	ChunkCount int
}
```

### 6.上传初始化

```go
//1.判断是否已经上传过
//2.生成唯一上传ID
//3.缓存分块初始化信息
// InitialMultipartUploadHandler : 初始化分块上传
func InitialMultipartUploadHandler(w http.ResponseWriter, r *http.Request) {
	// 1. 解析用户请求参数
	r.ParseForm()
	username := r.Form.Get("username")
	filehash := r.Form.Get("filehash")
	filesize, err := strconv.Atoi(r.Form.Get("filesize"))
	if err != nil {
		w.Write(util.NewRespMsg(-1, "params invalid", nil).JSONBytes())
		return
	}

	// 2. 获得redis的一个连接
	rConn := rPool.RedisPool().Get()
	defer rConn.Close()

	// 3. 生成分块上传的初始化信息
	upInfo := MultipartUploadInfo{
		FileHash:   filehash,
		FileSize:   filesize,
		UploadID:   username + fmt.Sprintf("%x", time.Now().UnixNano()),
		ChunkSize:  5 * 1024 * 1024, // 5MB
		ChunkCount: int(math.Ceil(float64(filesize) / (5 * 1024 * 1024))),
	}

	// 4. 将初始化信息写入到redis缓存
    
	rConn.Do("HSET", "MP_"+upInfo.UploadID, "chunkcount", upInfo.ChunkCount)
	rConn.Do("HSET", "MP_"+upInfo.UploadID, "filehash", upInfo.FileHash)
	rConn.Do("HSET", "MP_"+upInfo.UploadID, "filesize", upInfo.FileSize)

	// 5. 将响应初始化数据返回到客户端
	w.Write(util.NewRespMsg(0, "OK", upInfo).JSONBytes())
}
```

### 7.上传分块

```go
// UploadPartHandler : 上传文件分块
func UploadPartHandler(w http.ResponseWriter, r *http.Request) {
	// 1. 解析用户请求参数
	r.ParseForm()
	//	username := r.Form.Get("username")
	uploadID := r.Form.Get("uploadid")
	chunkIndex := r.Form.Get("index")

	// 2. 获得redis连接池中的一个连接
	rConn := rPool.RedisPool().Get()
	defer rConn.Close()

	// 3. 获得文件句柄，用于存储分块内容
	fpath := "/data/" + uploadID + "/" + chunkIndex
	os.MkdirAll(path.Dir(fpath), 0744)
	fd, err := os.Create(fpath)
	if err != nil {
		w.Write(util.NewRespMsg(-1, "Upload part failed", nil).JSONBytes())
		return
	}
	defer fd.Close()

	buf := make([]byte, 1024*1024)
	for {
		n, err := r.Body.Read(buf)
		fd.Write(buf[:n])
		if err != nil {
			break
		}
	}

	// 4. 更新redis缓存状态
	rConn.Do("HSET", "MP_"+uploadID, "chkidx_"+chunkIndex, 1)

	// 5. 返回处理结果到客户端
	w.Write(util.NewRespMsg(0, "OK", nil).JSONBytes())
}
```

### 8.通知上传合并

```go

// CompleteUploadHandler : 通知上传合并
func CompleteUploadHandler(w http.ResponseWriter, r *http.Request) {
	// 1. 解析请求参数
	r.ParseForm()
	upid := r.Form.Get("uploadid")
	username := r.Form.Get("username")
	filehash := r.Form.Get("filehash")
	filesize := r.Form.Get("filesize")
	filename := r.Form.Get("filename")

	// 2. 获得redis连接池中的一个连接
	rConn := rPool.RedisPool().Get()
	defer rConn.Close()

	// 3. 通过uploadid查询redis并判断是否所有分块上传完成
	data, err := redis.Values(rConn.Do("HGETALL", "MP_"+upid))
	if err != nil {
		w.Write(util.NewRespMsg(-1, "complete upload failed", nil).JSONBytes())
		return
	}
	totalCount := 0
	chunkCount := 0
	// 判断是否所有分块都已经上传
	// 这里的i + 2是表示 data 和 err 占用两个位置，因此每次要取出一个data，就应该跳两个
	for i := 0; i < len(data); i += 2 {
		k := string(data[i].([]byte))
		v := string(data[i+1].([]byte))
		if k == "chunkcount" {
			totalCount, _ = strconv.Atoi(v)
		} else if strings.HasPrefix(k, "chkidx_") && v == "1" {
			chunkCount++
		}
	}
	if totalCount != chunkCount {
		w.Write(util.NewRespMsg(-2, "invalid request", nil).JSONBytes())
		return
	}

	// 4. TODO：合并分块

	// 5. 更新唯一文件表及用户文件表
	fsize, _ := strconv.Atoi(filesize)
	dblayer.OnFileUploadFinished(filehash, filename, int64(fsize), "")
	dblayer.OnUserFileUploadFinished(username, filehash, filename, int64(fsize))

	// 6. 响应处理结果
	w.Write(util.NewRespMsg(0, "OK", nil).JSONBytes())
}
```

## 七、接入Ceph

### 1.Ceph的特点

这个和MinIO差不多。部署简单、可靠性强、性能高、开源、分布式、可扩展性强。

### 2.Ceph基础组件

| 名称    | 解释                                                         |
| ------- | ------------------------------------------------------------ |
| OSD     | 用于集群中所有数据与对象的存储，存储/复制/平衡/恢复数据等等  |
| Monitor | 监控集群状态，维护cluster MAP表，保证集群数据一致性          |
| MDS     | 保存文件系统服务的元数据                                     |
| RGW     | 提供与亚马逊S3和Swift兼容的RESTfu了API的gateway服务          |
| MGR     | 用于收集ceph集群状态、运行指标，比如存储利用率、当前性能指标和系统负载。对外提供ceph dashboard和restful 阿皮。 |

### 3.Docker搭建Ceph分布式

| 主机名称 | 主机IP          | 说明            |
| -------- | --------------- | --------------- |
| ceph1    | 192.168.161.135 | mon、osd        |
| ceph2    | 192.168.161.136 | mon、mgr、osd   |
| ceph3    | 192.168.161.137 | mon、osd        |
| manager  | 192.168.56.110  | ceph、rbd客户端 |

在三台服务器ceph1\2\3上分别创建Monitor配置文件路径：

```
mkdir -p /etc/ceph/ /var/lib/ceph
```

在三台服务器上创建monitor，命令如下：

```bash
docker run -d --net=host --name=mon -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph -e MON_IP=192.168.161.135 -e CEPH_PUBLIC_NETWORK=192.168.161.0/24 ceph/daemon mon
```

在三台服务器上创建osd，命令如下：

```bash
docker run -d --net=host --name=osd --privileged=true -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var /lib/ceph -v /dev/:/dev/ -v /ceph-rbd:/var/lib/ceph/osd ceph/daemon osd_directory
```

在ceph2上创建mgr

```bash
docker run -d --net=host --name=mgr -v etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph ceph/daemon mgrl
```

设置权限

```bash
docker exec -it gwnode radosgw-admin user create --uid=user1 --display-name=user1
```

### 4.将文件写入ceph中

```go
// 同时将文件写入ceph存储
newFile.Seek(0, 0)
data, _ := ioutil.ReadAll(newFile)
bucket := ceph.GetCephBucket("userfile")
cephPath := "/ceph/" + fileMeta.FileHash
_ = bucket.Put(cephPath, data, "octet-stream", s3.PublicRead)
fileMeta.Location = cephPath
```

## 八、RabbitMQ与异步任务

### 1.定义rabbitmq配置

```go
package config

const (
	// AsyncTransferEnable : 是否开启文件异步转移(默认同步)
	AsyncTransferEnable = false
	// TransExchangeName : 用于文件transfer的交换机
	TransExchangeName = "uploadserver.trans"
	// TransOSSQueueName : oss转移队列名
	TransOSSQueueName = "uploadserver.trans.oss"
	// TransOSSErrQueueName : oss转移失败后写入另一个队列的队列名
	TransOSSErrQueueName = "uploadserver.trans.oss.err"
	// TransOSSRoutingKey : routingkey
	TransOSSRoutingKey = "oss"
)

var (
	// RabbitURL : rabbitmq服务的入口url
	RabbitURL = "amqp://guest:guest@127.0.0.1:5672/"
)
```

### 2.定义消息载体的结构

```go
package mq

import "file-store/common"

// 消息载体
type TransferData struct {
	FileHash      string
	CurLocation   string
	DestLocation  string
	DestStoreType common.StoreType
}
```

### 3.消息发送——生产者

```go
package mq

import (
	"file-store/config"
	"github.com/streadway/amqp"
	"log"
)

var conn *amqp.Connection
var channel *amqp.Channel

// initChannel 初始化一个channel
func initChannel() bool {

	// 1. 判断channel是否已经创建过
	if channel != nil {
		return true
	}

	// 2. 获得rabbitmq的一个连接
	conn, err := amqp.Dial(config.RabbitURL)
	if err != nil {
		log.Println(err.Error())
		return false
	}

	// 3. 打开一个channel，用于消息的发布与接收等
	channel, err = conn.Channel()
	if err != nil {
		log.Println(err.Error())
		return false
	}

	return true
}

// Publish 发布消息
func Publish(exchange, routingKey string, msg []byte) bool {
	//1. 判断channel是否是正常的
	if !initChannel() {
		return false
	}
	// 2. 执行消息的发布动作
	err := channel.Publish(
		exchange,
		routingKey,
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        msg,
		},
	)

	if err != nil {
		log.Println(err.Error())
		return false
	}
	return true
}
```

### 4.消息接受——消费者

```go
package mq

import "log"

var done chan bool

// StartConsume 开始监听队列，获取消息
func StartConsume(qName, cName string, callback func(msg []byte) bool) {

	// 1. 通过channel.Consumer获得消息信道
	msgs, err := channel.Consume(
		qName,
		cName,
		true,
		false,
		false,
		false,
		nil,
	)

	if err != nil {
		log.Println(err.Error())
		return
	}

	// 2. 循环获取队列的消息
	done = make(chan bool)

	go func() {
		for msg := range msgs {
			// 3. 调用callback方法处理新的消息
			processSuc := callback(msg.Body)
			if !processSuc {
				// 将任务写道另一个队列中，用于异常情况的重试

			}
		}
	}()

	// done没有新的消息过来，会一直阻塞
	<-done
	// 关闭rabbitMQ通道
	channel.Close()
}
```

## 项目中遇到的小问题

### 1.使用sort.Sort()切片排序

```go
import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

// PersonSlice 按照 Person.Age 从大到小排序
type PersonSlice []Person

// Len 重写 Len() 方法
func (p PersonSlice) Len() int {
	return len(p)
}

// Swap 重写 Swap() 方法
func (p PersonSlice) Swap(i, j int) {
	p[i], p[j] = p[j], p[i]
}

// Less 重写 Less() 方法
// 使用不同字段进行对比，并依此排序，这里用了Age，可以更换成CreatedAt进行排序的
// 使用 < 从大到小排序，使用 > 从小到大排序
func (p PersonSlice) Less(i, j int) bool {
	return p[j].Age < p[i].Age
}

func main() {
	people1 := Person{
		Name: "牛奔1",
		Age:  18,
	}
	people2 := Person{
		Name: "牛奔2",
		Age:  19,
	}
	people3 := Person{
		Name: "牛奔3",
		Age:  20,
	}

	peopleSlice := PersonSlice{}
	peopleSlice = append(peopleSlice, people2, people1, people3)

	fmt.Println(peopleSlice)

	sort.Sort(peopleSlice) // 按照Age的逆序排序
	fmt.Println(peopleSlice)

	sort.Sort(sort.Reverse(peopleSlice)) // 按照Age的升序排序
	fmt.Println(peopleSlice)
}
```

### 2.使用sort.slice()切片排序

```go
var heros []Hero

hero := Hero{"李白", "Assassin", 13}
h := Hero{"妲己", "Mage", 15}
h2 := Hero{"貂蝉", "Assassin", 9}
h3 := Hero{"关羽", "Tank", 31}
h4 := Hero{"诸葛亮", "Mage", 21}

heros = append(heros, hero, h, h2, h3, h4)
// 闭包
sort.Slice(heros, func(i, j int) bool {
    return heros[i].age < heros[j].age
})
```

