# 多端全栈项目总结

## 第一章笔记

假设司机定向接单地址为北京火车站，但实际上不是说代驾订单的终点必须是北京火车站。2公里以内的都可以接单。

## 第二章笔记

微服务鉴权设计思想，BFF鉴权，就是设计后端微服务接口时，考虑到不同设备的需求，为不同的设备提供不同接口。客户端不是直接访问服务器公共接口，而是调用BFF层提供的接口，BFF层再调用基础的服务，不同客户端拥有不同的BFF层，它们定制客户端需要的接口。有了BFF层，客户端只需发起一次HTTP请求，BFF层就能够调用不同的服务，然后把汇总后的数据返回给客户端，只需要在不同客户端的BFF层的web方法上添加权限，就能够达到权限控制。

## 第三章笔记

### 一、使用了哪些云服务

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

### 二、项目构成
#### 1. 后端系统构成
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
#### 2. 前端系统构成
|序号|客户端|端口号|具体用途|
|--|--|--|--|
|1|hxds-driver-wx|–|司机端小程序|
|2|hxds-customer-wx|–|客户端小程序|
|3|hxds-mis-vue|3000|MIS系统前端项目|

### 三、技术栈
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

### 四、司机微服务用户注册功能

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

```sql
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
### 五、司机bff层用户注册功能

第四章所写的内容都是司机子系统中的业务逻辑，但是在本系统中采用的微服务方式，且使用bff层对不同客户端进行鉴权，因此，bff层收到客户端发送的请求后，去调用司机子系统的业务逻辑，而这一调用过程需要使用到feign接口。

#### 1.创建feign接口

在bff中声明一个接口，并使用@FeignClient(value = "")指明要调用的子系统，然后在接口中定义要调用的方法已经请求地址。然后定义service接口、serviceImpl实现类、以及controller类。

```java
// 定义feign接口
@FeignClient(value = "hxds-dr")
public interface DrServiceApi {

    @PostMapping("/driver/registerNewDriver")
    public R registerNewDriver(RegisterNewDriverForm form);
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
```

### 六、腾讯云文件存储服务

#### 1.开通腾讯云对象存储服务

可领取半年免费存储。首先，先创建密钥，主要记录APPID、SecretId、Secret Key。创建各种存储桶，设置不同类型的桶保存自己系统中不同的数据。存储桶可以设置访问用户。系统配置：首先在yml文件中设置如下内容。

```
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
```
<dependency>
       <groupId>com.qcloud</groupId>
       <artifactId>cos_api</artifactId>
       <version>5.6.89</version>
</dependency>
```
#### 2.开通腾讯数据万象服务（主要用来使用AI监测上传的图片是否存在暴力、色情）

注意：免费两个月，后期建议购买按量付费

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
        COSCredentials cred = new BasicCOSCredentials(secretId, secretKey);
        ClientConfig clientConfig = new ClientConfig(new Region(regionPublic));
        clientConfig.setHttpProtocol(HttpProtocol.https);
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
        String fileName = file.getOriginalFilename(); //上传文件的名字
        String fileType = fileName.substring(fileName.lastIndexOf(".")); //文件后缀名
        path += IdUtil.simpleUUID() + fileType; //避免重名图片在云端覆盖，所以用UUID作为文件名

        //元数据信息
        ObjectMetadata meta = new ObjectMetadata();
        meta.setContentLength(file.getSize());
        meta.setContentEncoding("UTF-8");
        meta.setContentType(file.getContentType());

        //向存储桶中保存文件
        PutObjectRequest putObjectRequest = new PutObjectRequest(bucketPublic, path, file.getInputStream(), meta);
        putObjectRequest.setStorageClass(StorageClass.Standard); //标准存储
        COSClient client = getCosPublicClient();
        PutObjectResult putObjectResult = client.putObject(putObjectRequest);

