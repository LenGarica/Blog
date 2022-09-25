# Redis实现文章自动保存

我们将文章设置为map结构。

## 1. 定义RedisUtil方法
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

## 2. 定义RedisService业务操作

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

## 3. RedisServiceImpl 

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
## 4. controller层

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