        //合成外网访问地址
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



### 八、开通腾讯人脸识别活体检测

腾讯云人脸识别既能够进行面部识别，又能够进行活体检测。在腾讯云人脸识别工具上新建一个人员库。调用该库的API接口如下：

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
### 九、司机小程序端工作台

司机首页工作台要显示的信息非常多，例如代驾时长、今日收入、今日成单。在订单表中查询当天代驾总时长、总收入和订单数。

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


## 第四章笔记

### 一、开通腾讯位置服务

在腾讯云控制面板上，你去创建一个地图应用，名字可以随便定义，类型选择“出行”，然后就能拿到地图应用的密钥了。拿到密钥，点击编辑授权自己的微信小程序APP ID。用户小程序端需要将起点和重点的经纬度传递给后端。传递的form表单如下所示：

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
        int status = json.getInt("status");
        String message = json.getStr("message");
        System.out.println(message);
        if (status != 0) {
            log.error(message);
            throw new HxdsException("预估里程异常：" + message);
        }
        JSONArray rows = json.getJSONObject("result").getJSONArray("rows");
        JSONObject element = rows.get(0, JSONObject.class).getJSONArray("elements").get(0, JSONObject.class);
        int distance = element.getInt("distance");
        String mileage = new BigDecimal(distance).divide(new BigDecimal(1000)).toString();
        int duration = element.getInt("duration");
        String temp = new BigDecimal(duration).divide(new BigDecimal(60), 0, RoundingMode.CEILING).toString();
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

#### 3.开启GPS实时定位

有两种开启小程序实时定位的方法：startLocationUpdate和startLocationUpdateBackground。其中后者是在小程序被挂到后台，也能获取实时定位。这个我们不太需要，我们只需要前者就可以了，只要小程序正常使用就能获得实时定位，挂到后台就不获取定位，这样还能节省手机电量。在App.vue页面中定义JS代码，开启实时获得GPS定位。因为小程序启动之后会运行onLaunch()函数，所以我们就把开启实时定位的代码写到onLaunch()函数里面。


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

小程序每次获得最新GPS定位坐标之后，都会执行onLocationChange()函数，然后我们就能拿到最新的经纬度坐标。但是我们想要把坐标数据传递给工作台页面，很多人会想到Storage机制。但是Storage在频繁读写的条件下会丢失数据，特别是小程序一秒钟会获得很多GPS定位，读写Storage很多次，必然会丢失数据，所以千万不能使用Storage机制。

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


```js
returnLocationHandle: function() {
    this.map.moveToLocation();
}
```

### 二、乘客下单

乘客提交创建订单的请求之后，需要后端Java程序重新计算最佳线路、里程和时间。这是为了避免有人用POSTMAN模拟客户端，随便输入代驾的起点、终点、里程和时长，所以后端程序必须重新计算这些数据，并且写入到订单记录中。

由于每笔代驾都要记录订单的预估金额，所以我们要根据里程和计费规则，预估出代驾的费用，这就需要用到规则引擎。

为什么要使用规则引擎？

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

#### 1.乘客下单

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

业务代码的编写


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

#### 2.司机接单

创建订单后，要寻找附近适合接单的司机。司机端的小程序实时将司机的GPS定位上传，然后将定位信息缓存到Redis中，利用Redis的Geo功能（主要用于存储地理位置信息，并对存储信息进行操作），GEO是redis3.2以后才有的功能。可使用如下命令添加地理位置信息和查询多少半径内的地理内容。

```
GEOADD china 116.403963 39.915119 tiananmen 116.417876 39.915411 wangfujing 116.404354 39.904748 qianmen
GEORADIUS china 116.4000 39.9000 1 km WITHDIST
```
Redis的GEO命令可以帮我们提取出某个坐标点指定距离以内的景点，如果Redis里面缓存的是司机的定位信息，那么我们用代驾单的起点坐标来查询附近几公里以内的司机，有个需要特别注意的问题就是Geo中的定位我们没办法设置超时时间，所以我们想要知道哪些司机上线中，就得额外创建带有超时的缓存。

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
        //封装成Point对象才能缓存到Redis里面
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
        Point point = new Point(startPlaceLongitude, startPlaceLatitude);
        //设置GEO距离单位为千米
        Metric metric = RedisGeoCommands.DistanceUnit.KILOMETERS;
        Distance distance = new Distance(5, metric);
        Circle circle = new Circle(point, distance);

        //创建GEO参数
        RedisGeoCommands.GeoRadiusCommandArgs args = RedisGeoCommands
                .GeoRadiusCommandArgs
                .newGeoRadiusArgs()
                .includeDistance() //结果中包含距离
                .includeCoordinates() //结果中包含坐标
                .sortAscending(); //升序排列

        //执行GEO计算，获得查询结果
        GeoResults<RedisGeoCommands.GeoLocation<String>> radius = redisTemplate.opsForGeo()
                .radius("driver_location", circle, args);

        ArrayList list = new ArrayList(); //需要通知的司机列表

        if (radius != null) {
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

### 三、消息收发

有很多人拿RabbitMQ和Kafka做对比，其实仅仅考虑性能还是不够的，我们还要权衡业务场景。例如RabbitMQ具有对消息的过滤功能，而Kafka则不能对Topic中的消息做过滤。也就是说消费者要接受该Topic中所有的消息。RabbitMQ允许我们对消息设置TTL过期时间，如果超过TTL时间，那么RabbitMQ会自动销毁队列或者主题中的消息。这个功能特别实用，例如一个用户10年前注册了微博帐户，在这10年中，微博会给该用户发送非常多的公告通知，但是该用户迟迟不上线。想这样的幽灵用户，微博系统中还有上百万。你说微博系统有必要把这些幽灵用户的公告消息永久保存吗？没必要是吧，所以我们用RabbitMQ缓存公告消息，超过1年就自动销毁。这样微博系统就不用长期持久保存那些幽灵用户的公告通知了。

#### 1.RabbitMQ的五种模式

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
    private String from;
    private String to;
    private String expectsFee;
    private String mileage;
    private String minute;
    private String distance;
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

#### 3.Redis事务解决超售问题

##### 1.Redis事务

Redis的事务就是个批处理机制，底层的原理和乐观锁相似，只不过把乐观锁拿到内存中实现。

比如说现在客户端A，在修改数据之前，会先要观察要修改的数据，相当于记下了数据的版本号。我们可以开启事务机制，所有的命令都不会立即发送给Redis，而是先缓存到客户端本地。等我们提交事务的时候，一次性，把这个命令发送给Redis。Redis分析版本号之后，发现没有问题，这时候就会执行这些批处理的命令，而且在执行过程中不允许打断，不会处理其他客户端的命令，这样就不会出现超售现象。在客户端A在本地缓存命令的时候，这个时候客户端B修改了数据，版本号同时也更新了。这个时候客户端A提交事物，Redis发现客户端A的版本号，跟现有的数据版本号有逻辑冲突，所以就禁止执行客户端A的命令，这样事物就失败了。失败归失败，但是没有产生超售的现象。

##### 2.Redis的AOF模式

由于Redis使用内存缓存数据，如果Redis宕机，重启Redis之后，原本内存中缓存的数据就全都消失了。为了在宕机之后能有效恢复之前缓存的数据，我们可以开启Redis的持久化功能。

Redis有RDB和AOF两种持久化方式。RDB会根据指定的规则定时将内存中的数据保存到硬盘中，容易因为持久化不及时，导致恢复的时候丢失一部分缓存数据。AOF会将每次执行的命令及时保存到硬盘中，实时性更好，丢失的数据更少。

redis事务代码如下：这部分比较重要

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
## 第五章笔记

无论司机端还是乘客端的小程序，如果遇到微信闪退，重新登录小程序之后，必须加载当前的订单。司机端和乘客端的小程序都有司乘同显页面，这也是我们本章要完成的工作。另外，订单在执行的过程中，小程序要实时采集录音，把录音和对话文本上传到后端系统。还有就是乘客和司机如果配合刷单，骗取平台补贴，这个事情我们要用程序做个判断，把骗取补贴这个漏洞给堵上。

### 一、司机端加载执行的订单

司机加载订单时，需要根据司机编号从订单表中查询出正在执行的订单，并根据订单表中的用户id，查询出用户信息。这个信息主要是乘客用户的名字、电话。

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

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer() {
        RedisMessageListenerContainer redisMessageListenerContainer = new RedisMessageListenerContainer();
        redisMessageListenerContainer.setConnectionFactory(redisConnectionFactory);
        return redisMessageListenerContainer;
    }
}

```

同时定义一个

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





